<command-name>hls-lowlatency</command-name>
<description>Set up Low-Latency HLS (LL-HLS) for reduced streaming delay.</description>

# Low-Latency HLS (LL-HLS)

Guide users through setting up Low-Latency HTTP Live Streaming for minimal playback delay.

## Overview

Low-Latency HLS reduces end-to-end latency from 20-30 seconds to 2-5 seconds through:
- **Partial segments** - Smaller chunks delivered before full segment completes
- **Blocking playlist requests** - Server holds request until new content available
- **Preload hints** - Client knows about next segment before it exists

## Requirements

- **Server:** Origin supporting blocking requests (not standard web server)
- **Encoder:** Sub-second latency to segmenter
- **CDN:** HTTP/2 support, blocking request support

## Latency Comparison

| Mode | Typical Latency |
|------|-----------------|
| Standard HLS | 20-30 seconds |
| Low-Latency HLS | 2-5 seconds |
| Apple Recommendation | < 3 seconds |

## Basic LL-HLS Setup

### Step 1: Configure Segmenter for Partial Segments

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

**Key LL-HLS Options:**
- `-w <ms>` - Partial segment duration (enables LL-HLS)
- `-W <ms>` - Maximum partial segment duration
- `-t 4` - Shorter target duration (4s recommended for LL-HLS)
- `-s 16` - Larger sliding window (more segments for catch-up)

**Recommended partial segment duration:** ~1 second (1000-1002ms)

### Step 2: Deploy LL-HLS Origin Server

Standard web servers cannot handle blocking playlist requests. Use:
- Apple's sample PHP/Go origin
- Commercial LL-HLS origins
- CDN with LL-HLS support

### Step 3: Create Multivariant Playlist

```m3u8
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0
hi/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=960x540,FRAME-RATE=30.0
mid/prog_index.m3u8
```

## LL-HLS Playlist Format

```m3u8
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-TARGETDURATION:4
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=3.0,CAN-SKIP-UNTIL=24.0
#EXT-X-PART-INF:PART-TARGET=1.002
#EXT-X-MEDIA-SEQUENCE:1847

#EXT-X-PROGRAM-DATE-TIME:2024-01-15T18:30:00.000Z
#EXT-X-MAP:URI="init.mp4"

#EXTINF:4.004,
fileSequence1847.m4s
#EXTINF:4.004,
fileSequence1848.m4s

#EXT-X-PART:DURATION=1.001,URI="fileSequence1849.0.m4s"
#EXT-X-PART:DURATION=1.001,URI="fileSequence1849.1.m4s"
#EXT-X-PART:DURATION=1.001,URI="fileSequence1849.2.m4s"
#EXT-X-PART:DURATION=1.001,URI="fileSequence1849.3.m4s",INDEPENDENT=YES
#EXTINF:4.004,
fileSequence1849.m4s

#EXT-X-PART:DURATION=1.001,URI="fileSequence1850.0.m4s"
#EXT-X-PART:DURATION=1.001,URI="fileSequence1850.1.m4s"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="fileSequence1850.2.m4s"
```

### Key LL-HLS Tags

| Tag | Purpose |
|-----|---------|
| `EXT-X-SERVER-CONTROL` | Server capabilities |
| `EXT-X-PART-INF` | Partial segment target duration |
| `EXT-X-PART` | Partial segment reference |
| `EXT-X-PRELOAD-HINT` | Upcoming partial segment |
| `EXT-X-SKIP` | Playlist delta updates |

### EXT-X-SERVER-CONTROL Attributes

| Attribute | Description |
|-----------|-------------|
| `CAN-BLOCK-RELOAD=YES` | Supports blocking playlist requests |
| `PART-HOLD-BACK=<seconds>` | How far back to start (3x part target) |
| `CAN-SKIP-UNTIL=<seconds>` | Supports playlist delta updates |
| `CAN-SKIP-DATERANGES=YES` | Can skip DATERANGE tags |

## Origin Server Requirements

### Blocking Playlist Requests

Client requests playlist with query parameters:
```
GET /live/prog_index.m3u8?_HLS_msn=1850&_HLS_part=2
```

Server holds response until:
- Requested media sequence number (msn) is available
- Requested partial segment (part) is complete

### PHP Origin Example (Simplified)

```php
<?php
// ll-hls-origin.php - Simplified LL-HLS origin

$playlistDir = '/var/www/html/live';
$timeout = 10; // seconds

// Parse request parameters
$msn = isset($_GET['_HLS_msn']) ? intval($_GET['_HLS_msn']) : null;
$part = isset($_GET['_HLS_part']) ? intval($_GET['_HLS_part']) : null;

// If blocking request, wait for content
if ($msn !== null) {
    $startTime = time();
    while (time() - $startTime < $timeout) {
        if (contentAvailable($playlistDir, $msn, $part)) {
            break;
        }
        usleep(100000); // 100ms
    }
}

// Serve playlist
header('Content-Type: application/vnd.apple.mpegurl');
header('Cache-Control: no-cache');
readfile("$playlistDir/prog_index.m3u8");

function contentAvailable($dir, $msn, $part) {
    // Check if requested segment/part exists
    $segmentFile = "$dir/fileSequence{$msn}.m4s";
    if ($part !== null) {
        $partFile = "$dir/fileSequence{$msn}.{$part}.m4s";
        return file_exists($partFile);
    }
    return file_exists($segmentFile);
}
?>
```

### Go Origin Example (Simplified)

```go
package main

import (
    "fmt"
    "net/http"
    "os"
    "strconv"
    "time"
)

func main() {
    http.HandleFunc("/live/", handlePlaylist)
    http.ListenAndServe(":8080", nil)
}

func handlePlaylist(w http.ResponseWriter, r *http.Request) {
    msnStr := r.URL.Query().Get("_HLS_msn")
    partStr := r.URL.Query().Get("_HLS_part")

    if msnStr != "" {
        msn, _ := strconv.Atoi(msnStr)
        part := -1
        if partStr != "" {
            part, _ = strconv.Atoi(partStr)
        }

        // Block until content available
        timeout := time.After(10 * time.Second)
        ticker := time.NewTicker(100 * time.Millisecond)
        defer ticker.Stop()

        for {
            select {
            case <-timeout:
                http.Error(w, "Timeout", http.StatusGatewayTimeout)
                return
            case <-ticker.C:
                if contentAvailable(msn, part) {
                    goto serve
                }
            }
        }
    }

serve:
    w.Header().Set("Content-Type", "application/vnd.apple.mpegurl")
    w.Header().Set("Cache-Control", "no-cache")
    data, _ := os.ReadFile("/var/www/html/live/prog_index.m3u8")
    w.Write(data)
}

func contentAvailable(msn, part int) bool {
    filename := fmt.Sprintf("/var/www/html/live/fileSequence%d", msn)
    if part >= 0 {
        filename = fmt.Sprintf("%s.%d.m4s", filename, part)
    } else {
        filename = filename + ".m4s"
    }
    _, err := os.Stat(filename)
    return err == nil
}
```

## CDN Configuration

### Requirements
- HTTP/2 support (for multiplexed requests)
- Blocking request support or bypass to origin
- Short cache TTL for playlists (< 1 second)

### Cloudflare Workers Example

```javascript
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
    const url = new URL(request.url)

    // Forward blocking requests to origin
    if (url.searchParams.has('_HLS_msn')) {
        return fetch(request, {
            cf: { cacheTtl: 0 }
        })
    }

    // Cache non-blocking requests briefly
    return fetch(request, {
        cf: { cacheTtl: 1 }
    })
}
```

## Multi-Bitrate LL-HLS

```bash
#!/bin/bash
# Multi-bitrate LL-HLS setup

OUTPUT_DIR="/var/www/html/live"
PART_DURATION=1002  # ~1 second

# Start test stream generator
tsrecompressor -g -h -a \
  -H 224.0.0.50:9121 \
  -O 224.0.0.50:9123 \
  -L 224.0.0.50:9125 &

sleep 2

# Segment each quality with partial segments
mediastreamsegmenter -w $PART_DURATION -t 4 -s 16 -D -T \
  --format iso -f "$OUTPUT_DIR/hi" 224.0.0.50:9121 &

mediastreamsegmenter -w $PART_DURATION -t 4 -s 16 -D -T \
  --format iso -f "$OUTPUT_DIR/mid" 224.0.0.50:9123 &

mediastreamsegmenter -w $PART_DURATION -t 4 -s 16 -D -T \
  --format iso -f "$OUTPUT_DIR/lo" 224.0.0.50:9125 &

# Create multivariant playlist
cat > "$OUTPUT_DIR/master.m3u8" << 'EOF'
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-STREAM-INF:BANDWIDTH=7500000,AVERAGE-BANDWIDTH=6500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1920x1080,FRAME-RATE=30.0
hi/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0
mid/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=960x540,FRAME-RATE=30.0
lo/prog_index.m3u8
EOF

echo "LL-HLS stream available at: http://localhost/live/master.m3u8"
wait
```

## Playlist Delta Updates (EXT-X-SKIP)

For very low latency, clients can request delta updates:

```
GET /live/prog_index.m3u8?_HLS_skip=YES
```

Server responds with:
```m3u8
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-TARGETDURATION:4
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,CAN-SKIP-UNTIL=24.0

#EXT-X-SKIP:SKIPPED-SEGMENTS=5

#EXTINF:4.004,
fileSequence1852.m4s
#EXT-X-PART:DURATION=1.001,URI="fileSequence1853.0.m4s"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="fileSequence1853.1.m4s"
```

## Rendition Reports

For seamless quality switching, use rendition reports:

```m3u8
#EXT-X-RENDITION-REPORT:URI="../mid/prog_index.m3u8",LAST-MSN=1850,LAST-PART=2
#EXT-X-RENDITION-REPORT:URI="../lo/prog_index.m3u8",LAST-MSN=1850,LAST-PART=2
```

This tells clients what's available on other renditions without extra requests.

## Validation

```bash
mediastreamvalidator \
  -V \
  -O validation.json \
  http://example.com/live/master.m3u8

hlsreport -V -o report.html validation.json
```

Check for:
- EXT-X-PART-INF present
- PART-HOLD-BACK >= 3x part target
- Partial segments have correct duration
- INDEPENDENT=YES on first part of each GOP

## Tips

1. **Use fMP4 format** (`--format iso`) for best LL-HLS support
2. **Set PART-HOLD-BACK** to at least 3x part target duration
3. **Include INDEPENDENT=YES** on I-frame parts
4. **Deploy proper origin** - standard web servers won't work
5. **Use HTTP/2** for efficient multiplexed requests
6. **Monitor latency** - measure glass-to-glass delay
7. **Test with Safari** - native LL-HLS support

## Latency Optimization

| Factor | Recommendation |
|--------|----------------|
| Part duration | 0.5-1.0 seconds |
| Segment duration | 4 seconds |
| Encoder latency | < 1 second |
| Origin | Use blocking requests |
| CDN | HTTP/2, short TTL |

## Troubleshooting

### High latency despite LL-HLS
- Check origin supports blocking requests
- Verify CDN isn't caching playlists
- Reduce encoder latency
- Check PART-HOLD-BACK setting

### Playback stuttering
- Increase part duration slightly
- Add more segments to sliding window (`-s`)
- Check network bandwidth

### Parts not appearing in playlist
- Verify `-w` flag is set
- Check fMP4 format is used
- Ensure encoder GOP structure is correct

### "Playlist update too slow"
- Origin not supporting blocking requests
- CDN caching playlist too long
- Network latency to origin

## Apple Documentation

- [Enabling Low-Latency HLS](https://developer.apple.com/documentation/http-live-streaming/enabling-low-latency-http-live-streaming-hls)
- [WWDC 2019: Introducing Low-Latency HLS](https://developer.apple.com/videos/play/wwdc2019/502/)
- [WWDC 2020: What's new in Low-Latency HLS](https://developer.apple.com/videos/play/wwdc2020/10228/)
