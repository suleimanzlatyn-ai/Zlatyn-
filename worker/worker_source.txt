#!/usr/bin/env python3
"""Zlatyn local clipping worker.

The web UI is only the control surface. Video acquisition, transcription and
rendering happen on the user's machine so there is no per-clip cloud charge.
Only process media you own or are authorized to process.
"""
import json, os, re, shutil, subprocess, threading, uuid, math
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse

ROOT = os.path.abspath(os.path.dirname(__file__))
OUT = os.path.join(ROOT, "output")
TMP = os.path.join(ROOT, ".zlatyn_tmp")
os.makedirs(OUT, exist_ok=True)
os.makedirs(TMP, exist_ok=True)

JOBS = {}
LOCK = threading.Lock()

HOOK_PHRASES = {
    "but": 3, "because": 2, "nobody": 4, "most people": 4,
    "the truth": 5, "here's": 4, "i realized": 4, "i was wrong": 5,
    "the problem": 3, "the reason": 3, "you need": 3, "never": 3,
    "always": 2, "actually": 2, "secret": 4, "mistake": 4,
    "changed": 3, "crazy": 3, "imagine": 4, "why": 3, "how": 2,
    "what if": 5, "nobody tells": 5, "i didn't": 3, "i never": 3,
}


def run_cmd(*args, timeout=None):
    return subprocess.run(list(args), stdout=subprocess.PIPE, stderr=subprocess.PIPE,
                          text=True, timeout=timeout)


def installed(name):
    return shutil.which(name) is not None


def clamp(x, lo=0, hi=100):
    return max(lo, min(hi, x))


def score_segment(text, dur):
    """Heuristic first-pass ranking; designed to be conservative rather than flashy."""
    t = re.sub(r"\s+", " ", text.strip())
    low = t.lower()
    words = re.findall(r"\b\w+[’']?\w*\b", low)
    n = len(words)
    score = 46.0

    # Hook language / curiosity.
    for phrase, weight in HOOK_PHRASES.items():
        if phrase in low:
            score += weight
    if "?" in t:
        score += 6
    if "!" in t:
        score += 2
    if any(x in low for x in ("first time", "then", "until", "turns out", "ended up")):
        score += 4

    # Useful spoken density without rewarding rambling.
    if 35 <= n <= 145:
        score += 8
    elif n < 14:
        score -= 16
    elif n > 190:
        score -= 6

    # Prefer common short-form lengths but allow exceptional stories.
    if 18 <= dur <= 42:
        score += 8
    elif 42 < dur <= 65:
        score += 5
    elif dur < 10 or dur > 85:
        score -= 12

    # Penalize obvious filler openings.
    if re.match(r"^(so|okay|um|uh|like|yeah|well)\b", low):
        score -= 5
    if low.count("you know") >= 2:
        score -= 3

    # A clean ending is usually better than a fragment.
    if re.search(r"[.!?]$", t):
        score += 3
    if t.endswith(("and", "but", "because", "so", "or")):
        score -= 7

    return round(clamp(score), 2)


def transcribe(audio):
    try:
        from faster_whisper import WhisperModel
    except Exception as exc:
        raise RuntimeError("Faster-Whisper is not installed. Run: python3 -m pip install -r requirements.txt") from exc

    model_size = os.environ.get("ZLATYN_WHISPER_MODEL", "small")
    model = WhisperModel(model_size, device="auto", compute_type="auto")
    segments, _ = model.transcribe(audio, word_timestamps=True, vad_filter=True,
                                    beam_size=3, condition_on_previous_text=False)
    out = []
    for s in segments:
        text = (s.text or "").strip()
        if text:
            out.append({"start": float(s.start), "end": float(s.end), "text": text})
    return out


def make_candidates(segs):
    candidates = []
    # Multiple window sizes create both punchy clips and deeper story clips.
    sizes = (6, 8, 10, 12)
    for i in range(len(segs)):
        for size in sizes:
            j = min(len(segs), i + size)
            if j <= i:
                continue
            start = max(0.0, segs[i]["start"] - 0.45)
            end = segs[j - 1]["end"] + 0.15
            dur = end - start
            if dur < 14 or dur > 75:
                continue
            text = " ".join(x["text"] for x in segs[i:j]).strip()
            score = score_segment(text, dur)
            candidates.append({"start": start, "end": end, "duration": dur,
                               "text": text, "score": score})
    return candidates


def similarity(a, b):
    sa = set(re.findall(r"\b[a-z0-9]{4,}\b", a.lower()))
    sb = set(re.findall(r"\b[a-z0-9]{4,}\b", b.lower()))
    if not sa or not sb:
        return 0.0
    return len(sa & sb) / max(1, len(sa | sb))


def pick_candidates(candidates, limit):
    candidates.sort(key=lambda x: x["score"], reverse=True)
    picked = []
    for c in candidates:
        # Don't return the same sentence/topic repeatedly.
        if any(abs(c["start"] - p["start"]) < 14 for p in picked):
            continue
        if any(similarity(c["text"], p["text"]) >= 0.58 for p in picked):
            continue
        picked.append(c)
        if len(picked) >= limit:
            break
    return sorted(picked, key=lambda x: x["start"])


def render(source, c, out):
    # Clean 9:16 framing, no captions/watermarks/effects. Mild audio leveling only.
    vf = "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920"
    p = run_cmd("ffmpeg", "-y", "-ss", f"{c['start']:.3f}", "-i", source,
                "-t", f"{c['duration']:.3f}", "-vf", vf,
                "-c:v", "libx264", "-preset", "veryfast", "-crf", "20",
                "-c:a", "aac", "-b:a", "160k", "-af", "loudnorm=I=-14:TP=-1.5:LRA=11",
                "-movflags", "+faststart", out, timeout=900)
    if p.returncode:
        raise RuntimeError(p.stderr[-1800:])


def set_job(job, **values):
    with LOCK:
        JOBS.setdefault(job, {}).update(values)


def process_job(job, url, max_clips):
    work = os.path.join(TMP, job)
    os.makedirs(work, exist_ok=True)
    try:
        if not installed("ffmpeg"):
            raise RuntimeError("FFmpeg is not installed")
        if not installed("yt-dlp"):
            raise RuntimeError("yt-dlp is not installed. Run: python3 -m pip install -r requirements.txt")

        set_job(job, state="downloading", progress=8, message="Downloading the authorized source locally")
        source_pattern = os.path.join(work, "source.%(ext)s")
        p = run_cmd("yt-dlp", "--no-playlist", "-f", "bv*+ba/b",
                    "--merge-output-format", "mp4", "-o", source_pattern, url, timeout=1800)
        if p.returncode:
            raise RuntimeError(p.stderr[-1800:])
        files = [os.path.join(work, x) for x in os.listdir(work)
                 if x.lower().endswith((".mp4", ".mkv", ".webm", ".mov"))]
        if not files:
            raise RuntimeError("No source video was produced")
        source = max(files, key=os.path.getsize)

        set_job(job, state="audio", progress=18, message="Preparing audio")
        audio = os.path.join(work, "audio.wav")
        p = run_cmd("ffmpeg", "-y", "-i", source, "-vn", "-ac", "1", "-ar", "16000", audio, timeout=600)
        if p.returncode:
            raise RuntimeError(p.stderr[-1800:])

        set_job(job, state="transcribing", progress=25, message="Transcribing locally")
        segs = transcribe(audio)
        if not segs:
            raise RuntimeError("No speech was detected")

        set_job(job, state="ranking", progress=52, message="Ranking hooks and removing repeats")
        candidates = make_candidates(segs)
        picked = pick_candidates(candidates, max_clips)
        if not picked:
            raise RuntimeError("No strong clip candidates were found")

        results = []
        total = len(picked)
        for idx, c in enumerate(picked, 1):
            out = os.path.join(OUT, f"{job}-{idx:02d}.mp4")
            try:
                render(source, c, out)
            except Exception:
                continue
            results.append({
                "title": c["text"][:180],
                "score": c["score"],
                "duration": round(c["duration"], 2),
                "reason": "hook + pacing + uniqueness",
                "download": "/output/" + os.path.basename(out),
            })
            set_job(job, progress=55 + int(42 * idx / max(1, total)),
                    message=f"Rendering clip {idx} of {total}")

        if not results:
            raise RuntimeError("FFmpeg could not render any selected clips")
        set_job(job, state="done", progress=100, message="Finished", clips=results)
    except Exception as exc:
        set_job(job, state="error", progress=100, message=str(exc), error=str(exc))
    finally:
        shutil.rmtree(work, ignore_errors=True)


class Handler(BaseHTTPRequestHandler):
    def send_headers(self, typ="application/json"):
        self.send_header("Content-Type", typ)
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Headers", "Content-Type")
        self.send_header("Access-Control-Allow-Methods", "GET,POST,OPTIONS")
        self.end_headers()

    def do_OPTIONS(self):
        self.send_response(204)
        self.send_headers()

    def do_GET(self):
        parsed = urlparse(self.path)
        if parsed.path == "/health":
            self.send_response(200); self.send_headers()
            self.wfile.write(json.dumps({"ok": True, "ffmpeg": installed("ffmpeg"),
                                         "yt_dlp": installed("yt-dlp"),
                                         "worker": "zlatyn-async-v2"}).encode())
            return
        if parsed.path.startswith("/jobs/"):
            job = parsed.path.rsplit("/", 1)[-1]
            with LOCK: data = dict(JOBS.get(job, {}))
            if not data:
                self.send_error(404); return
            self.send_response(200); self.send_headers(); self.wfile.write(json.dumps(data).encode()); return
        if parsed.path.startswith("/output/"):
            fn = os.path.basename(parsed.path)
            path = os.path.join(OUT, fn)
            if not os.path.isfile(path): self.send_error(404); return
            self.send_response(200)
            self.send_header("Content-Type", "video/mp4")
            self.send_header("Access-Control-Allow-Origin", "*")
            self.end_headers()
            with open(path, "rb") as f:
                while True:
                    chunk = f.read(1024 * 1024)
                    if not chunk: break
                    self.wfile.write(chunk)
            return
        self.send_error(404)

    def do_POST(self):
        if self.path != "/process": self.send_error(404); return
        try:
            n = int(self.headers.get("Content-Length", "0"))
            data = json.loads(self.rfile.read(n))
            url = data.get("url", "").strip()
            if not re.match(r"^https?://(www\.)?(youtube\.com|youtu\.be)/", url):
                raise ValueError("Only YouTube URLs are accepted. Process only videos you own or are authorized to use.")
            max_clips = min(30, max(1, int(data.get("max_clips", 30))))
            job = uuid.uuid4().hex[:10]
            set_job(job, state="queued", progress=1, message="Queued", clips=[])
            threading.Thread(target=process_job, args=(job, url, max_clips), daemon=True).start()
            self.send_response(202); self.send_headers(); self.wfile.write(json.dumps({"job": job}).encode())
        except Exception as exc:
            self.send_response(400); self.send_headers(); self.wfile.write(json.dumps({"error": str(exc)}).encode())


if __name__ == "__main__":
    print("Zlatyn worker: http://127.0.0.1:8765")
    print("Output:", OUT)
    print("ffmpeg:", installed("ffmpeg"), "yt-dlp:", installed("yt-dlp"))
    ThreadingHTTPServer(("127.0.0.1", 8765), Handler).serve_forever()
