# Youtube Guitar Tab Parser

CLI that turns a YouTube guitar-lesson video into a PDF of the guitar tab.

It downloads the video, samples frames, uses Claude vision to locate the tab
region, crops every frame to that region, de-duplicates the crops by the bar
number printed on each line of the score, and stitches the distinct tab lines
vertically into a PDF. It works out of the box with no configuration — the PDF
is written to `out/<video-title>.pdf`, with the video title as a heading on the
first page and in the document metadata.

## Example

Generated from the YouTube lesson
[Game of Thrones - Guitar Lesson + TAB](https://youtu.be/WgU5tDGC-Vc):

- 🎬 Source video: <https://youtu.be/WgU5tDGC-Vc>
- 📄 Output PDF: [examples/Game of Thrones.pdf](examples/Game%20of%20Thrones.pdf)

## Requirements

- Node.js ≥ 20
- [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) and [`ffmpeg`](https://ffmpeg.org/) on your `PATH`
  (`brew install yt-dlp ffmpeg`)
- An Anthropic API key

## Setup

```bash
npm install
npm run build
cp .env.example .env   # then put your ANTHROPIC_API_KEY in it
```

## Usage

```bash
# with a .env file
node --env-file=.env dist/cli.js "https://www.youtube.com/watch?v=WgU5tDGC-Vc"

# or with the key already exported
export ANTHROPIC_API_KEY=sk-ant-...
node dist/cli.js "https://www.youtube.com/watch?v=WgU5tDGC-Vc"
```

During development you can skip the build step:

```bash
node --env-file=.env --import tsx src/cli.ts "<url>"
```

The result is written to `out/<video-title>.pdf` (its path is also printed to
stdout); progress goes to stderr.

## Options

Everything has a sensible default; you normally only need the URL.

```
-i, --interval <seconds>   screenshot interval (default 2)
--model <id>               Claude vision model (default claude-sonnet-5)
--sample <n>               frames sampled for tab-region detection (default 6)
--dedup-threshold <n>      pre-dedup Hamming distance, cost control (default 12)
--max-height <px>          cap download resolution (default 720)
--keep-temp                keep intermediate frames/crops
```

## How it works

1. **Download** — `yt-dlp` fetches the video (≤ `--max-height`).
2. **Frames** — `ffmpeg` extracts one frame every `--interval` seconds.
3. **Detect** — a labeled horizontal-band grid is drawn on `--sample` frames and
   Claude vision reports which bands contain sheet music. The median first/last
   music band (robust to an outlier) gives the vertical extent; width is kept full.
   (Coarse labeled bands are far more reliable than asking a vision model for
   precise pixel coordinates.)
4. **Crop** — `sharp` crops every frame to that region.
5. **Pre-dedup** — a dHash perceptual hash drops near-identical consecutive crops.
   This is only a cost control to reduce the number of vision calls in the next step.
6. **Bar-number dedup** — Claude reads the measure/bar number printed at the start
   of each line and whether the crop is real sheet music. The tool keeps exactly
   one crop per distinct bar number (first appearance wins) and drops non-tab
   crops (title cards, intros/outros). Because the bar number is constant while
   the playback cursor sweeps a line and only changes when the score advances,
   this collapses all the near-identical cursor frames of a line into a single page.
7. **PDF** — `pdf-lib` stacks the distinct tab lines vertically down A4 pages,
   in the order they appear in the video. The video title (read from `yt-dlp`)
   becomes the file name, a heading on the first page, and the document metadata
   title. Output: `out/<video-title>.pdf`.
