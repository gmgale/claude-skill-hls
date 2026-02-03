<command-name>hls-live</command-name>
<description>Set up live HLS streaming using tsrecompressor and mediastreamsegmenter.</description>

# HLS Live Streaming Workflow

Guide users through setting up live HLS streams using Apple's streaming tools.

## Overview

Live streaming requires:
1. **Source:** Camera/encoder producing MPEG-2 Transport Stream via UDP
2. **Segmenter:** `mediastreamsegmenter` consuming the stream
3. **Web Server:** Serving segments and playlists to clients

## Prerequisites

```bash
# Verify tools
which mediastreamsegmenter tsrecompressor

# For testing without a real source, use tsrecompressor to generate test patterns
```

## Architecture

```
┌─────────────┐     UDP      ┌──────────────────────┐     HTTP     ┌─────────┐
│   Source    │ ──────────▶  │ mediastreamsegmenter │ ──────────▶  │ Clients │
│ (encoder)   │  multicast   │   (segmenter)        │   (nginx)    │         │
└─────────────┘              └──────────────────────┘              └─────────┘
```

## Basic Live Streaming Setup

### Step 1: Generate Test Stream (for testing)

Use `tsrecompressor` to generate a test pattern:

```bash
# Generate test stream to multicast address
tsrecompressor -g -O 224.0.0.50:9123
```

**Options:**
- `-g` - Generate test pattern (BITC video + LTC audio)
- `-c` - Capture from camera/microphone instead
- `-h` - Use hardware encoder (recommended)
- `-O <addr>:<port>` - Output to multicast address (mid quality ~4 Mbps)

### Step 2: Start Segmenter

```bash
mediastreamsegmenter \
  -f /var/www/html/live \
  -t 6 \
  -s 10 \
  -D \
  --format iso \
  224.0.0.50:9123
```

**Key Options:**
- `-f <path>` - Output directory (required)
- `-t <seconds>` - Target segment duration (default: 10)
- `-s <count>` - Sliding window size (segments to keep in playlist)
- `-D` - Delete expired segments
- `--format iso` - Use fMP4 format
- `<addr>:<port>` - UDP multicast address to receive stream

### Step 3: Create Multivariant Playlist (Manually for Live)

For live streams, create a simple multivariant playlist:

```m3u8
#EXTM3U
#EXT-X-VERSION:7

#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0
live/prog_index.m3u8
```

## Multi-Bitrate Live Streaming

### Step 1: Generate Multiple Quality Streams

```bash
# Terminal 1: Generate all quality levels
tsrecompressor -g -h -x -a \
  -L 224.0.0.50:9125 \
  -O 224.0.0.50:9123 \
  -H 224.0.0.50:9121
```

**Output options:**
| Flag | Quality | Bitrate | Default Port |
|------|---------|---------|--------------|
| `-H` | High | ~7.5 Mbps | 9121 |
| `-O` | Mid | ~4 Mbps | 9123 |
| `-L` | Low | ~2 Mbps | 9125 |
| `-P` | Preview | ~0.5 Mbps | - |

### Step 2: Start Segmenters for Each Quality

```bash
# Terminal 2: High quality
mediastreamsegmenter \
  -f /var/www/html/live/hi \
  -t 6 -s 10 -D -T \
  --format iso \
  224.0.0.50:9121

# Terminal 3: Mid quality
mediastreamsegmenter \
  -f /var/www/html/live/mid \
  -t 6 -s 10 -D -T \
  --format iso \
  224.0.0.50:9123

# Terminal 4: Low quality
mediastreamsegmenter \
  -f /var/www/html/live/lo \
  -t 6 -s 10 -D -T \
  --format iso \
  224.0.0.50:9125
```

### Step 3: Create Multivariant Playlist

Create `/var/www/html/live/master.m3u8`:

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=7500000,AVERAGE-BANDWIDTH=6500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1920x1080,FRAME-RATE=30.0
hi/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0
mid/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=960x540,FRAME-RATE=30.0
lo/prog_index.m3u8
```

## mediastreamsegmenter Options Reference

### Input/Output
| Option | Description |
|--------|-------------|
| `<addr>:<port>` | UDP multicast address to receive (required) |
| `-f <path>` | Output directory (required) |
| `-b <url>` | Base URL for playlist |
| `-o <name>` | Index filename (default: prog_index.m3u8) |

### Segment Control
| Option | Description |
|--------|-------------|
| `-t <seconds>` | Target segment duration (default: 10) |
| `-s <count>` | Sliding window size (default: 5) |
| `-D` | Delete expired segments |
| `-S` | Start segment numbering at 1 |

### Format Options
| Option | Description |
|--------|-------------|
| `--format iso` | Fragmented MP4 |
| `--format transport` | MPEG-2 TS (default) |

### Low-Latency Options
| Option | Description |
|--------|-------------|
| `-w <ms>` | Partial segment duration (enables LL-HLS) |
| `-W <ms>` | Partial segment max duration |

### VOD Capture
| Option | Description |
|--------|-------------|
| `-p` | Enable program capture (VOD from live) |
| `--program-duration <mins>` | Duration to capture |

### Other Options
| Option | Description |
|--------|-------------|
| `-T` | Generate program date-time tags |
| `-V` | Verbose output |
| `-z <name>` | I-frame playlist filename |
| `-M <seconds>` | Discontinuity timeout |

## tsrecompressor Options Reference

### Mode Selection
| Option | Description |
|--------|-------------|
| `-g` | Generate test pattern |
| `-c` | Capture from camera/mic |
| `-i <file>` | Input from file |

### Output Destinations
| Option | Description |
|--------|-------------|
| `-H <addr>:<port>` | High quality output (~7.5 Mbps) |
| `-O <addr>:<port>` | Mid quality output (~4 Mbps) |
| `-L <addr>:<port>` | Low quality output (~2 Mbps) |
| `-P <addr>:<port>` | Preview output (~0.5 Mbps) |

### Encoding Options
| Option | Description |
|--------|-------------|
| `-h` | Use hardware encoder |
| `-x` | Use HEVC (default is H.264) |
| `-a` | Include audio |

## Live Playlist Structure

Generated `prog_index.m3u8`:
```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:1847
#EXT-X-PROGRAM-DATE-TIME:2024-01-15T18:30:00.000Z
#EXT-X-MAP:URI="fileSequence0.mp4"
#EXTINF:6.006,
fileSequence1847.m4s
#EXTINF:6.006,
fileSequence1848.m4s
#EXTINF:6.006,
fileSequence1849.m4s
#EXTINF:6.006,
fileSequence1850.m4s
#EXTINF:6.006,
fileSequence1851.m4s
```

**Key differences from VOD:**
- No `EXT-X-PLAYLIST-TYPE` tag
- No `EXT-X-ENDLIST` (content is ongoing)
- `EXT-X-PROGRAM-DATE-TIME` MUST be present
- `EXT-X-MEDIA-SEQUENCE` increments as segments expire

## VOD Recording from Live

Capture a VOD recording while streaming live:

```bash
mediastreamsegmenter \
  -f /var/www/html/live \
  -t 6 -s 10 -D \
  -p --program-duration 60 \
  --format iso \
  224.0.0.50:9123
```

This creates a sliding-window live playlist plus captures 60 minutes for VOD.

## Web Server Configuration

### Nginx Example

```nginx
server {
    listen 80;
    server_name example.com;

    location /live/ {
        alias /var/www/html/live/;

        # CORS headers for HLS
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, OPTIONS';

        # Cache control for live content
        add_header Cache-Control "no-cache, no-store, must-revalidate";

        # MIME types
        types {
            application/vnd.apple.mpegurl m3u8;
            video/mp2t ts;
            video/mp4 mp4 m4s;
        }
    }
}
```

### Apache Example

```apache
<Directory "/var/www/html/live">
    Header set Access-Control-Allow-Origin "*"
    Header set Cache-Control "no-cache, no-store, must-revalidate"

    AddType application/vnd.apple.mpegurl .m3u8
    AddType video/mp2t .ts
    AddType video/mp4 .mp4 .m4s
</Directory>
```

## Complete Live Streaming Script

```bash
#!/bin/bash
# Multi-bitrate live streaming setup

OUTPUT_DIR="/var/www/html/live"
MULTICAST_BASE="224.0.0.50"

# Create directories
mkdir -p "$OUTPUT_DIR"/{hi,mid,lo}

# Start test stream generator (background)
tsrecompressor -g -h -a \
  -H "$MULTICAST_BASE:9121" \
  -O "$MULTICAST_BASE:9123" \
  -L "$MULTICAST_BASE:9125" &
GENERATOR_PID=$!

sleep 2  # Wait for stream to start

# Start segmenters (background)
mediastreamsegmenter -f "$OUTPUT_DIR/hi" -t 6 -s 10 -D -T --format iso "$MULTICAST_BASE:9121" &
mediastreamsegmenter -f "$OUTPUT_DIR/mid" -t 6 -s 10 -D -T --format iso "$MULTICAST_BASE:9123" &
mediastreamsegmenter -f "$OUTPUT_DIR/lo" -t 6 -s 10 -D -T --format iso "$MULTICAST_BASE:9125" &

# Create multivariant playlist
cat > "$OUTPUT_DIR/master.m3u8" << 'EOF'
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=7500000,AVERAGE-BANDWIDTH=6500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1920x1080,FRAME-RATE=30.0
hi/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0
mid/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=960x540,FRAME-RATE=30.0
lo/prog_index.m3u8
EOF

echo "Live stream available at: http://localhost/live/master.m3u8"
echo "Press Ctrl+C to stop..."

# Cleanup on exit
trap "kill $GENERATOR_PID 2>/dev/null; pkill -P $$" EXIT
wait
```

## Tips

1. **Use hardware encoding** (`-h`) for real-time performance
2. **Match segment boundaries** across all quality levels
3. **Use `-T` flag** to include program date-time (required for live)
4. **Set sliding window** (`-s`) to at least 6 segments
5. **Use `-D` flag** to automatically clean up old segments
6. **Monitor disk space** - segments accumulate quickly without `-D`
7. **Test latency** by comparing wall clock to program date-time

## Troubleshooting

### "No packets received"
- Check multicast address is correct
- Verify network allows multicast (may need igmp snooping)
- Try unicast address for testing

### High latency
- Reduce segment duration (`-t 4`)
- Consider Low-Latency HLS (`/hls-lowlatency`)
- Check encoder settings for keyframe interval

### Segments not deleting
- Ensure `-D` flag is set
- Check file permissions in output directory

### Playlist not updating
- mediastreamsegmenter may have crashed
- Check for error messages in verbose mode (`-V`)
