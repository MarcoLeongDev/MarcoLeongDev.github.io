---
layout: post
title:  "The MP4 That Can't Stream: Why Your Browser Stutters (and How to Fix It)"
author: marco
categories: [ mp4, streaming, ffmpeg, video, browser, h264, isobmff, web, optimization, troubleshooting ]
image: assets/images/mp4-streaming-box-structure.png
featured: true
seo:
  description: "Understand why concatenated MP4s and fragmented MP4s break browser streaming — and how a single ffmpeg command with -movflags faststart fixes both."
  keywords: [mp4 streaming fix, fragmented mp4, concatenated mp4, moof boxes, mdat boxes, ffmpeg faststart, browser video stutter, isobmff, html5 video not working, mp4 remux, youtube-dl mp4 fix, moov atom location]
---

## The Problem: It Works Locally, But Not in the Browser

You download a video, open it in VLC — perfect. You drop it into a web page or a self-hosted streaming app, click play, and... nothing. Blank screen. Infinite spinner. Or a 2-second delay before playback starts, every single time.

The video file is valid. The browser supports H.264. The server is sending the right bytes. What gives?

The answer is hiding in the MP4 container's internal structure. Most developers know MP4 is a "container format" — but few realize there are *structural variants* that look identical from the outside yet behave completely differently in a browser streaming context. Two particularly nasty variants are **concatenated MP4s** and **fragmented MP4s**.

---

## A Quick Primer: How an MP4 Is Organized

An MP4 file is built from ISO Base Media File Format (ISOBMFF) boxes (also called atoms). Every MP4 has at least these boxes in order:

```
[ftyp] → [moov] → [mdat]
```

- **`ftyp`**: File type box — declares the brand (e.g., `mp42`, `isom`).
- **`moov`**: Movie box — the index. Contains sample tables, track metadata, and byte offsets telling the decoder where each frame lives inside `mdat`.
- **`mdat`**: Media data box — the actual audio and video frames.

The browser needs the `moov` box to know the video's duration, dimensions, codec, and — crucially — where each frame is. That's what enables seeking, time display, and the playhead.

When streaming over HTTP (as every browser <video> element does), the browser sends Range requests to fetch chunks of the file. The `moov` box must be readable early, or the player has no map of the territory.

---

## Variant 1: Concatenated MP4s — The Frankenstein File

### What It Looks Like

```
[ftyp] → [moov] → [mdat A] → [mdat B] → [mdat C]
```

A concatenated MP4 has **multiple consecutive `mdat` boxes** after a single `moov`. This happens when files are naively joined with `cat` or when download tools (notably YouTube-DL / yt-dlp when interrupted and resumed) write multiple media segments into one container.

### The Root Cause

The `moov` box's sample tables only reference the **first** `mdat` box. The second and third `mdat` boxes are orphaned — the decoder doesn't know about them. The file is not technically malformed (it follows ISOBMFF grammar), but it's semantically broken for any player that actually processes the sample table.

### The Symptom

- **Blank player**: Browser shows a black screen or loading spinner forever.
- **Only first N seconds play**: Some players detect the first `mdat`, play that segment, then freeze.
- **Works in VLC, fails in browser**: VLC is forgiving — it often brute-force scans the entire file to find media. Browsers follow the spec strictly and rely on the `moov` sample table.

### Where to Look

Hex dump or binary scan. Look for the `mdat` ASCII marker after the `moov` box. If you see:

```
... mdat ... mdat ... mdat
```

consecutively (no `moof` or `moov` between them), you have a concatenated MP4. You can detect this programmatically by walking the box hierarchy:

```python
def is_concat_mp4(filepath: str) -> bool:
    with open(filepath, "rb") as f:
        # Validate ftyp header
        header = f.read(8)
        if len(header) < 8 or header[4:8] != b"ftyp":
            return False
        ftyp_size = int.from_bytes(header[0:4], "big")
        f.seek(ftyp_size - 8, 1)

        # Walk boxes looking for moov, then count mdat boxes
        while True:
            box_header = f.read(8)
            if len(box_header) < 8:
                return False
            box_size = int.from_bytes(box_header[0:4], "big")
            box_type = box_header[4:8]

            if box_type == b"moov":
                f.seek(box_size - 8, 1)
                mdat_count = 0
                while True:
                    next_header = f.read(8)
                    if len(next_header) < 8:
                        break
                    next_size = int.from_bytes(next_header[0:4], "big")
                    next_type = next_header[4:8]
                    if next_type == b"mdat":
                        mdat_count += 1
                        f.seek(next_size - 8, 1)
                    elif next_type in (b"moof", b"moov"):
                        return False  # fragmented, not concat
                    else:
                        break
                return mdat_count > 1
            else:
                f.seek(box_size - 8, 1)
```

---

## Variant 2: Fragmented MP4s (FMP4) — The Segment Monster

### What It Looks Like

```
[ftyp] → [moov] → [moof] → [mdat] → [moof] → [mdat] → [moof] → [mdat]
```

A fragmented MP4 (FMP4) breaks the media into **Movie Fragments**. Instead of one giant `moov` + one giant `mdat`, it stores a lightweight initial `moov` (often called the "movie header") plus alternating `moof` (movie fragment) and `mdat` pairs. Each `moof`+`mdat` segment is self-contained: the `moof` describes how to decode the following `mdat`.

This is the standard format for **MPEG-DASH** and **HLS** streaming, where segments need to be independently addressable for adaptive bitrate switching. But when you take a fragmented MP4 and serve it as a plain file over HTTP Range requests, things break.

### The Root Cause

With fragmented MP4s, the browser doesn't know the full duration or frame layout until it has parsed **every `moof` box**. The initial `moov` only contains the "skeleton" — track IDs, timescale — but not the sample table. The browser must:

1. Download and parse the initial `moov`.
2. Start downloading segments.
3. Parse each `moof` to discover frame offsets.
4. Only then can it report duration, enable seeking, and render frames.

For a 60-minute video, this means the browser might need to download and parse dozens or hundreds of `moof` boxes before it can build a complete picture. Every seek or load triggers this process anew.

### The Symptom

- **1–2 second delay on every page load**: The browser is parsing moof boxes before playback can begin.
- **Seeking is sluggish or broken**: After a seek, the browser must find the nearest moof+mdat pair's keyframe.
- **Duration shows "NaN" or "0:00" initially**: The player can't know the length until it has processed all moof boxes.
- **Works fine in VLC**: VLC pre-scans the entire file and builds an internal index. The browser spec doesn't require this.

### Where to Look

Open the file in a hex viewer or scan for the ASCII string `moof`:

```python
def is_fragmented_mp4(filepath: str) -> bool:
    with open(filepath, "rb") as f:
        data = f.read(10 * 1024 * 1024)  # First 10MB
        return b"moof" in data
```

In practice, detecting fragmented MP4s is simpler than detecting concatenated ones — just check for the presence of `moof` anywhere in the file. These boxes are at minimum 8 bytes, so a substring match on raw binary data is reliable.

---

## The Single Fix: `ffmpeg -movflags faststart`

Both problems share a common solution: **remux with faststart**. Faststart (also called "Web Optimized" or "MOOV at front") does two things in a single pass:

1. **Merges all media segments** into a single `mdat` box, eliminating concatenation artifacts.
2. **Rewrites the sample table** so the `moov` box is complete and placed at the beginning of the file (before the `mdat`), eliminating the fragmented structure.

The command:

```bash
ffmpeg -i input.mp4 -c copy -movflags faststart -f mp4 -y output.mp4
```

### Why This Works

Flag | Effect
-----|-------
`-c copy` | Stream copies — no re-encoding, no quality loss, near-instant.
`-movflags faststart` | Repositions the `moov` box to the front of the file and rebuilds the sample table from all media data.
`-f mp4` | Forces MP4 output format (ensures correct structure).
`-y` | Overwrite output without prompting.

After remux:
```
[moov] → [mdat (single, complete)]
```

The browser receives `moov` first, immediately knows the full structure, and can start playback instantly. Seeking works. Duration shows correctly. No more blank screen.

### Why `-c copy` Matters

Re-encoding a 4GB video could take hours. Stream copying (`-c copy`) tells FFmpeg to copy the compressed bitstream verbatim — no decode, no re-encode. The operation is pure container surgery: it reads the existing frames and writes them into a new container with correct box structure. A 10GB video remuxes in seconds.

### Verification

Always verify the output:

```bash
ffprobe -v quiet -print_format json -show_format -show_streams output.mp4
```

Check that:
- `format.nb_streams` equals the expected count (usually 2: video + audio).
- `streams[].codec_type` and `codec_name` match the original.
- The file opens instantaneously in a browser `<video>` element.

---

## Suggested Image Description

> A simplified ISOBMFF box diagram showing three MP4 structures side by side. Left: a normal MP4 with `[ftyp] → [moov] → [mdat]` — clean and compact. Center: a concatenated MP4 with `[ftyp] → [moov] → [mdat A] → [mdat B] → [mdat C]` — three mdat boxes in a row, the second and third orphaned. Right: a fragmented MP4 with `[ftyp] → [moov] → [moof] → [mdat] → [moof] → [mdat]` — alternating moof/mdat pairs. Each moof box is highlighted in red to emphasize the overhead. Below all three, an arrow pointing to the fixed version: `[moov at front] → [mdat, single]`. Use a dark background with blue boxes for normal structure, red highlights for problematic elements, and green for the fixed output.

---

## Summary

| Variant | Symptom | Cause | Detection |
|---------|---------|-------|-----------|
| **Concatenated MP4** | Blank player, only first N seconds play | Multiple orphaned `mdat` boxes; moov only references the first | Binary scan for consecutive `mdat` after `moov` |
| **Fragmented MP4** | 1–2s startup delay, seeking broken | Alternating `moof`+`mdat` segments; browser must parse all moof boxes | Binary scan for `moof` in first 10MB |

Both are trivially fixed with a single FFmpeg command. No re-encoding needed. No quality loss. Just a few seconds of container surgery, and your browser gets the well-behaved MP4 it deserves.

Happy streaming 🎥
