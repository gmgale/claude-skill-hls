<command-name>hls-encrypt</command-name>
<description>Set up content encryption for HLS streams using AES-128 or sample-AES.</description>

# HLS Content Encryption

Guide users through encrypting HLS content for content protection.

## Overview

HLS supports two encryption methods:
- **AES-128** - Full segment encryption (most common)
- **Sample-AES** - Sample-level encryption (for FairPlay, requires Apple authorization)

## Encryption Methods

### AES-128 (Recommended for standard protection)
- Encrypts entire segment with AES-128-CBC
- Key delivered via EXT-X-KEY tag
- Works with all HLS clients

### Sample-AES
- Encrypts individual samples (NAL units for video, audio frames)
- Required for Apple FairPlay Streaming
- Requires special provisioning from Apple

## Basic AES-128 Encryption

### Step 1: Generate Encryption Key

```bash
# Generate 16-byte (128-bit) key
openssl rand 16 > encryption.key

# Generate key in hex format (for reference)
xxd -p encryption.key
```

### Step 2: Segment with Encryption

```bash
mediafilesegmenter \
  -f output/encrypted \
  -t 6 \
  --format iso \
  -k encryption.key \
  -K https://secure.example.com/keys/ \
  input.mp4
```

**Key options:**
- `-k <path>` - Path to encryption key file
- `-K <url>` - Base URL where key will be served

### Step 3: Serve Key Securely

The key file must be served via HTTPS. The playlist will contain:

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="https://secure.example.com/keys/encryption.key"
```

## Key Rotation

Rotate keys periodically for enhanced security:

```bash
mediafilesegmenter \
  -f output/encrypted \
  -t 6 \
  --format iso \
  -k keys/ \
  -K https://secure.example.com/keys/ \
  --key-rotation-period 15 \
  input.mp4
```

**Options:**
- `-k <directory>` - Directory for multiple keys (auto-generated)
- `--key-rotation-period <n>` - Rotate key every n segments

This generates multiple keys and rotates them in the playlist:

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="https://secure.example.com/keys/key_001.key"
#EXTINF:6.0,
fileSequence0.m4s
...
#EXT-X-KEY:METHOD=AES-128,URI="https://secure.example.com/keys/key_002.key"
#EXTINF:6.0,
fileSequence15.m4s
```

## Key Info File (Advanced)

For more control, use a key info file:

### Create key_info.txt
```
https://secure.example.com/keys/encryption.key
/path/to/encryption.key
0123456789abcdef0123456789abcdef
```

Format:
1. Key URI (in playlist)
2. Key file path (local)
3. IV (initialization vector, optional - 32 hex chars)

### Use with mediafilesegmenter
```bash
mediafilesegmenter \
  -f output/encrypted \
  -t 6 \
  --format iso \
  -F key_info.txt \
  input.mp4
```

## Initialization Vector (IV)

By default, segment sequence number is used as IV. For explicit IV:

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="https://secure.example.com/keys/key.key",IV=0x0123456789abcdef0123456789abcdef
```

## Encrypted Playlist Example

```m3u8
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD

#EXT-X-KEY:METHOD=AES-128,URI="https://secure.example.com/keys/encryption.key"

#EXT-X-MAP:URI="fileSequence0.mp4"
#EXTINF:6.006,
fileSequence1.m4s
#EXTINF:6.006,
fileSequence2.m4s
#EXTINF:6.006,
fileSequence3.m4s

#EXT-X-ENDLIST
```

## Live Stream Encryption

For live streams with mediastreamsegmenter:

```bash
mediastreamsegmenter \
  -f /var/www/html/live \
  -t 6 \
  -s 10 \
  -D \
  --format iso \
  -k keys/ \
  -K https://secure.example.com/keys/ \
  --key-rotation-period 30 \
  224.0.0.50:9123
```

## Key Server Requirements

### Secure Key Delivery
1. **Use HTTPS** - Keys must never be transmitted in plain text
2. **Authentication** - Require user authentication before key delivery
3. **Token validation** - Use time-limited tokens in key URLs
4. **Geographic restrictions** - Enforce regional licensing

### Example Key Server (Node.js)
```javascript
const express = require('express');
const fs = require('fs');
const app = express();

app.get('/keys/:keyId', (req, res) => {
    // Validate authentication
    const token = req.headers.authorization;
    if (!validateToken(token)) {
        return res.status(401).send('Unauthorized');
    }

    // Validate token hasn't expired
    // Validate geographic region
    // Log access for auditing

    const keyPath = `/secure/keys/${req.params.keyId}`;
    if (fs.existsSync(keyPath)) {
        res.setHeader('Content-Type', 'application/octet-stream');
        res.send(fs.readFileSync(keyPath));
    } else {
        res.status(404).send('Key not found');
    }
});

app.listen(443);
```

## FairPlay Streaming (Sample-AES)

FairPlay requires:
1. Apple Developer Program membership
2. FairPlay Streaming deployment package from Apple
3. Key Security Module (KSM) implementation

### Request FairPlay Access
1. Go to [Apple Developer](https://developer.apple.com/streaming/fps/)
2. Request FairPlay Streaming deployment package
3. Implement KSM per Apple's specifications

### Sample-AES Playlist Format
```m3u8
#EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://key-server.example.com/key123",KEYFORMAT="com.apple.streamingkeydelivery",KEYFORMATVERSIONS="1"
```

**Note:** FairPlay implementation details are covered in Apple's confidential deployment documentation.

## Multi-DRM Support

For broad device support, include multiple key formats:

```m3u8
#EXT-X-KEY:METHOD=SAMPLE-AES,URI="skd://fairplay.example.com/key",KEYFORMAT="com.apple.streamingkeydelivery",KEYFORMATVERSIONS="1"
#EXT-X-KEY:METHOD=SAMPLE-AES-CTR,URI="data:text/plain;base64,...",KEYFORMAT="urn:uuid:edef8ba9-79d6-4ace-a3c8-27dcd51d21ed",KEYFORMATVERSIONS="1"
```

This allows:
- Apple devices → FairPlay
- Android/other → Widevine (requires separate packaging)

## Security Best Practices

1. **Never hardcode keys** in application code
2. **Use HTTPS** for all key delivery
3. **Implement token expiration** (short-lived tokens)
4. **Rotate keys** regularly for live content
5. **Monitor key requests** for abuse patterns
6. **Use geographic restrictions** where required by licensing
7. **Consider FairPlay** for premium content requiring stronger protection

## Validation

Validate encrypted streams:

```bash
mediastreamvalidator -V https://example.com/encrypted/master.m3u8
```

Validator will:
- Check EXT-X-KEY syntax
- Verify key URL is accessible (if public)
- Report encryption method used

## Complete Encryption Workflow

```bash
#!/bin/bash
# Encrypt VOD content with key rotation

INPUT="source_video.mp4"
OUTPUT_DIR="encrypted_output"
KEY_DIR="$OUTPUT_DIR/keys"
KEY_URL="https://secure.example.com/keys"

# Create directories
mkdir -p "$OUTPUT_DIR" "$KEY_DIR"

# Segment with encryption and key rotation
mediafilesegmenter \
    -f "$OUTPUT_DIR" \
    -t 6 \
    --format iso \
    -z iframe_index.m3u8 \
    -k "$KEY_DIR" \
    -K "$KEY_URL/" \
    --key-rotation-period 10 \
    "$INPUT"

# List generated keys
echo "Generated keys:"
ls -la "$KEY_DIR"

# Validate
mediastreamvalidator -V "$OUTPUT_DIR/prog_index.m3u8"

echo "Done! Deploy keys to $KEY_URL and content to your CDN."
```

## Troubleshooting

### "Key not found" errors
- Verify key URL is correct and accessible
- Check CORS headers on key server
- Ensure HTTPS is working

### "Decryption failed"
- Key file must be exactly 16 bytes
- IV must be 16 bytes (32 hex chars)
- Verify key matches what was used for encryption

### "Unsupported key format"
- Check client supports encryption method
- AES-128 has broadest support
- Sample-AES requires specific client support

### Keys exposed in playlist
- This is normal - security comes from:
  - HTTPS transport encryption
  - Authentication on key server
  - Token validation
