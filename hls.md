<command-name>hls</command-name>
<description>Help users work with Apple's HTTP Live Streaming (HLS) Tools for creating, validating, and deploying HLS and Low-Latency HLS streams on macOS.</description>

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

## Available Workflows

Route the user to the appropriate sub-skill based on their intent:

| User Intent | Sub-Skill | Description |
|-------------|-----------|-------------|
| Segment video files for on-demand streaming | `/hls-vod` | VOD segmentation with mediafilesegmenter |
| Set up live streaming | `/hls-live` | Live streaming with mediastreamsegmenter |
| Validate HLS streams | `/hls-validate` | Validation with mediastreamvalidator + hlsreport |
| Add subtitles/captions | `/hls-subtitles` | Subtitle segmentation with mediasubtitlesegmenter |
| Encrypt content | `/hls-encrypt` | Encryption configuration |
| Add metadata/ID3 tags | `/hls-metadata` | ID3 tag generation with id3taggenerator |
| Set up low-latency HLS | `/hls-lowlatency` | LL-HLS with partial segments |

## Quick Reference

### Three Components of HLS
1. **Server Component** - Encodes media, creates segments using mediafilesegmenter or mediastreamsegmenter
2. **Distribution Component** - Web server or CDN delivering segments over HTTP
3. **Client Software** - AVKit, AVFoundation, WebKit (iOS 3.0+, Safari 4.0+)

### Supported Formats
- **Video:** HEVC, H.264, Dolby Vision, AV1
- **Audio:** AAC, HE-AAC, xHE-AAC, AC-3, E-AC-3, Apple Lossless, FLAC
- **Container:** Fragmented MPEG-4 (fMP4) or MPEG-2 Transport Stream

### Core Tools Summary

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

## Standard Workflow Phases

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
# Validate stream
mediastreamvalidator -O validation_data.json -V <playlist_url>

# Generate report
hlsreport -V -o report.html validation_data.json
```

### Phase 4: Deployment
Configure web server MIME types:
- `.m3u8` → `application/vnd.apple.mpegurl`
- `.ts` → `video/mp2t`
- `.mp4/.m4s` → `video/mp4`

## Key Authoring Requirements

### Video Rules
- Key frames (IDRs) every 2 seconds
- Deinterlace all interlaced content
- Frame rates ≤ 60 fps
- Use consistent color space

### Segment Rules
- Target duration: 6 seconds recommended
- Segments must not exceed target by > 0.5 seconds
- Video segments must start with IDR frame

### Playlist Rules
- VOD playlists MUST have `EXT-X-PLAYLIST-TYPE:VOD`
- Live playlists MUST have `EXT-X-PROGRAM-DATE-TIME`
- Live playlists need minimum 6 segments
- Multivariant playlists MUST include CODECS, RESOLUTION, FRAME-RATE

## Apple Documentation References

- [Deploying a Basic HLS Stream](https://developer.apple.com/documentation/http-live-streaming/deploying-a-basic-http-live-streaming-hls-stream)
- [HLS Authoring Specification](https://developer.apple.com/documentation/http-live-streaming/hls-authoring-specification-for-apple-devices)
- [Using Apple's HLS Tools](https://developer.apple.com/documentation/http-live-streaming/using-apple-s-http-live-streaming-hls-tools)
- [Enabling Low-Latency HLS](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-http-live-streaming-hls)
- [HLS Protocol Specification (IETF)](https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis)

## Best Practices

1. **Always backup** source media before processing
2. **Validate early and often** using mediastreamvalidator
3. **Use hardware encoding** (`-h` flag) when available
4. **Set appropriate target durations** (4-6s live, 6-10s VOD)
5. **Include I-frame playlists** for trick play support
6. **Test on target devices** using `-d` flag in validator
7. **Use fMP4 format** (`--format iso`) for modern compatibility
8. **Fix all "Critical Must Fix" and "Must Fix"** validation issues

## Getting Help

For detailed man pages:
```bash
man mediafilesegmenter
man mediastreamsegmenter
man mediastreamvalidator
man hlsreport
```

Documentation is also available at `/usr/local/share/hlstools/`
