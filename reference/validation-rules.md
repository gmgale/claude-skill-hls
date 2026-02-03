# HLS Validation Rules Reference

Complete reference for hlsreport validation rule sets and common issues.

## Issue Categories

### Priority Order
1. **Critical Must Fix** - Will cause playback failures
2. **Must Fix** - Serious issues, may not prevent playback
3. **Should Fix** - Permissible but recommended to fix
4. **Other Issues** - Informative, not actionable

## Rule Sets

### Standard (`-R standard`)
General HLS Authoring Requirements applicable to all platforms.

**Key Rules:**
- CODECS attribute required
- RESOLUTION attribute required for video
- BANDWIDTH must be accurate
- Segment duration must not exceed target + 0.5s
- EXT-X-PROGRAM-DATE-TIME required for live
- EXT-X-PLAYLIST-TYPE:VOD required for VOD

### iOS (`-R iOS`)
iOS-specific requirements.

**Additional Rules:**
- H.264 Level ≤ 6.0
- HEVC Level ≤ 5.1 High Tier
- Dolby Vision Profile 5 or 10, Level ≤ 9
- Cellular: variant ≤ 192 kbps peak required
- xHE-AAC supported (A12+ devices)

### tvOS (`-R tvOS`)
tvOS-specific requirements.

**Additional Rules:**
- H.264 Level ≤ 5.1
- HDR content must provide 30 fps variant
- Live playlist ≥ 120 minutes content
- No audio-only variants
- AC-3/E-AC-3 supported

### macOS (`-R macOS`)
macOS-specific requirements.

**Additional Rules:**
- H.264 Level ≤ 5.0
- Full codec support (including FLAC, Apple Lossless)
- Hardware decode support varies by Mac model

### AirPlay 2-Enabled TVs (`-R airplay2tv`)
Requirements for AirPlay 2 streaming.

**Additional Rules:**
- All video variants must have same segment boundaries
- Don't switch codecs at discontinuities
- Subtitles must be WebVTT only
- Consistent GOP structure across variants

## Common Issues and Fixes

### Critical Must Fix

#### "Average segment bitrate exceeds BANDWIDTH"
**Cause:** Declared BANDWIDTH is lower than actual peak bitrate.
**Fix:**
```m3u8
# Before (incorrect)
#EXT-X-STREAM-INF:BANDWIDTH=2000000,...

# After (correct - actual peak is 4.2 Mbps)
#EXT-X-STREAM-INF:BANDWIDTH=4200000,...
```

#### "Video codec not supported on target device"
**Cause:** Using HEVC on devices without HEVC support, or unsupported profile/level.
**Fix:** Include H.264 variants for compatibility:
```m3u8
# HEVC for modern devices
#EXT-X-STREAM-INF:BANDWIDTH=4000000,CODECS="hvc1.2.4.L123.B0,mp4a.40.2",...

# H.264 fallback for older devices
#EXT-X-STREAM-INF:BANDWIDTH=4500000,CODECS="avc1.640028,mp4a.40.2",...
```

#### "Segment duration exceeds target duration by more than 0.5 seconds"
**Cause:** Keyframe interval too long or irregular.
**Fix:** Re-encode with consistent 2-second keyframe interval:
```bash
ffmpeg -i input.mp4 -g 48 -keyint_min 48 -sc_threshold 0 output.mp4
```

#### "EXT-X-MAP missing for fMP4"
**Cause:** Initialization segment not declared.
**Fix:** Ensure playlist includes:
```m3u8
#EXT-X-MAP:URI="init.mp4"
```

### Must Fix

#### "EXT-X-PROGRAM-DATE-TIME missing in live playlist"
**Cause:** Live playlist lacks program date-time tag.
**Fix:** Use `-T` flag with mediastreamsegmenter, or add manually:
```m3u8
#EXT-X-PROGRAM-DATE-TIME:2024-01-15T18:30:00.000Z
```

#### "CODECS attribute missing"
**Cause:** Multivariant playlist lacks codec declaration.
**Fix:** Add CODECS to EXT-X-STREAM-INF:
```m3u8
#EXT-X-STREAM-INF:BANDWIDTH=4000000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720
```

#### "RESOLUTION attribute missing"
**Cause:** Video variant lacks resolution declaration.
**Fix:** Add RESOLUTION:
```m3u8
#EXT-X-STREAM-INF:BANDWIDTH=4000000,CODECS="avc1.640028",RESOLUTION=1280x720
```

#### "No I-frame playlist provided"
**Cause:** Missing trick play support.
**Fix:** Generate I-frame playlist:
```bash
mediafilesegmenter -f output -z iframe_index.m3u8 input.mp4
```
Add to multivariant playlist:
```m3u8
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=375000,CODECS="avc1.640028",RESOLUTION=1280x720,URI="iframe_index.m3u8"
```

#### "AVERAGE-BANDWIDTH missing"
**Cause:** Multivariant playlist lacks average bandwidth.
**Fix:** Add AVERAGE-BANDWIDTH:
```m3u8
#EXT-X-STREAM-INF:BANDWIDTH=4000000,AVERAGE-BANDWIDTH=3500000,...
```

#### "FRAME-RATE missing"
**Cause:** Video variant lacks frame rate declaration.
**Fix:** Add FRAME-RATE:
```m3u8
#EXT-X-STREAM-INF:BANDWIDTH=4000000,FRAME-RATE=30.0,...
```

### Should Fix

#### "Segment duration varies significantly"
**Cause:** Inconsistent keyframe placement or scene-change detection.
**Fix:** Disable scene-change keyframe insertion:
```bash
ffmpeg -i input.mp4 -g 48 -sc_threshold 0 output.mp4
```

#### "Peak bitrate exceeds 200% of average"
**Cause:** Highly variable bitrate encoding.
**Fix:** Use CBR or constrained VBR:
```bash
ffmpeg -i input.mp4 -b:v 3000k -maxrate 6000k -bufsize 6000k output.mp4
```

#### "Only one variant provided"
**Cause:** Single bitrate stream without alternatives.
**Fix:** Provide multiple bitrate variants per Apple's ladder.

#### "Target duration should be 6 seconds or less"
**Cause:** Long segments increase startup latency.
**Fix:** Use `-t 6` with segmenters.

#### "EXT-X-VERSION should be at least 4"
**Cause:** Using old protocol version.
**Fix:** mediafilesegmenter automatically uses appropriate version for features used.

### Other Issues (Informational)

#### "Segment duration is less than 2 seconds"
**Info:** Very short segments increase request overhead.
**Note:** May be intentional for low-latency streaming.

#### "Audio bitrate exceeds 160 kbps for HE-AAC"
**Info:** HE-AAC is optimized for lower bitrates.
**Note:** Use AAC-LC instead for bitrates > 64 kbps.

## Bandwidth Validation

### VOD Tolerances
| Metric | Requirement |
|--------|-------------|
| Average vs AVERAGE-BANDWIDTH | Within ±10% |
| Peak vs BANDWIDTH | Within ±10% |
| Peak vs Average | < 200% |

### Live Tolerances
| Metric | Requirement |
|--------|-------------|
| Average vs AVERAGE-BANDWIDTH | < 110% |
| Peak vs BANDWIDTH | < 125% |

## Codec Compatibility Matrix

### Video Codecs
| Codec | iOS | tvOS | macOS | AirPlay 2 TV |
|-------|-----|------|-------|--------------|
| H.264 | ✓ | ✓ | ✓ | ✓ |
| HEVC | ✓ (A9+) | ✓ | ✓ (2017+) | Varies |
| Dolby Vision | ✓ (A12+) | ✓ | ✓ | Varies |
| AV1 | ✓ (A17+) | ✓ (4K) | ✓ (M3+) | - |

### Audio Codecs
| Codec | iOS | tvOS | macOS | AirPlay 2 TV |
|-------|-----|------|-------|--------------|
| AAC-LC | ✓ | ✓ | ✓ | ✓ |
| HE-AAC | ✓ | ✓ | ✓ | ✓ |
| xHE-AAC | ✓ (A12+) | ✓ | ✓ | - |
| AC-3 | ✓ (A7+) | ✓ | ✓ | ✓ |
| E-AC-3 | ✓ (A7+) | ✓ | ✓ | ✓ |
| Dolby Atmos | ✓ (A12+) | ✓ | ✓ | Varies |

## Validation Best Practices

### Before Release Checklist

1. **Run full validation:**
```bash
mediastreamvalidator -t 600 -O validation.json http://example.com/master.m3u8
```

2. **Generate reports for all platforms:**
```bash
for ruleset in standard iOS tvOS macOS airplay2tv; do
    hlsreport -V -R $ruleset -o report_${ruleset}.html validation.json
done
```

3. **Priority order:**
   - Fix all **Critical Must Fix** issues
   - Fix all **Must Fix** issues
   - Review and address **Should Fix** issues
   - Document any intentional deviations

4. **Device testing:**
```bash
# Test on each target device type
mediastreamvalidator -d iphone -O ios.json http://example.com/master.m3u8
mediastreamvalidator -d ipad -O ipad.json http://example.com/master.m3u8
mediastreamvalidator -d atv -O tvos.json http://example.com/master.m3u8
```

5. **Live stream monitoring:**
   - Validate periodically (every 5-15 minutes)
   - Alert on new Critical/Must Fix issues
   - Track issue trends over time

### Quick Validation (Playlist Only)

For rapid syntax checking without downloading segments:
```bash
mediastreamvalidator -p http://example.com/master.m3u8
```

This validates:
- Playlist syntax
- Tag ordering
- Required attributes
- URL accessibility

But does **not** validate:
- Actual bitrates
- Segment content
- Codec compatibility

## Interpreting hlsreport Output

### Summary Section
- Total issues by category
- Pass/fail status for each rule set
- Overall compliance score

### Variant Details
- Per-variant bandwidth measurements
- Codec and resolution information
- Segment statistics

### Issue Details
- Specific issue location (playlist, segment)
- Remediation suggestions
- Related specification references

### Exporting Results
```bash
# JSON for programmatic processing
mediastreamvalidator -j -O detailed.json http://example.com/master.m3u8

# HTML for human review
hlsreport -V -c all -o detailed_report.html validation.json
```
