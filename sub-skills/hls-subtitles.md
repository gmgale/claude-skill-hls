<command-name>hls-subtitles</command-name>
<description>Segment subtitles and captions for HLS using mediasubtitlesegmenter.</description>

# HLS Subtitle Segmentation

Guide users through adding subtitles and captions to HLS streams.

## Overview

HLS supports multiple subtitle formats:
- **WebVTT** - Recommended, best compatibility
- **IMSC1** - Advanced styling, TTML-based
- **tx3g** - QuickTime text track (converted to WebVTT)
- **SRT** - SubRip format (converted to WebVTT)

## Tool: mediasubtitlesegmenter

Segments subtitle files into HLS-compatible chunks.

## Basic Usage

```bash
mediasubtitlesegmenter \
  -f output/subtitles/en \
  -t 6 \
  -D 3600 \
  subtitles_en.vtt
```

**Required Options:**
- `-f <path>` - Output directory
- Input file (last argument)

**Common Options:**
| Option | Description |
|--------|-------------|
| `-t <seconds>` | Target segment duration (default: 60) |
| `-D <seconds>` | Overall presentation duration |
| `-o <name>` | Index filename (default: prog_index.m3u8) |
| `-m <offset>` | MPEG timestamp offset |
| `-b <url>` | Base URL for playlist entries |

## WebVTT Format

WebVTT is the recommended format for HLS subtitles.

### Basic WebVTT Structure

```vtt
WEBVTT

00:00:01.000 --> 00:00:04.000
Hello, and welcome to the stream.

00:00:05.000 --> 00:00:08.000
Today we'll be discussing HLS subtitles.

00:00:09.500 --> 00:00:12.000
Let's get started!
```

### WebVTT with Styling

```vtt
WEBVTT

STYLE
::cue {
  background-color: rgba(0,0,0,0.8);
  color: white;
  font-family: sans-serif;
}
::cue(.speaker1) {
  color: yellow;
}

00:00:01.000 --> 00:00:04.000
<c.speaker1>Speaker 1:</c> Hello everyone.

00:00:05.000 --> 00:00:08.000 align:left
This text is left-aligned.
```

## Complete Subtitle Workflow

### Step 1: Prepare Subtitle Files

Create WebVTT files for each language:

**English (subtitles_en.vtt):**
```vtt
WEBVTT

00:00:01.000 --> 00:00:04.000
Welcome to our presentation.

00:00:05.000 --> 00:00:08.000
Today's topic is HLS streaming.
```

**Spanish (subtitles_es.vtt):**
```vtt
WEBVTT

00:00:01.000 --> 00:00:04.000
Bienvenidos a nuestra presentación.

00:00:05.000 --> 00:00:08.000
El tema de hoy es streaming HLS.
```

### Step 2: Segment Subtitles

```bash
# Get video duration (in seconds)
DURATION=3600  # 1 hour

# Segment English subtitles
mediasubtitlesegmenter \
  -f output/subtitles/en \
  -t 6 \
  -D $DURATION \
  subtitles_en.vtt

# Segment Spanish subtitles
mediasubtitlesegmenter \
  -f output/subtitles/es \
  -t 6 \
  -D $DURATION \
  subtitles_es.vtt
```

### Step 3: Add to Multivariant Playlist

Add subtitle tracks using `EXT-X-MEDIA`:

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

# Subtitle tracks
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",DEFAULT=YES,AUTOSELECT=YES,FORCED=NO,LANGUAGE="en",URI="subtitles/en/prog_index.m3u8"
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="Spanish",DEFAULT=NO,AUTOSELECT=YES,FORCED=NO,LANGUAGE="es",URI="subtitles/es/prog_index.m3u8"

# Video variants with subtitles
#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,CODECS="avc1.640028,mp4a.40.2,wvtt",RESOLUTION=1280x720,FRAME-RATE=30.0,SUBTITLES="subs"
hi/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.640028,mp4a.40.2,wvtt",RESOLUTION=960x540,FRAME-RATE=30.0,SUBTITLES="subs"
mid/prog_index.m3u8
```

**Key attributes:**
- `TYPE=SUBTITLES` - Subtitle track
- `GROUP-ID` - Links subtitles to video variants
- `NAME` - Display name in player UI
- `DEFAULT=YES` - Default subtitle track
- `AUTOSELECT=YES` - Auto-select based on user locale
- `FORCED=NO` - Not forced subtitles (burned in)
- `LANGUAGE` - ISO 639-1 language code
- `URI` - Path to subtitle playlist

## Closed Captions vs Subtitles

### Subtitles (EXT-X-MEDIA TYPE=SUBTITLES)
- Separate WebVTT files
- Can be turned on/off by user
- Multiple languages supported

### Closed Captions (EXT-X-MEDIA TYPE=CLOSED-CAPTIONS)
- Embedded in video stream (CEA-608/708)
- Extracted by player
- Declare with `INSTREAM-ID`:

```m3u8
#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS,GROUP-ID="cc",NAME="English",DEFAULT=YES,LANGUAGE="en",INSTREAM-ID="CC1"

#EXT-X-STREAM-INF:BANDWIDTH=4000000,...,CLOSED-CAPTIONS="cc"
hi/prog_index.m3u8
```

## Forced Subtitles

For foreign language segments or signs:

```m3u8
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English (Forced)",DEFAULT=NO,AUTOSELECT=YES,FORCED=YES,LANGUAGE="en",URI="subtitles/en-forced/prog_index.m3u8"
```

`FORCED=YES` means subtitles display automatically when needed (e.g., foreign dialogue).

## Converting SRT to WebVTT

If you have SRT files, convert to WebVTT:

### Manual Conversion
```bash
# Simple conversion
echo "WEBVTT" > output.vtt
echo "" >> output.vtt
sed 's/,/./g' input.srt >> output.vtt
```

### Using FFmpeg
```bash
ffmpeg -i input.srt output.vtt
```

## Output Structure

After segmentation:

```
output/subtitles/en/
├── prog_index.m3u8          # Subtitle playlist
├── fileSequence0.webvtt     # Subtitle segments
├── fileSequence1.webvtt
└── ...
```

Generated `prog_index.m3u8`:
```m3u8
#EXTM3U
#EXT-X-TARGETDURATION:6
#EXT-X-VERSION:3
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:6.0,
fileSequence0.webvtt
#EXTINF:6.0,
fileSequence1.webvtt
#EXTINF:6.0,
fileSequence2.webvtt
#EXT-X-ENDLIST
```

## Synchronization Tips

### Matching Video Segments
- Use same target duration (`-t`) as video
- Set correct presentation duration (`-D`)

### Timestamp Offset
If subtitles start at different point than video:
```bash
mediasubtitlesegmenter \
  -f output/subtitles/en \
  -t 6 \
  -D 3600 \
  -m 90000 \
  subtitles_en.vtt
```

The `-m` option sets MPEG timestamp offset (90000 = 1 second at 90kHz).

## IMSC1 Subtitles

For advanced styling needs, use IMSC1 (TTML-based):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<tt xml:lang="en" xmlns="http://www.w3.org/ns/ttml"
    xmlns:tts="http://www.w3.org/ns/ttml#styling">
  <head>
    <styling>
      <style xml:id="default" tts:fontFamily="sansSerif"
             tts:fontSize="100%" tts:textAlign="center"/>
    </styling>
  </head>
  <body>
    <div>
      <p begin="00:00:01.000" end="00:00:04.000">
        Welcome to our presentation.
      </p>
      <p begin="00:00:05.000" end="00:00:08.000">
        Today's topic is HLS streaming.
      </p>
    </div>
  </body>
</tt>
```

Declare IMSC1 in playlist with codec `stpp.ttml.im1t`.

## Validation

Validate subtitle integration:

```bash
mediastreamvalidator -V http://example.com/master.m3u8
```

Check for:
- Subtitle playlists accessible
- Duration matches video
- Language codes valid
- CODECS includes `wvtt` for WebVTT

## Complete Example Script

```bash
#!/bin/bash
# Add subtitles to HLS stream

VIDEO_DURATION=3600  # seconds
OUTPUT_DIR="output"
SEGMENT_DURATION=6

# Segment subtitles for each language
for lang in en es fr; do
    if [ -f "subtitles_${lang}.vtt" ]; then
        echo "Segmenting $lang subtitles..."
        mkdir -p "$OUTPUT_DIR/subtitles/$lang"
        mediasubtitlesegmenter \
            -f "$OUTPUT_DIR/subtitles/$lang" \
            -t $SEGMENT_DURATION \
            -D $VIDEO_DURATION \
            "subtitles_${lang}.vtt"
    fi
done

echo "Done! Add subtitle tracks to your multivariant playlist."
```

## Tips

1. **Use WebVTT** for maximum compatibility
2. **Match segment duration** with video segments
3. **Set presentation duration** (`-D`) accurately
4. **Include CODECS** attribute with `wvtt`
5. **Validate after adding** subtitles to playlist
6. **Test on target devices** - subtitle rendering varies
7. **Use FORCED=YES** for essential foreign dialogue

## Troubleshooting

### Subtitles not appearing
- Check URI path in EXT-X-MEDIA
- Verify GROUP-ID matches SUBTITLES attribute
- Confirm playlist is accessible

### Out of sync
- Check `-D` matches actual video duration
- Verify WebVTT timestamps are correct
- Use `-m` for timestamp offset if needed

### Styling not applied
- WebVTT styling support varies by player
- Consider IMSC1 for advanced styling
- Test on target devices

### "Subtitle track too long"
- Subtitle playlist duration should match video
- Use `-D` option with correct duration
