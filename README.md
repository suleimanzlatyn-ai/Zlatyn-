# Zlatyn — Private Clipping Engine

Zlatyn is a local-first personal clipping workspace. The published web page is only the interface; the heavy video work runs on your own computer. That keeps the workflow free of per-clip AI/API charges and avoids Vercel video-storage limits.

## Pipeline
- Accept a YouTube URL for a video you own or are authorized to process.
- Download the source locally with `yt-dlp`.
- Transcribe locally with Faster-Whisper.
- Generate multiple candidate windows.
- Rank hook strength, curiosity, spoken density, pacing and clean endings.
- Remove nearby and semantically similar candidates.
- Render up to 30 selected clips as clean 1080x1920 MP4.
- No captions, watermark, meme effects or forced templates.
- Jobs run in the background, so long videos do not make the browser request time out.

## Install
1. Install FFmpeg and make sure `ffmpeg` works in your terminal.
2. Run `./start-zlatyn.sh`. This installs the Python packages and starts the worker on `http://127.0.0.1:8765`.
3. Open the Zlatyn web page. It can be hosted on Vercel or opened locally through a static web server.

## Local UI test
In another terminal, from this folder run `python3 -m http.server 3000`, then open `http://127.0.0.1:3000`.

## Performance
The first Faster-Whisper run downloads the selected model. `ZLATYN_WHISPER_MODEL=small` is the default; a stronger model can improve transcription at the cost of speed/RAM. Rendering is CPU/GPU dependent. There is no honest way to promise unlimited zero-cost cloud processing: the free architecture relies on your own machine's CPU/GPU, RAM, disk and electricity.

## Important
Use only source videos you own or have permission to process, and comply with the source platform's terms and the rules of any content-reward campaign.
