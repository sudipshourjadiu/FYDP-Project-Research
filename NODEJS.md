Here’s a full example in **Node.js** using `spawn` from the `child_process` module to:

1. Ingest an RTSP stream via FFmpeg
2. Segment the stream into MP4 files
3. Encrypt each segment
4. (Optional) Decrypt a segment

This script assumes you're using FFmpeg compiled with encryption support.

---

### ✅ **Requirements**

Before running:

```bash
npm install uuid
```

---

### ✅ **Node.js Code Example**

```js
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');
const uuid = require('uuid');

const streamURL = 'rtsp://your-camera-ip/stream'; // Replace with real RTSP URL
const streamFFmpegInputPrefix = '-rtsp_transport tcp';
const streamFFmpegParameter = '-f mpegts';
const RECORDING_PATH = './recordings';
const FILE_DURATION = '10'; // in seconds
const ENCRYPTION_KEY = '00112233445566778899aabbccddeeff'; // 32-char hex
const ENCRYPTION_KEY_ID = '0102030405060708090a0b0c0d0e0f10'; // 32-char hex

// Ensure output directory exists
fs.mkdirSync(RECORDING_PATH, { recursive: true });

// Step 1: Ingest RTSP and pipe to segmenter
const rtspArgs = [
  ...streamFFmpegInputPrefix.split(' '),
  '-i', streamURL,
  ...streamFFmpegParameter.split(' ')
];

// Step 2: Segment to MP4
const segmentArgs = [
  '-y',
  '-hide_banner',
  '-i', 'pipe:0',
  '-codec', 'copy',
  '-f', 'segment',
  '-segment_time', FILE_DURATION,
  '-segment_format', 'mp4',
  '-movflags', 'frag_keyframe+empty_moov',
  '-strftime', '1',
  path.join(RECORDING_PATH, 'DID001_%Y-%m-%dT%H:%M:%S.mp4')
];

// Launch FFmpeg for RTSP ingestion
const ingest = spawn('ffmpeg', rtspArgs, { stdio: ['ignore', 'pipe', 'inherit'] });
const segmenter = spawn('ffmpeg', segmentArgs, { stdio: ['pipe', 'inherit', 'inherit'] });

// Pipe RTSP output to segmenter input
ingest.stdout.pipe(segmenter.stdin);

// Log events
ingest.on('exit', code => console.log(`[RTSP] exited with code ${code}`));
segmenter.on('exit', code => console.log(`[Segmenter] exited with code ${code}`));

// --- After some delay, encrypt segmented files (e.g., via interval or watch) ---
setInterval(() => {
  fs.readdir(RECORDING_PATH, (err, files) => {
    if (err) return console.error('Read dir error:', err);
    files.filter(f => f.endsWith('.mp4') && !f.includes('.enc')).forEach(file => {
      const inputPath = path.join(RECORDING_PATH, file);
      const encryptedName = `${uuid.v4()}.enc.mp4`;
      const encryptedPath = path.join(RECORDING_PATH, encryptedName);

      const args = [
        '-i', inputPath,
        '-codec', 'copy',
        '-encryption_scheme', 'cenc-aes-ctr',
        '-encryption_key', ENCRYPTION_KEY,
        '-encryption_kid', ENCRYPTION_KEY_ID,
        encryptedPath
      ];

      const encrypt = spawn('ffmpeg', args, { stdio: 'inherit' });

      encrypt.on('exit', code => {
        if (code === 0) {
          console.log(`[Encrypt] Success: ${file} -> ${encryptedName}`);
          fs.unlinkSync(inputPath); // Optional: remove original
        } else {
          console.error(`[Encrypt] Failed for ${file}`);
        }
      });
    });
  });
}, 15000); // Check every 15 sec

// --- Decryption Example ---
function decryptFile(encryptedFilePath, outputPath) {
  const args = [
    '-decryption_key', ENCRYPTION_KEY,
    '-i', encryptedFilePath,
    '-codec', 'copy',
    outputPath
  ];

  const decrypt = spawn('ffmpeg', args, { stdio: 'inherit' });

  decrypt.on('exit', code => {
    if (code === 0) console.log(`[Decrypt] Success: ${outputPath}`);
    else console.error(`[Decrypt] Failed: ${encryptedFilePath}`);
  });
}

// Example: decrypt after 1 min
setTimeout(() => {
  const encFile = fs.readdirSync(RECORDING_PATH).find(f => f.endsWith('.enc.mp4'));
  if (encFile) {
    const encPath = path.join(RECORDING_PATH, encFile);
    const decPath = path.join(RECORDING_PATH, `DECRYPTED_${Date.now()}.mp4`);
    decryptFile(encPath, decPath);
  }
}, 60000);
```

---

### 🔐 **Important Notes**:

* `ENCRYPTION_KEY` and `ENCRYPTION_KEY_ID` must be 16-byte (32-character hex) strings.
* Make sure your FFmpeg build supports encryption (`ffmpeg -hide_banner -encoders | grep cenc`).
* This code continuously ingests and encrypts segments; decryption is optional and just shown once.

---

Want this to run as a service, handle upload to cloud, or log segment metadata? Let me know!
