---
name: hls
description: Help users work with Apple's HTTP Live Streaming (HLS) Tools for creating, validating, and deploying HLS and Low-Latency HLS streams on macOS. Covers mediafilesegmenter, mediastreamsegmenter, mediastreamvalidator, hlsreport, and related tools.
---

# HLS Tools Skill

You are an expert assistant for Apple's HTTP Live Streaming (HLS) Tools. Help users create, validate, and deploy HLS streams following Apple's authoring specifications.

## Prerequisites

Before using HLS tools, verify installation:
```bash
which mediafilesegmenter mediastreamsegmenter mediastreamvalidator hlsreport variantplaylistcreator
```

Tools should be installed from "HTTP Live Streaming Tools for macOS Sequoia and later" available at:
https://developer.apple.com/download/all/?q=HTTP%20Live%20Streaming%20Tools

**Installation locations:**
- Binaries: `/usr/local/bin/`
- Documentation: `/usr/local/share/hlstools/`

## Core Tools

| Tool | Purpose |
|------|---------|
| `mediafilesegmenter` | Segment files for VOD |
| `mediastreamsegmenter` | Segment live streams |
| `mediastreamvalidator` | Validate HLS streams |
| `hlsreport` | Generate HTML validation reports |
| `variantplaylistcreator` | Create multivariant playlists |
| `mediasubtitlesegmenter` | Segment subtitles |
| `tsrecompressor` | Generate/capture test streams |
| `id3taggenerator` | Create ID3 metadata tags |

## Standard Workflow

### Phase 1: Preparation
```bash
# Verify tools are installed
which mediafilesegmenter mediastreamvalidator

# Backup source media
cp -r source_dir source_dir_backup_$(date +%Y%m%d_%H%M%S)

# Create output structure
mkdir -p output/{segments,keys,reports}
```

### Phase 2: Processing
- **VOD:** Use `mediafilesegmenter` → `variantplaylistcreator`
- **Live:** Use `tsrecompressor` → `mediastreamsegmenter`

### Phase 3: Validation
```bash
mediastreamvalidator -O validation_data.json -V <playlist_url>
hlsreport -V -o report.html validation_data.json
```

### Phase 4: Deployment
Configure web server MIME types:
- `.m3u8` → `application/vnd.apple.mpegurl`
- `.ts` → `video/mp2t`
- `.mp4/.m4s` → `video/mp4`

---

## VOD Workflow (mediafilesegmenter)

### Basic VOD Segmentation
```bash
mediafilesegmenter \
  -f output/video \
  -t 6 \
  --format iso \
  -z iframe_index.m3u8 \
  input.mp4
```

### Key Options
| Option | Description |
|--------|-------------|
| `-f <path>` | Output directory (required) |
| `-t <seconds>` | Target segment duration (default 10, recommend 6) |
| `--format iso` | Use fMP4 format (recommended) |
| `--format transport` | Use MPEG-2 TS |
| `-z <name>` | Generate I-frame playlist |
| `-b <url>` | Base URL for playlist |
| `-a` | Audio only |
| `-A` | Video only |
| `-k <path>` | Encryption key file |

### Multi-bitrate VOD
```bash
# Segment each quality level
mediafilesegmenter -f output/hi -t 6 --format iso -z iframe.m3u8 input_high.mp4
mediafilesegmenter -f output/mid -t 6 --format iso -z iframe.m3u8 input_mid.mp4
mediafilesegmenter -f output/lo -t 6 --format iso -z iframe.m3u8 input_low.mp4

# Create multivariant playlist
variantplaylistcreator -o output/master.m3u8 \
  http://example.com/hi/prog_index.m3u8 output/hi/prog_index.m3u8.plist \
  --iframe-url http://example.com/hi/iframe.m3u8 \
  http://example.com/mid/prog_index.m3u8 output/mid/prog_index.m3u8.plist \
  http://example.com/lo/prog_index.m3u8 output/lo/prog_index.m3u8.plist
```

---

## Live Streaming Workflow

### Generate Test Stream
```bash
tsrecompressor -g -h -a \
  -H 224.0.0.50:9121 \
  -O 224.0.0.50:9123 \
  -L 224.0.0.50:9125
```

### Segment Live Stream
```bash
mediastreamsegmenter \
  -f /var/www/html/live \
  -t 6 \
  -s 10 \
  -D \
  -T \
  --format iso \
  224.0.0.50:9123
```

### Key Options
| Option | Description |
|--------|-------------|
| `-f <path>` | Output directory |
| `-t <seconds>` | Target segment duration |
| `-s <count>` | Sliding window size |
| `-D` | Delete expired segments |
| `-T` | Include program date-time |
| `-w <ms>` | Partial segment duration (LL-HLS) |

---

## Validation

### Validate Stream
```bash
mediastreamvalidator \
  -t 120 \
  -O validation.json \
  -V \
  http://example.com/master.m3u8
```

### Generate Report
```bash
hlsreport -V -R iOS -o report.html validation.json
```

### Rule Sets
| Rule Set | Description |
|----------|-------------|
| `standard` | General requirements |
| `iOS` | iOS-specific |
| `tvOS` | tvOS-specific |
| `macOS` | macOS-specific |
| `airplay2tv` | AirPlay 2 TVs |

### Issue Categories (Priority Order)
1. **Critical Must Fix** - Will cause playback failures
2. **Must Fix** - Serious issues
3. **Should Fix** - Recommended improvements
4. **Other Issues** - Informational

---

## Low-Latency HLS

Enable partial segments for 2-5 second latency:

```bash
mediastreamsegmenter \
  -f /var/www/html/live \
  -t 4 \
  -s 16 \
  -D \
  -T \
  -w 1002 \
  --format iso \
  224.0.0.50:9123
```

**Requirements:**
- Origin server supporting blocking playlist requests
- HTTP/2 recommended
- Short segment duration (4s)
- Partial segments (~1s)

---

## Subtitles

### Segment WebVTT
```bash
mediasubtitlesegmenter \
  -f output/subtitles/en \
  -t 6 \
  -D 3600 \
  subtitles.vtt
```

### Add to Multivariant Playlist
```m3u8
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",DEFAULT=YES,LANGUAGE="en",URI="subs/en/prog_index.m3u8"

#EXT-X-STREAM-INF:BANDWIDTH=4000000,...,SUBTITLES="subs"
video/prog_index.m3u8
```

---

## Encryption

### AES-128 Encryption
```bash
# Generate key
openssl rand 16 > encryption.key

# Segment with encryption
mediafilesegmenter \
  -f output/encrypted \
  -t 6 \
  --format iso \
  -k encryption.key \
  -K https://keys.example.com/ \
  input.mp4
```

### Key Rotation
```bash
mediafilesegmenter \
  -f output/encrypted \
  -k keys/ \
  -K https://keys.example.com/ \
  --key-rotation-period 15 \
  input.mp4
```

---

## ID3 Metadata

### Send to Live Stream
```bash
id3taggenerator \
  -a 127.0.0.1:5555 \
  --title "Song Title" \
  --artist "Artist Name"
```

Configure segmenter with `--meta-port 5555` to receive tags.

---

## Bitrate Ladder (H.264)

| Resolution | Bitrate | Frame Rate |
|------------|---------|------------|
| 416 x 234 | 145 kbps | ≤30 fps |
| 640 x 360 | 365 kbps | ≤30 fps |
| 768 x 432 | 730 kbps | ≤30 fps |
| 960 x 540 | 2000 kbps | source |
| 1280 x 720 | 3000-4500 kbps | source |
| 1920 x 1080 | 6000-7800 kbps | source |

---

## Common CODECS Values

| Codec | Value |
|-------|-------|
| H.264 High L4.1 | `avc1.640029` |
| HEVC Main 10 L5.0 | `hvc1.2.4.L150.B0` |
| AAC-LC | `mp4a.40.2` |
| HE-AAC | `mp4a.40.5` |
| AC-3 | `ac-3` |
| E-AC-3 | `ec-3` |
| WebVTT | `wvtt` |

---

## Multivariant Playlist Example

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=4500000,AVERAGE-BANDWIDTH=4000000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0
720p/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.4d001f,mp4a.40.2",RESOLUTION=960x540,FRAME-RATE=30.0
540p/prog_index.m3u8

#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=375000,CODECS="avc1.640028",RESOLUTION=1280x720,URI="720p/iframe_index.m3u8"
```

---

## Key Authoring Requirements

1. **Keyframes:** Every 2 seconds
2. **Segments:** 6 seconds recommended, never exceed target + 0.5s
3. **Live playlists:** Must have EXT-X-PROGRAM-DATE-TIME
4. **VOD playlists:** Must have EXT-X-PLAYLIST-TYPE:VOD
5. **Multivariant:** Must include CODECS, RESOLUTION, FRAME-RATE, AVERAGE-BANDWIDTH
6. **Cellular:** Must have variant ≤ 192 kbps peak for iOS

---

## Apple Documentation

- [Using Apple's HLS Tools](https://developer.apple.com/documentation/http-live-streaming/using-apple-s-http-live-streaming-hls-tools)
- [HLS Authoring Specification](https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices)
- [Enabling Low-Latency HLS](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-http-live-streaming-hls)
- [HLS Protocol (IETF)](https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis)

For detailed man pages: `man mediafilesegmenter`, `man mediastreamvalidator`, etc.