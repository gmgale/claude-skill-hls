<command-name>hls-validate</command-name>
<description>Validate HLS streams using mediastreamvalidator and generate reports with hlsreport.</description>

# HLS Stream Validation

Guide users through validating HLS streams against Apple's authoring specification.

## Overview

Validation ensures HLS streams:
- Conform to the HLS specification
- Meet Apple device requirements
- Will play correctly across target platforms

## Tools

- **mediastreamvalidator** - Downloads and validates streams, outputs JSON
- **hlsreport** - Generates HTML reports from validation JSON

## Basic Validation Workflow

### Step 1: Run Validator

```bash
mediastreamvalidator \
  -O validation_data.json \
  -V \
  http://example.com/master.m3u8
```

### Step 2: Generate Report

```bash
hlsreport \
  -V \
  -o report.html \
  validation_data.json
```

### Step 3: Review Issues

Open `report.html` in a browser and address issues by category:
1. **Critical Must Fix** - Will cause playback failures
2. **Must Fix** - Serious issues
3. **Should Fix** - Recommended improvements
4. **Other Issues** - Informational

## mediastreamvalidator Options

### Output Options
| Option | Description |
|--------|-------------|
| `-O <path>` | Output JSON file path |
| `-V` | Verbose console output |
| `-q` | Quiet mode |

### Validation Control
| Option | Description |
|--------|-------------|
| `-t <seconds>` | Validation timeout (default: 300) |
| `-p` | Parse playlist only (no segment download) |
| `-d <device>` | Simulate device (ipod, iphone, ipad, atv, visionpro) |

### Network Options
| Option | Description |
|--------|-------------|
| `-u <user:pass>` | HTTP authentication |
| `-c <path>` | Client certificate |
| `-r <url>` | HTTP referer |

### Other Options
| Option | Description |
|--------|-------------|
| `-s` | Skip SSL certificate validation |
| `-j` | Detailed JSON output |

## hlsreport Options

### Output Options
| Option | Description |
|--------|-------------|
| `-o <file>` | Output HTML file |
| `-V` | Verbose (detailed tables) |

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

## Device-Specific Validation

Validate for specific Apple platforms:

```bash
# iOS validation
mediastreamvalidator -d iphone -O ios_validation.json http://example.com/master.m3u8
hlsreport -R iOS -o ios_report.html ios_validation.json

# tvOS validation
mediastreamvalidator -d atv -O tvos_validation.json http://example.com/master.m3u8
hlsreport -R tvOS -o tvos_report.html tvos_validation.json

# visionOS validation
mediastreamvalidator -d visionpro -O visionos_validation.json http://example.com/master.m3u8
hlsreport -R standard -o visionos_report.html visionos_validation.json
```

## Common Validation Issues

### Critical Must Fix

#### "Average segment bitrate exceeds BANDWIDTH"
**Cause:** Declared BANDWIDTH is lower than actual peak bitrate
**Fix:** Increase BANDWIDTH in multivariant playlist or re-encode at lower bitrate

#### "Video codec not supported on target device"
**Cause:** Using HEVC on older devices or unsupported profile/level
**Fix:** Include H.264 fallback variants

#### "Segment duration exceeds target duration"
**Cause:** Keyframe interval too long
**Fix:** Re-encode with keyframes every 2 seconds

### Must Fix

#### "EXT-X-PROGRAM-DATE-TIME missing"
**Cause:** Live playlist lacks program date-time
**Fix:** Use `-T` flag with mediastreamsegmenter

#### "CODECS attribute missing"
**Cause:** Multivariant playlist missing codec declaration
**Fix:** Add CODECS="avc1.640028,mp4a.40.2" to EXT-X-STREAM-INF

#### "No I-frame playlist"
**Cause:** Missing trick play support
**Fix:** Generate I-frame playlist with `-z` flag

### Should Fix

#### "Segment duration varies significantly"
**Cause:** Inconsistent keyframe placement
**Fix:** Re-encode with fixed keyframe interval

#### "Peak bitrate exceeds 200% of average"
**Cause:** Highly variable bitrate encoding
**Fix:** Use CBR or constrained VBR encoding

## Bandwidth Validation Tolerances

### VOD Streams
| Metric | Tolerance |
|--------|-----------|
| Average vs AVERAGE-BANDWIDTH | ±10% |
| Peak vs BANDWIDTH | ±10% |
| Peak vs Average | < 200% |

### Live/Linear Streams
| Metric | Tolerance |
|--------|-----------|
| Average vs AVERAGE-BANDWIDTH | < 110% |
| Peak vs BANDWIDTH | < 125% |

## Validation Script

```bash
#!/bin/bash
# Comprehensive HLS validation

STREAM_URL="$1"
OUTPUT_DIR="${2:-validation_results}"

if [ -z "$STREAM_URL" ]; then
    echo "Usage: $0 <stream_url> [output_dir]"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

echo "Validating stream: $STREAM_URL"

# Run validator
mediastreamvalidator \
    -t 120 \
    -O "$OUTPUT_DIR/validation_data.json" \
    -V \
    "$STREAM_URL"

if [ $? -ne 0 ]; then
    echo "Validation failed!"
    exit 1
fi

# Generate reports for all rule sets
for ruleset in standard iOS tvOS macOS airplay2tv; do
    echo "Generating $ruleset report..."
    hlsreport \
        -V \
        -R "$ruleset" \
        -c all \
        -o "$OUTPUT_DIR/report_${ruleset}.html" \
        "$OUTPUT_DIR/validation_data.json"
done

echo ""
echo "Validation complete!"
echo "Reports saved to: $OUTPUT_DIR/"
echo ""
echo "Open reports in browser:"
ls "$OUTPUT_DIR"/*.html | while read f; do
    echo "  file://$f"
done
```

## Interpreting JSON Output

The `validation_data.json` contains detailed information:

```json
{
    "issues": [
        {
            "category": "Critical Must Fix",
            "message": "Description of the issue",
            "playlist": "path/to/playlist.m3u8",
            "segment": "fileSequence1.m4s"
        }
    ],
    "playlists": [
        {
            "url": "path/to/playlist.m3u8",
            "type": "VOD",
            "targetDuration": 6,
            "segments": [...]
        }
    ],
    "variants": [
        {
            "bandwidth": 4000000,
            "averageBandwidth": 3500000,
            "codecs": "avc1.640028,mp4a.40.2",
            "resolution": "1280x720"
        }
    ]
}
```

## Playlist-Only Validation

For quick syntax checks without downloading segments:

```bash
mediastreamvalidator -p http://example.com/master.m3u8
```

This validates:
- Playlist syntax
- Tag ordering
- Required attributes
- URL accessibility

But does NOT validate:
- Actual bitrates
- Codec compatibility
- Segment content

## Continuous Validation

For monitoring live streams:

```bash
#!/bin/bash
# Monitor live stream health

STREAM_URL="$1"
INTERVAL=300  # 5 minutes

while true; do
    timestamp=$(date +%Y%m%d_%H%M%S)
    echo "[$timestamp] Validating..."

    mediastreamvalidator \
        -t 60 \
        -O "validation_${timestamp}.json" \
        "$STREAM_URL"

    # Check for critical issues
    if grep -q '"Critical Must Fix"' "validation_${timestamp}.json"; then
        echo "ALERT: Critical issues found!"
        # Send notification here
    fi

    sleep $INTERVAL
done
```

## Tips

1. **Validate early and often** during development
2. **Use device simulation** (`-d`) to catch platform-specific issues
3. **Check all rule sets** for cross-platform compatibility
4. **Fix Critical issues first** - they will cause playback failures
5. **Monitor live streams** periodically to catch drift
6. **Save validation JSON** for historical comparison
7. **Use verbose mode** (`-V`) to understand issues in context

## Common CODECS Values

Include these in your multivariant playlist:

| Codec | Value |
|-------|-------|
| H.264 Baseline L3.1 | `avc1.42001f` |
| H.264 Main L4.0 | `avc1.4d0028` |
| H.264 High L4.1 | `avc1.640029` |
| HEVC Main 10 L4.1 | `hvc1.2.4.L123.B0` |
| AAC-LC | `mp4a.40.2` |
| HE-AAC | `mp4a.40.5` |
| AC-3 | `ac-3` |
| E-AC-3 | `ec-3` |
