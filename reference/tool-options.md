# HLS Tools Command Reference

Complete option reference for Apple's HTTP Live Streaming tools.

## mediafilesegmenter

Segments media files (MOV, MP4, M4V, M4A, MP3) for VOD streaming.

### Basic Usage
```bash
mediafilesegmenter [options] <input_file>
```

### Output Options
| Option | Description | Default |
|--------|-------------|---------|
| `-f <path>` | Output directory (required) | - |
| `-b <url>` | Base URL for playlist entries | - |
| `-B <url>` | Base URL for key files | - |
| `-o <name>` | Index filename | `prog_index.m3u8` |

### Format Options
| Option | Description |
|--------|-------------|
| `--format iso` | Fragmented MP4 (fMP4) - recommended |
| `--format transport` | MPEG-2 Transport Stream |
| `--format packed` | Packed audio (AAC in ADTS) |
| `--format cmaf` | Common Media Application Format |

### Segment Options
| Option | Description | Default |
|--------|-------------|---------|
| `-t <seconds>` | Target segment duration | 10 |
| `-z <name>` | I-frame playlist filename | - |
| `-I` | Generate I-frame playlist only | - |

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
| `-F <file>` | Key info file for sample-AES |

### Other Options
| Option | Description |
|--------|-------------|
| `-V` | Verbose output |
| `-q` | Quiet mode |
| `-s <WxH>` | Scale video to width x height |

### Examples
```bash
# Basic VOD segmentation
mediafilesegmenter -f output -t 6 --format iso input.mp4

# With I-frame playlist
mediafilesegmenter -f output -t 6 --format iso -z iframe_index.m3u8 input.mp4

# Audio only
mediafilesegmenter -f output/audio -t 6 --format iso -a input.mp4

# With encryption
mediafilesegmenter -f output -t 6 --format iso -k key.key -K https://keys.example.com/ input.mp4
```

---

## mediastreamsegmenter

Segments live MPEG-2 transport streams for live/VOD streaming.

### Basic Usage
```bash
mediastreamsegmenter [options] <addr>:<port>
```

### Input/Output
| Option | Description | Default |
|--------|-------------|---------|
| `<addr>:<port>` | UDP multicast address (required) | - |
| `-f <path>` | Output directory (required) | - |
| `-b <url>` | Base URL for playlist entries | - |
| `-o <name>` | Index filename | `prog_index.m3u8` |

### Segment Control
| Option | Description | Default |
|--------|-------------|---------|
| `-t <seconds>` | Target segment duration | 10 |
| `-s <count>` | Sliding window size | 5 |
| `-D` | Delete expired segments | off |
| `-S` | Start numbering at 1 | 0 |

### Format Options
| Option | Description |
|--------|-------------|
| `--format iso` | Fragmented MP4 (fMP4) |
| `--format transport` | MPEG-2 Transport Stream (default) |

### Low-Latency Options
| Option | Description |
|--------|-------------|
| `-w <ms>` | Partial segment duration (enables LL-HLS) |
| `-W <ms>` | Max partial segment duration |

### VOD Capture
| Option | Description |
|--------|-------------|
| `-p` | Enable program capture |
| `--program-duration <mins>` | Duration to capture |

### Metadata
| Option | Description |
|--------|-------------|
| `-T` | Include program date-time tags |
| `--meta-port <port>` | UDP port for ID3 metadata |
| `-z <name>` | I-frame playlist filename |
| `-M <seconds>` | Discontinuity timeout |

### Other Options
| Option | Description |
|--------|-------------|
| `-V` | Verbose output |
| `-q` | Quiet mode |

### Examples
```bash
# Basic live segmentation
mediastreamsegmenter -f /var/www/live -t 6 -s 10 -D 224.0.0.50:9123

# With fMP4 format
mediastreamsegmenter -f /var/www/live -t 6 -s 10 -D --format iso 224.0.0.50:9123

# Low-latency HLS
mediastreamsegmenter -f /var/www/live -t 4 -s 16 -D -w 1002 --format iso 224.0.0.50:9123

# VOD capture from live
mediastreamsegmenter -f /var/www/live -t 6 -p --program-duration 60 224.0.0.50:9123
```

---

## mediastreamvalidator

Validates HLS streams against Apple's specification.

### Basic Usage
```bash
mediastreamvalidator [options] <playlist_url>
```

### Output Options
| Option | Description | Default |
|--------|-------------|---------|
| `-O <path>` | JSON output file | - |
| `-V` | Verbose console output | off |
| `-q` | Quiet mode | off |
| `-j` | Detailed JSON output | off |

### Validation Control
| Option | Description | Default |
|--------|-------------|---------|
| `-t <seconds>` | Validation timeout | 300 |
| `-p` | Parse playlist only (no download) | off |
| `-d <device>` | Simulate device | - |

### Device Options
| Value | Description |
|-------|-------------|
| `ipod` | iPod touch |
| `iphone` | iPhone |
| `ipad` | iPad |
| `atv` | Apple TV |
| `visionpro` | Apple Vision Pro |

### Network Options
| Option | Description |
|--------|-------------|
| `-u <user:pass>` | HTTP authentication |
| `-c <path>` | Client certificate |
| `-r <url>` | HTTP referer |
| `-s` | Skip SSL validation |

### Examples
```bash
# Basic validation
mediastreamvalidator http://example.com/master.m3u8

# With JSON output
mediastreamvalidator -O validation.json -V http://example.com/master.m3u8

# Simulate iPhone
mediastreamvalidator -d iphone -O ios_validation.json http://example.com/master.m3u8

# Playlist only (quick check)
mediastreamvalidator -p http://example.com/master.m3u8
```

---

## hlsreport

Generates HTML reports from validation JSON.

### Basic Usage
```bash
hlsreport [options] <validation_json>
```

### Output Options
| Option | Description | Default |
|--------|-------------|---------|
| `-o <file>` | Output HTML file | stdout |
| `-V` | Verbose (detailed tables) | off |

### Rule Sets
| Option | Description |
|--------|-------------|
| `-R standard` | General HLS requirements |
| `-R iOS` | iOS-specific requirements |
| `-R tvOS` | tvOS-specific requirements |
| `-R macOS` | macOS-specific requirements |
| `-R airplay2tv` | AirPlay 2-Enabled TV requirements |

### Additional Columns
| Option | Description |
|--------|-------------|
| `-c all` | Show all available columns |
| `-c aspect` | Show aspect ratio |
| `-c format` | Show container format |
| `-c hdcp` | Show HDCP requirements |
| `-c pl` | Show playlist details |

### Examples
```bash
# Basic report
hlsreport -o report.html validation.json

# Verbose with iOS rules
hlsreport -V -R iOS -o ios_report.html validation.json

# All columns
hlsreport -V -c all -o detailed_report.html validation.json
```

---

## variantplaylistcreator

Creates multivariant playlists from multiple VOD streams.

### Basic Usage
```bash
variantplaylistcreator -o <output> <url1> <plist1> [--iframe-url <iframe1>] ...
```

### Options
| Option | Description |
|--------|-------------|
| `-o <file>` | Output playlist file (required) |

### Arguments
For each variant:
1. Stream URL (as it will appear in playlist)
2. Path to .plist file (from mediafilesegmenter)
3. Optional: `--iframe-url <url>` for I-frame playlist

### Example
```bash
variantplaylistcreator -o master.m3u8 \
  http://example.com/hi/prog_index.m3u8 output/hi/prog_index.m3u8.plist \
  --iframe-url http://example.com/hi/iframe_index.m3u8 \
  http://example.com/mid/prog_index.m3u8 output/mid/prog_index.m3u8.plist \
  --iframe-url http://example.com/mid/iframe_index.m3u8 \
  http://example.com/lo/prog_index.m3u8 output/lo/prog_index.m3u8.plist \
  --iframe-url http://example.com/lo/iframe_index.m3u8
```

---

## mediasubtitlesegmenter

Segments subtitle files (tx3g, SRT, WebVTT) for HLS.

### Basic Usage
```bash
mediasubtitlesegmenter [options] <subtitle_file>
```

### Options
| Option | Description | Default |
|--------|-------------|---------|
| `-f <path>` | Output directory (required) | - |
| `-t <seconds>` | Target segment duration | 60 |
| `-D <seconds>` | Overall presentation duration | - |
| `-o <name>` | Index filename | `prog_index.m3u8` |
| `-m <offset>` | MPEG timestamp offset | 0 |
| `-b <url>` | Base URL for playlist entries | - |

### Example
```bash
mediasubtitlesegmenter -f output/subs/en -t 6 -D 3600 subtitles_en.vtt
```

---

## tsrecompressor

Generates or captures continuous media streams for testing.

### Basic Usage
```bash
tsrecompressor [options]
```

### Mode Selection
| Option | Description |
|--------|-------------|
| `-g` | Generate test pattern (BITC + LTC) |
| `-c` | Capture from camera/microphone |
| `-i <file>` | Input from file |

### Output Destinations
| Option | Description | Quality |
|--------|-------------|---------|
| `-H <addr>:<port>` | High quality output | ~7.5 Mbps |
| `-O <addr>:<port>` | Mid quality output | ~4 Mbps |
| `-L <addr>:<port>` | Low quality output | ~2 Mbps |
| `-P <addr>:<port>` | Preview output | ~0.5 Mbps |

### Encoding Options
| Option | Description |
|--------|-------------|
| `-h` | Use hardware encoder |
| `-x` | Use HEVC (default H.264) |
| `-a` | Include audio |

### Example
```bash
# Generate test pattern with multiple outputs
tsrecompressor -g -h -a \
  -H 224.0.0.50:9121 \
  -O 224.0.0.50:9123 \
  -L 224.0.0.50:9125
```

---

## id3taggenerator

Creates ID3 metadata tags for injection into live streams.

### Basic Usage
```bash
id3taggenerator [options]
```

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

### Examples
```bash
# Generate ID3 file
id3taggenerator -o metadata.id3 -t "Custom text data"

# Send to live stream
id3taggenerator -a 127.0.0.1:5555 --title "Song Title" --artist "Artist Name"

# Repeating station ID
id3taggenerator -a 127.0.0.1:5555 -r -t "WXYZ Radio"
```

---

## Quick Reference

### Common Workflows

**VOD Segmentation:**
```bash
mediafilesegmenter -f output -t 6 --format iso -z iframe_index.m3u8 input.mp4
```

**Live Streaming:**
```bash
mediastreamsegmenter -f /var/www/live -t 6 -s 10 -D -T --format iso 224.0.0.50:9123
```

**Validation:**
```bash
mediastreamvalidator -O validation.json -V http://example.com/master.m3u8
hlsreport -V -o report.html validation.json
```

**Test Stream:**
```bash
tsrecompressor -g -h -a -O 224.0.0.50:9123
```
