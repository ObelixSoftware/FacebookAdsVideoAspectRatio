# Facebook ad video (`output.mp4`)

## What Meta allows (and how “resize” works)

Use the official Meta documentation before each campaign; limits change by placement.

In general:

- **Containers:** MP4 and MOV are widely accepted for video ads.
- **Video:** **H.264** with **4:2:0** chroma (e.g. **yuv420p**) is the usual safe choice.
- **Audio:** **AAC** is typical (stereo is common; mono is allowed if your source is mono).
- **Aspect ratios:** Feed commonly supports **1:1** and **4:5**; **9:16** is common for Reels/Stories style placements. This project standardises on **4:5** at **1080×1350** for `output.mp4`.
- **How the app “resizes” your file:** The file keeps **fixed pixel dimensions**. Facebook’s apps **scale the whole frame** to fit the device and placement (letterboxing or fitting inside the slot). The logo stays a **constant fraction of the creative** because it is burned into those pixels—same behaviour as QuickTime scaling the entire picture. That is **valid** and expected; Meta does not keep a separate “floating” logo layer in the MP4.

## Canonical FFmpeg command (produces `output.mp4`)

Source: `input.mp4` + `logo.png`. Output: **1080×1350**, **H.264**, **yuv420p**, **AAC**, **fast start** for web. Logo size is set to **22% of output width**.

**FFmpeg 8:** The two-input `scale` for the logo uses the composed frame as reference; **`split`** is required so that frame can also feed `overlay`.

**zsh:** Quote `-map '[outv]'` and `-map '0:a?'` so brackets are not globbed.

```bash
ffmpeg -y -i input.mp4 -i logo.png -filter_complex "[0:v]scale=1080:1350:force_original_aspect_ratio=decrease,pad=1080:1350:(ow-iw)/2:(oh-ih)/2,setsar=1[bg];[bg]split=2[bg1][bg2];[1:v][bg1]scale=w=rw*0.22:h=-1:flags=lanczos[logo];[bg2][logo]overlay=W-w-20:20[outv]" -map '[outv]' -map '0:a?' -c:v libx264 -pix_fmt yuv420p -crf 20 -c:a aac -b:a 128k -movflags +faststart output.mp4
```

### Windows (PowerShell / CMD) command

Use this equivalent command on Windows (same filter, Windows-safe quoting):

```bash
ffmpeg -y -i input.mp4 -i logo.png -filter_complex "[0:v]scale=1080:1350:force_original_aspect_ratio=decrease,pad=1080:1350:(ow-iw)/2:(oh-ih)/2,setsar=1[bg];[bg]split=2[bg1][bg2];[1:v][bg1]scale=w=rw*0.22:h=-1:flags=lanczos[logo];[bg2][logo]overlay=W-w-20:20[outv]" -map "[outv]" -map "0:a?" -c:v libx264 -pix_fmt yuv420p -crf 20 -c:a aac -b:a 128k -movflags +faststart output.mp4
```

### Logo sizing options

Current default in this project:

- `scale=w=rw*0.22:h=-1` -> logo width is 22% of the output width, height follows original aspect ratio (no distortion).
- With current output size `1080x1350` and source logo `1376x768`, the rendered logo is approximately **237x132 px**.
- Overlay position is top-right with 20 px margin: `overlay=W-w-20:20`.

If you want explicit width + height (in pixels), replace only the logo scale part with:

```text
scale=w=260:h=140
```

Notes:

- This forces both dimensions and can stretch/squash the logo.
- To keep proportions, keep `h=-1` and only adjust `w` (for example `rw*0.25`).

## Verification (machine-checked)

After encoding, run:

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,codec_name,pix_fmt,display_aspect_ratio -of default=noprint_wrappers=1:nokey=0 output.mp4
```

**Recorded result** for the `output.mp4` in this folder (FFmpeg 8.0.1, encode completed successfully):

```text
codec_name=h264
width=1080
height=1350
display_aspect_ratio=4:5
pix_fmt=yuv420p
```

Audio stream: **AAC**, **48000 Hz**, **mono**, ~128 kb/s.

## Visual proof

A single decoded frame from `output.mp4` is saved as **`proof_frame.jpg`** (grabbed at ~3s). Open it to confirm the **1080×1350** canvas, letterboxing if the source is wider than 4:5, and the **logo** in the top-right.

## Client preview page

Open **`index.html`** to show the client:

- A 4:5 preview card (our actual exported file)
- A 1:1 preview card (allowed feed ratio)
- A 9:16 preview card (vertical/Reels-style ratio)
- Local validation evidence (`ffprobe` findings + `proof_frame.jpg`)
- Official references

## References

- [Meta Business Help Center - Ad specs and technical requirements overview](https://www.facebook.com/business/help/742478679120153)
- [Meta Business Help Center - Best practices for aspect ratios across placements](https://www.facebook.com/business/help/103816146375741)
- [Facebook Ads Guide - Facebook Feed video ad specs](https://www.facebook.com/business/ads-guide/update/video/facebook-feed/outcome-engagement)
- [Facebook Ads Guide - Facebook Reels video ad specs](https://www.facebook.com/business/ads-guide/update/video/facebook-facebook-reels)
