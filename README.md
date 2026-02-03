# HLS Tools Skill for Claude Code

A Claude Code skill for working with Apple's HTTP Live Streaming (HLS) Tools on macOS. This skill provides expert guidance for creating, validating, and deploying HLS and Low-Latency HLS streams.

## Features

- **VOD Workflow** - Segment video files with `mediafilesegmenter` and create multivariant playlists
- **Live Streaming** - Set up live streams with `tsrecompressor` and `mediastreamsegmenter`
- **Validation** - Validate streams with `mediastreamvalidator` and generate reports with `hlsreport`
- **Low-Latency HLS** - Configure partial segments for 2-5 second latency
- **Subtitles** - Segment WebVTT/SRT files with `mediasubtitlesegmenter`
- **Encryption** - AES-128 encryption with key rotation
- **Metadata** - ID3 tag injection with `id3taggenerator`
- **Reference Data** - Complete bitrate ladders, CODECS values, and validation rules

## Prerequisites

### 1. Install HLS Tools

Download and install "HTTP Live Streaming Tools for macOS Sequoia and later" from Apple:

https://developer.apple.com/download/all/?q=HTTP%20Live%20Streaming%20Tools

After installation, tools are available at:
- **Binaries:** `/usr/local/bin/`
- **Documentation:** `/usr/local/share/hlstools/`

### 2. Verify Installation

```bash
which mediafilesegmenter mediastreamsegmenter mediastreamvalidator hlsreport variantplaylistcreator
```

## Installation

### Option 1: Copy to Claude Code Skills Directory

```bash
# Create skills directory if it doesn't exist
mkdir -p ~/.claude/skills/hls

# Copy the skill file
cp SKILL.md ~/.claude/skills/hls/
```

### Option 2: Clone and Install

```bash
git clone https://github.com/yourusername/claude-hls-skill.git
mkdir -p ~/.claude/skills/hls
cp claude-hls-skill/SKILL.md ~/.claude/skills/hls/
```

## Usage

Once installed, invoke the skill in Claude Code:

```
/hls
```

Or ask Claude Code questions about HLS streaming - it will use the skill automatically when relevant.

### Example Prompts

- "Help me segment a video file for HLS streaming"
- "Set up a multi-bitrate VOD stream"
- "Validate my HLS stream and generate a report"
- "Configure low-latency HLS for live streaming"
- "Add subtitles to my HLS stream"
- "Set up encryption for my HLS content"

## Project Structure

```
claude-hls-skill/
├── README.md                    # This file
├── hls.md                       # Original skill draft
├── SKILL.md                     # Main skill file (install this)
├── sub-skills/
│   ├── hls-vod.md              # VOD workflow details
│   ├── hls-live.md             # Live streaming workflow
│   ├── hls-validate.md         # Validation and reporting
│   ├── hls-subtitles.md        # Subtitle segmentation
│   ├── hls-encrypt.md          # Encryption setup
│   ├── hls-metadata.md         # ID3 tag generation
│   └── hls-lowlatency.md       # Low-latency HLS
└── reference/
    ├── bitrate-ladders.md      # H.264/HEVC encoding tables
    ├── tool-options.md         # Complete tool reference
    └── validation-rules.md     # hlsreport rule sets
```

## Quick Reference

### Core Tools

| Tool | Purpose |
|------|---------|
| `mediafilesegmenter` | Segment files for VOD |
| `mediastreamsegmenter` | Segment live streams |
| `mediastreamvalidator` | Validate HLS streams |
| `hlsreport` | Generate HTML reports |
| `variantplaylistcreator` | Create multivariant playlists |
| `mediasubtitlesegmenter` | Segment subtitles |
| `tsrecompressor` | Generate/capture test streams |
| `id3taggenerator` | Create ID3 metadata |

### Common Commands

**Segment a VOD file:**
```bash
mediafilesegmenter -f output -t 6 --format iso -z iframe_index.m3u8 input.mp4
```

**Validate a stream:**
```bash
mediastreamvalidator -O validation.json -V http://example.com/master.m3u8
hlsreport -V -o report.html validation.json
```

**Start a live test stream:**
```bash
tsrecompressor -g -h -a -O 224.0.0.50:9123
mediastreamsegmenter -f /var/www/live -t 6 -s 10 -D -T --format iso 224.0.0.50:9123
```

### Bitrate Ladder (H.264)

| Resolution | Bitrate | Use Case |
|------------|---------|----------|
| 416 x 234 | 145 kbps | Cellular fallback |
| 640 x 360 | 365 kbps | Low bandwidth |
| 960 x 540 | 2000 kbps | Default Wi-Fi |
| 1280 x 720 | 3000-4500 kbps | HD |
| 1920 x 1080 | 6000-7800 kbps | Full HD |

### Key Requirements

- Keyframes every 2 seconds
- Target segment duration: 6 seconds
- Include CODECS, RESOLUTION, FRAME-RATE in multivariant playlist
- iOS cellular: must have variant ≤ 192 kbps peak

## Apple Documentation

- [Using Apple's HLS Tools](https://developer.apple.com/documentation/http-live-streaming/using-apple-s-http-live-streaming-hls-tools)
- [HLS Authoring Specification](https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices)
- [Enabling Low-Latency HLS](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-http-live-streaming-hls)
- [HLS Protocol (IETF)](https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis)

## License

MIT License - See LICENSE file for details.

## Contributing

Contributions welcome! Please submit issues and pull requests.

## Acknowledgments

- Apple for the HLS Tools and documentation
- Anthropic for Claude Code
