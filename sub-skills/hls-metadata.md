<command-name>hls-metadata</command-name>
<description>Generate and inject ID3 metadata tags into HLS streams using id3taggenerator.</description>

# HLS Metadata and ID3 Tags

Guide users through adding timed metadata to HLS streams.

## Overview

HLS supports timed metadata through:
- **ID3 tags** - Embedded in transport stream segments
- **EXT-X-DATERANGE** - Playlist-level metadata
- **JSON chapters** - Chapter markers

## Tool: id3taggenerator

Creates ID3 metadata tags for injection into live streams.

## Basic Usage

### Generate ID3 Tag File

```bash
id3taggenerator \
  -o metadata.id3 \
  -t "Now playing: Artist - Song Title"
```

### Send to Live Stream

```bash
id3taggenerator \
  -a 127.0.0.1:5555 \
  -t "Breaking News: Event Details"
```

The `-a` option sends tags directly to mediastreamsegmenter via UDP.

## id3taggenerator Options

### Output Options
| Option | Description |
|--------|-------------|
| `-o <file>` | Output to file |
| `-a <addr>:<port>` | Send to mediastreamsegmenter |

### Content Options
| Option | Description |
|--------|-------------|
| `-t <text>` | Custom text frame (TXXX) |
| `--title <string>` | Title (TIT2) |
| `--artist <string>` | Artist (TPE1) |
| `--album <string>` | Album (TALB) |
| `-i <file>` | Picture frame (JPEG/PNG) |

### Behavior Options
| Option | Description |
|--------|-------------|
| `-r` | Repeat at every segment boundary |
| `-q` | Quiet mode |

## mediastreamsegmenter ID3 Setup

Configure mediastreamsegmenter to receive ID3 tags:

```bash
mediastreamsegmenter \
  -f /var/www/html/live \
  -t 6 \
  -s 10 \
  -D \
  --meta-port 5555 \
  --format transport \
  224.0.0.50:9123
```

The `--meta-port` option enables UDP listener for ID3 injection.

**Note:** ID3 tags require transport stream format (`--format transport`).

## Common Use Cases

### Now Playing Information

```bash
# One-time update
id3taggenerator \
  -a 127.0.0.1:5555 \
  --title "Song Title" \
  --artist "Artist Name" \
  --album "Album Name"

# With album art
id3taggenerator \
  -a 127.0.0.1:5555 \
  --title "Song Title" \
  --artist "Artist Name" \
  -i album_cover.jpg
```

### Repeating Metadata

For continuous display (e.g., station ID):

```bash
id3taggenerator \
  -a 127.0.0.1:5555 \
  -r \
  -t "WXYZ Radio - Your Music Station"
```

The `-r` flag repeats the tag at each segment boundary.

### Custom Text Data

```bash
id3taggenerator \
  -a 127.0.0.1:5555 \
  -t '{"event":"ad_break","duration":30,"id":"ad123"}'
```

Custom JSON in text frames allows rich metadata for client-side processing.

### Program Information

```bash
id3taggenerator \
  -a 127.0.0.1:5555 \
  -t "Program: Evening News" \
  --title "Evening News" \
  --artist "News Team"
```

## EXT-X-DATERANGE (Playlist Metadata)

For playlist-level timed metadata, use EXT-X-DATERANGE:

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6

#EXT-X-DATERANGE:ID="ad1",START-DATE="2024-01-15T18:30:00.000Z",DURATION=30.0,X-AD-ID="commercial_123"

#EXT-X-PROGRAM-DATE-TIME:2024-01-15T18:30:00.000Z
#EXTINF:6.0,
segment1.ts
```

**Attributes:**
- `ID` - Unique identifier
- `START-DATE` - ISO 8601 timestamp
- `DURATION` - Duration in seconds
- `END-DATE` - End time (alternative to DURATION)
- `X-<name>` - Custom attributes

### Use Cases for EXT-X-DATERANGE
- Ad break markers
- Chapter markers
- Program boundaries
- Interstitial content

## JSON Chapters

For chapter metadata, create a WebVTT file with JSON cues:

### chapters.vtt
```vtt
WEBVTT

00:00:00.000 --> 00:05:00.000
{
  "title": "Introduction",
  "image": "https://example.com/chapter1.jpg"
}

00:05:00.000 --> 00:15:00.000
{
  "title": "Main Content",
  "image": "https://example.com/chapter2.jpg"
}

00:15:00.000 --> 00:20:00.000
{
  "title": "Conclusion",
  "image": "https://example.com/chapter3.jpg"
}
```

### Add to Multivariant Playlist

```m3u8
#EXTM3U

#EXT-X-SESSION-DATA:DATA-ID="com.example.chapters",URI="chapters.vtt"

#EXT-X-STREAM-INF:BANDWIDTH=4000000,...
hi/prog_index.m3u8
```

## Client-Side Metadata Handling

### JavaScript (HLS.js)

```javascript
const hls = new Hls();
hls.on(Hls.Events.FRAG_PARSING_METADATA, (event, data) => {
    data.samples.forEach(sample => {
        // Parse ID3 tag
        const id3 = parseID3(sample.data);
        console.log('Metadata:', id3);
        // Update UI with now playing info
    });
});
```

### AVFoundation (iOS/macOS)

```swift
let player = AVPlayer(url: streamURL)

player.currentItem?.asset.loadMetadata(for: .id3Metadata) { metadata, error in
    guard let metadata = metadata else { return }
    for item in metadata {
        if let value = item.value as? String {
            print("ID3: \(value)")
        }
    }
}
```

## Complete Live Streaming with Metadata

```bash
#!/bin/bash
# Live stream with dynamic metadata injection

OUTPUT_DIR="/var/www/html/live"
MULTICAST="224.0.0.50:9123"
META_PORT=5555

# Start segmenter with metadata port
mediastreamsegmenter \
  -f "$OUTPUT_DIR" \
  -t 6 \
  -s 10 \
  -D \
  -T \
  --meta-port $META_PORT \
  --format transport \
  "$MULTICAST" &

SEGMENTER_PID=$!

# Wait for segmenter to start
sleep 2

# Send initial station ID (repeating)
id3taggenerator \
  -a 127.0.0.1:$META_PORT \
  -r \
  -t "WXYZ Radio"

echo "Segmenter running (PID: $SEGMENTER_PID)"
echo "Send metadata: id3taggenerator -a 127.0.0.1:$META_PORT -t 'Your text'"

# Cleanup on exit
trap "kill $SEGMENTER_PID 2>/dev/null" EXIT
wait $SEGMENTER_PID
```

### Update Now Playing

```bash
# Script to update now playing from playlist
while read line; do
    artist=$(echo "$line" | cut -d'|' -f1)
    title=$(echo "$line" | cut -d'|' -f2)

    id3taggenerator \
      -a 127.0.0.1:5555 \
      --artist "$artist" \
      --title "$title"

    sleep 180  # Song duration
done < playlist.txt
```

## ID3 Tag Types Reference

### Text Information Frames
| Frame | Description |
|-------|-------------|
| TIT2 | Title |
| TPE1 | Artist |
| TALB | Album |
| TYER | Year |
| TCON | Genre |
| TXXX | User-defined text |

### Other Frames
| Frame | Description |
|-------|-------------|
| APIC | Attached picture |
| PRIV | Private frame |
| COMM | Comments |

## Tips

1. **Use transport stream format** - ID3 requires TS, not fMP4
2. **Test metadata display** on target players
3. **Keep text concise** - Limited display space
4. **Use JSON in TXXX** for structured data
5. **Implement client-side parsing** for custom metadata
6. **Consider EXT-X-DATERANGE** for fMP4 streams

## Troubleshooting

### Metadata not appearing
- Verify `--format transport` (ID3 requires TS)
- Check `--meta-port` matches id3taggenerator destination
- Confirm UDP connectivity between tools

### Metadata delayed
- Tags appear at segment boundaries
- Reduce segment duration for faster updates

### Client not receiving
- Verify client supports ID3 metadata
- Check player event handlers
- Test with Safari (native HLS support)

### Tags truncated
- ID3 frames have size limits
- Use shorter text for titles
- Optimize image file sizes
