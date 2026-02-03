# HLS Bitrate Ladders Reference

Complete encoding bitrate recommendations from Apple's HLS Authoring Specification.

## H.264/AVC Bitrate Ladder (16:9 Aspect Ratio)

### Standard Video
| Resolution | Frame Rate | Bitrate (kbps) | Profile/Level |
|------------|------------|----------------|---------------|
| 416 x 234 | ≤ 30 fps | 145 | Baseline 3.0 |
| 640 x 360 | ≤ 30 fps | 365 | Main 3.1 |
| 768 x 432 | ≤ 30 fps | 730 | Main 3.1 |
| 768 x 432 | ≤ 30 fps | 1100 | Main 3.1 |
| 960 x 540 | source | 2000 | Main 3.1 |
| 1280 x 720 | source | 3000 | High 4.0 |
| 1280 x 720 | source | 4500 | High 4.0 |
| 1920 x 1080 | source | 6000 | High 4.1 |
| 1920 x 1080 | source | 7800 | High 4.1 |

### I-Frame Playlist (Trick Play)
| Resolution | Bitrate (kbps) |
|------------|----------------|
| 640 x 360 | 45 |
| 768 x 432 | 90 |
| 960 x 540 | 250 |
| 1280 x 720 | 375 |
| 1920 x 1080 | 580 |

## HEVC/H.265 Bitrate Ladder (16:9 Aspect Ratio)

### SDR Content (30 fps)
| Resolution | Bitrate (kbps) | Profile/Level |
|------------|----------------|---------------|
| 640 x 360 | 145 | Main 3.0 |
| 768 x 432 | 300 | Main 3.1 |
| 960 x 540 | 600 | Main 3.1 |
| 960 x 540 | 900 | Main 4.0 |
| 960 x 540 | 1600 | Main 4.0 |
| 1280 x 720 | 2400 | Main 4.0 |
| 1280 x 720 | 3400 | Main 4.0 |
| 1920 x 1080 | 4500 | Main 4.1 |
| 1920 x 1080 | 5800 | Main 4.1 |
| 2560 x 1440 | 8100 | Main 5.0 |
| 3840 x 2160 | 11600 | Main 5.1 |
| 3840 x 2160 | 16800 | Main 5.1 |

### HDR Content (30 fps)
| Resolution | Bitrate (kbps) | Profile/Level |
|------------|----------------|---------------|
| 640 x 360 | 160 | Main 10 3.0 |
| 768 x 432 | 360 | Main 10 3.1 |
| 960 x 540 | 730 | Main 10 3.1 |
| 960 x 540 | 1090 | Main 10 4.0 |
| 960 x 540 | 1930 | Main 10 4.0 |
| 1280 x 720 | 2900 | Main 10 4.0 |
| 1280 x 720 | 4080 | Main 10 4.0 |
| 1920 x 1080 | 5400 | Main 10 4.1 |
| 1920 x 1080 | 7000 | Main 10 4.1 |
| 2560 x 1440 | 9700 | Main 10 5.0 |
| 3840 x 2160 | 13900 | Main 10 5.1 |
| 3840 x 2160 | 20000 | Main 10 5.1 |

### HEVC I-Frame Playlist (SDR)
| Resolution | Bitrate (kbps) |
|------------|----------------|
| 768 x 432 | 40 |
| 960 x 540 | 75 |
| 960 x 540 | 200 |
| 1280 x 720 | 300 |
| 1920 x 1080 | 525 |

### HEVC I-Frame Playlist (HDR)
| Resolution | Bitrate (kbps) |
|------------|----------------|
| 768 x 432 | 55 |
| 960 x 540 | 94 |
| 960 x 540 | 238 |
| 1280 x 720 | 360 |
| 1920 x 1080 | 650 |

## Frame Rate Adjustments

| Source Frame Rate | Bitrate Adjustment |
|-------------------|-------------------|
| 24 fps | Reduce ~20% |
| 25 fps | Reduce ~17% |
| 30 fps | Base rate |
| 50 fps | Increase ~40% |
| 60 fps | Increase ~50% |

## Audio Bitrate Recommendations

### Stereo (2.0)
| Codec | Bitrate Range (kbps) | Recommended |
|-------|----------------------|-------------|
| AAC-LC | 64-160 | 128 |
| HE-AAC v1 | 32-64 | 48 |
| HE-AAC v2 | 24-48 | 32 |
| xHE-AAC | 24-160 | 48-96 |
| AC-3 | 192-384 | 192 |
| E-AC-3 | 96-160 | 128 |

### 5.1 Surround
| Codec | Bitrate (kbps) |
|-------|----------------|
| AAC-LC | 320 |
| AC-3 (Dolby Digital) | 384 |
| E-AC-3 (Dolby Digital Plus) | 192 |

### 7.1 Surround
| Codec | Bitrate (kbps) |
|-------|----------------|
| E-AC-3 (Dolby Digital Plus) | 384 |

## Default Variant Selection

| Network Type | Recommended Bitrate |
|--------------|---------------------|
| Wi-Fi | 2000 kbps |
| Cellular | 730 kbps |
| Low bandwidth | ≤ 192 kbps peak |

## Bandwidth Declaration Rules

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

## Platform Maximum Levels

### H.264/AVC
| Platform | Max Level |
|----------|-----------|
| iOS | 6.0 |
| tvOS | 5.1 |
| macOS | 5.0 |
| General compatibility | 4.1 |

### HEVC/H.265
| Platform | Max Level/Tier |
|----------|---------------|
| iOS | Main 10, 5.1 High |
| tvOS | Main 10, 5.1 High |
| macOS | Main 10, 5.1 High |
| General compatibility | Main 10, 4.0 Main |

## Key Frame Interval

- **Recommended:** Every 2 seconds
- **Maximum:** Should not exceed target segment duration
- **Formula:** GOP size = frame rate × 2

## CODECS Attribute Examples

### Video
| Description | CODECS Value |
|-------------|--------------|
| H.264 Baseline L3.0 | `avc1.42001e` |
| H.264 Baseline L3.1 | `avc1.42001f` |
| H.264 Main L3.1 | `avc1.4d001f` |
| H.264 Main L4.0 | `avc1.4d0028` |
| H.264 High L4.0 | `avc1.640028` |
| H.264 High L4.1 | `avc1.640029` |
| H.264 High L5.1 | `avc1.640033` |
| HEVC Main L4.0 | `hvc1.1.4.L120.B0` |
| HEVC Main 10 L4.1 | `hvc1.2.4.L123.B0` |
| HEVC Main 10 L5.0 | `hvc1.2.4.L150.B0` |
| HEVC Main 10 L5.1 | `hvc1.2.4.L153.B0` |
| Dolby Vision 5 L6 | `dvh1.05.06` |
| Dolby Vision 8.1 | `dvh1.08.01` |

### Audio
| Description | CODECS Value |
|-------------|--------------|
| AAC-LC | `mp4a.40.2` |
| HE-AAC v1 | `mp4a.40.5` |
| HE-AAC v2 | `mp4a.40.29` |
| xHE-AAC | `mp4a.40.42` |
| AC-3 | `ac-3` |
| E-AC-3 | `ec-3` |
| E-AC-3 + Atmos | `ec-3` |
| FLAC | `fLaC` |
| Apple Lossless | `alac` |

### Subtitles
| Description | CODECS Value |
|-------------|--------------|
| WebVTT | `wvtt` |
| IMSC1 | `stpp.ttml.im1t` |

## Example Multivariant Playlist

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS

# Audio variants
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio-aac",NAME="English",DEFAULT=YES,AUTOSELECT=YES,LANGUAGE="en",CHANNELS="2",URI="audio/en/prog_index.m3u8"

# Subtitles
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",DEFAULT=YES,AUTOSELECT=YES,LANGUAGE="en",URI="subs/en/prog_index.m3u8"

# Video variants (H.264)
#EXT-X-STREAM-INF:BANDWIDTH=145000,AVERAGE-BANDWIDTH=130000,CODECS="avc1.42001e,mp4a.40.2",RESOLUTION=416x234,FRAME-RATE=30.0,AUDIO="audio-aac",SUBTITLES="subs"
416x234/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=365000,AVERAGE-BANDWIDTH=330000,CODECS="avc1.4d001f,mp4a.40.2",RESOLUTION=640x360,FRAME-RATE=30.0,AUDIO="audio-aac",SUBTITLES="subs"
640x360/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,AVERAGE-BANDWIDTH=1800000,CODECS="avc1.4d001f,mp4a.40.2",RESOLUTION=960x540,FRAME-RATE=30.0,AUDIO="audio-aac",SUBTITLES="subs"
960x540/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=4500000,AVERAGE-BANDWIDTH=4000000,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1280x720,FRAME-RATE=30.0,AUDIO="audio-aac",SUBTITLES="subs"
1280x720/prog_index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=7800000,AVERAGE-BANDWIDTH=7000000,CODECS="avc1.640029,mp4a.40.2",RESOLUTION=1920x1080,FRAME-RATE=30.0,AUDIO="audio-aac",SUBTITLES="subs"
1920x1080/prog_index.m3u8

# I-Frame variants
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=90000,CODECS="avc1.4d001f",RESOLUTION=768x432,URI="768x432/iframe_index.m3u8"
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=375000,CODECS="avc1.640028",RESOLUTION=1280x720,URI="1280x720/iframe_index.m3u8"
#EXT-X-I-FRAME-STREAM-INF:BANDWIDTH=580000,CODECS="avc1.640029",RESOLUTION=1920x1080,URI="1920x1080/iframe_index.m3u8"
```
