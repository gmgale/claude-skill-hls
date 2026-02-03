<command-name>hls-vod</command-name>
<description>Create Video-on-Demand (VOD) HLS streams using mediafilesegmenter and variantplaylistcreator.</description>

# HLS VOD Workflow

Guide users through creating Video-on-Demand HLS streams from media files.

## Overview

The VOD workflow segments pre-recorded media files into HLS-compatible segments with multiple bitrate variants for adaptive streaming.

## Prerequisites

```bash
# Verify tools
which mediafilesegmenter variantplaylistcreator mediastreamvalidator

# Supported input formats
# MOV, MP4, M4V, M4A, MP3
```

## Workflow Steps

### Step 1: Prepare Output Directory

```bash
mkdir -p output/{hi,mid,lo,audio,subtitles}
```

### Step 2: Segment Media at Multiple Bitrates

The source video should ideally be pre-encoded at multiple bitrates. If working with a single high-quality source, segment it directly:

```bash
# High quality (original)
mediafilesegmenter \
  -f output/hi \
  -t 6 \
  --format iso \
  -z iframe_index.m3u8 \
  input.mp4
```

**Key Options:**
- `-f <path>` - Output directory (required)
- `-t <duration>` - Target segment duration (default 10s, recommend 6s)
- `--format iso` - Use fMP4 format (recommended for modern compatibility)
- `--format transport` - Use MPEG-2 Transport Stream
- `-z <name>` - Generate I-frame playlist for trick play
- `-b <url>` - Base URL prefix for playlist entries
- `-I` - Generate only I-frame playlist

### Step 3: Create Audio-Only Variant (Optional but Recommended)

For bandwidth-constrained scenarios:

```bash
mediafilesegmenter \
  -f output/audio \
  -t 6 \
  --format iso \
  -a \
  input.mp4
```

The `-a` flag extracts audio only.

### Step 4: Create Multivariant Playlist

After segmenting, each output directory contains a `.plist` file with stream metadata. Use these to create the multivariant playlist:

```bash
variantplaylistcreator \
  -o output/master.m3u8 \
  http://example.com/hi/prog_index.m3u8 output/hi/prog_index.m3u8.plist \
  --iframe-url http://example.com/hi/iframe_index.m3u8 \
  http://example.com/mid/prog_index.m3u8 output/mid/prog_index.m3u8.plist \
  --iframe-url http://example.com/mid/iframe_index.m3u8 \
  http://example.com/lo/prog_index.m3u8 output/lo/prog_index.m3u8.plist \
  --iframe-url http://example.com/lo/iframe_index.m3u8
```

**Syntax:** `variantplaylistcreator -o <output> <url1> <plist1> [--iframe-url <iframe1>] <url2> <plist2> ...`

### Step 5: Validate the Stream

```bash
mediastreamvalidator -O validation.json -V output/master.m3u8
hlsreport -V -o report.html validation.json
```

## Complete VOD Example

```bash
#!/bin/bash
# VOD HLS Encoding Workflow

INPUT="source_video.mp4"
OUTPUT_DIR="hls_output"
BASE_URL="http://example.com/video"

# Create directories
mkdir -p "$OUTPUT_DIR"/{hi,mid,lo}

# Segment high quality
mediafilesegmenter \
  -f "$OUTPUT_DIR/hi" \
  -b "$BASE_URL/hi" \
  -t 6 \
  --format iso \
  -z iframe_index.m3u8 \
  "$INPUT"

# Create multivariant playlist (single variant example)
variantplaylistcreator \
  -o "$OUTPUT_DIR/master.m3u8" \
  "$BASE_URL/hi/prog_index.m3u8" "$OUTPUT_DIR/hi/prog_index.m3u8.plist" \
  --iframe-url "$BASE_URL/hi/iframe_index.m3u8"

# Validate
mediastreamvalidator -O "$OUTPUT_DIR/validation.json" "$OUTPUT_DIR/master.m3u8"
hlsreport -V -o "$OUTPUT_DIR/report.html" "$OUTPUT_DIR/validation.json"

echo "Done! Open $OUTPUT_DIR/report.html to review validation results."
```

## mediafilesegmenter Options Reference

### Output Options
| Option | Description |
|--------|-------------|
| `-f <path>` | Output directory (required) |
| `-b <url>` | Base URL for playlist entries |
| `-B <url>` | Base URL for key files |
| `-o <name>` | Index filename (default: prog_index.m3u8) |

### Format Options
| Option | Description |
|--------|-------------|
| `--format iso` | Fragmented MP4 (fMP4) - recommended |
| `--format transport` | MPEG-2 Transport Stream |
| `--format packed` | Packed audio (AAC in ADTS) |
| `--format cmaf` | Common Media Application Format |

### Segment Options
| Option | Description |
|--------|-------------|
| `-t <seconds>` | Target segment duration (default: 10) |
| `-z <name>` | I-frame playlist filename |
| `-I` | Generate I-frame playlist only |

### Track Selection
| Option | Description |
|--------|-------------|
| `-a` | Audio only |
| `-A` | Video only (no audio) |
| `-T <id>` | Specific track ID |

### Encryption Options
| Option | Description |
|--------|-------------|
| `-k <path>` | Key file or directory |
| `-K <url>` | Key URL prefix |
| `--key-rotation-period <n>` | Rotate key every n segments |

### Other Options
| Option | Description |
|--------|-------------|
| `-V` | Verbose output |
| `-q` | Quiet mode |
| `-F <file>` | Key info file for sample-AES |

## Output Files

After running mediafilesegmenter, you'll find:

```
output/
├── prog_index.m3u8           # Media playlist
├── prog_index.m3u8.plist     # Metadata for variantplaylistcreator
├── iframe_index.m3u8         # I-frame playlist (if -z used)
├── fileSequence0.mp4         # Initialization segment (fMP4)
├── fileSequence1.m4s         # Media segments
├── fileSequence2.m4s
└── ...
```

## VOD Playlist Example

Generated `prog_index.m3u8`:
```m3u8
#EXTM3U
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-TARGETDURATION:6
#EXT-X-VERSION:7
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-MAP:URI="fileSequence0.mp4"
#EXTINF:6.006,
fileSequence1.m4s
#EXTINF:6.006,
fileSequence2.m4s
#EXTINF:5.839,
fileSequence3.m4s
#EXT-X-ENDLIST
```

**Key tags:**
- `EXT-X-PLAYLIST-TYPE:VOD` - Content won't change
- `EXT-X-TARGETDURATION` - Maximum segment duration
- `EXT-X-MAP` - Initialization segment (fMP4)
- `EXT-X-ENDLIST` - Playlist is complete

## Recommended Bitrate Ladder

### H.264/AVC (16:9)
| Resolution | Bitrate | Frame Rate |
|------------|---------|------------|
| 416 x 234 | 145 kbps | ≤30 fps |
| 640 x 360 | 365 kbps | ≤30 fps |
| 768 x 432 | 730 kbps | ≤30 fps |
| 960 x 540 | 2000 kbps | source |
| 1280 x 720 | 3000-4500 kbps | source |
| 1920 x 1080 | 6000-7800 kbps | source |

### HEVC/H.265 SDR (30 fps)
| Resolution | Bitrate |
|------------|---------|
| 640 x 360 | 145 kbps |
| 960 x 540 | 600-1600 kbps |
| 1280 x 720 | 2400-3400 kbps |
| 1920 x 1080 | 4500-5800 kbps |
| 3840 x 2160 | 11600-16800 kbps |

## Tips

1. **Use fMP4 format** (`--format iso`) for better compatibility with modern players
2. **Always generate I-frame playlists** (`-z`) to enable scrubbing/trick play
3. **Target 6-second segments** for good balance of latency and efficiency
4. **Include audio-only variant** for very low bandwidth scenarios
5. **Pre-encode source** at target bitrates before segmenting for best quality
6. **Validate immediately** after segmenting to catch issues early

## Troubleshooting

### "No video track found"
- Input file may be audio-only or corrupted
- Check with: `ffprobe input.mp4`

### "Unsupported codec"
- mediafilesegmenter supports H.264, HEVC, AAC
- Transcode unsupported codecs first

### Segments too long
- Ensure source has keyframes at regular intervals (every 2s recommended)
- Re-encode with: `ffmpeg -i input.mp4 -g 48 -keyint_min 48 output.mp4` (for 24fps)

### Missing .plist file
- Required for variantplaylistcreator
- Re-run mediafilesegmenter; it generates automatically
