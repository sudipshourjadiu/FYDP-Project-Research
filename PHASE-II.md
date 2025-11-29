# *Decentralized Edge-Based Video Storage + Encryption + Distributed Storage + Blockchain Audit*

# **1. System Overview (What We Are Building)**

We are designing a **secure surveillance storage system** where:

1. **Cameras** generate live video streams (RTSP).
2. **Edge devices** (small servers located on each site / vehicle) receive the streams, break them into short video files, encrypt them, and store them locally.
3. The **same edge devices** also act as **distributed storage nodes**, storing pieces of video data from other edges (Storj-style architecture).
4. A **central on-premise server** coordinates:

   * metadata,
   * node health,
   * erasure coding,
   * key management,
   * blockchain anchoring.
5. A **blockchain layer** stores a compact proof (Merkle root), ensuring the integrity of all stored video segments.

This architecture provides:

* **Offline capture** capability,
* **Encrypted storage**,
* **Redundant distributed storage**,
* **Tamper-evident blockchain verification**,
* **Scalability across multiple sites**.

---

# **2. Detailed System Components**

## **2.1 IP Cameras**

* Cameras output live video using **RTSP (Real Time Streaming Protocol)**.
* RTSP provides a persistent connection from which video frames are delivered in real-time.
* Format is typically **H.264 or H.265**, which is compatible with FFmpeg.

---

## **2.2 Edge Device (Most Critical Component)**

Each site has one **Edge Device**:

* A small PC / server / industrial box.
* Runs the **video ingestion pipeline**.
* Stores data locally (guaranteed offline operation).
* Acts as a **storage node** in the distributed storage network.

The Edge must run several internal services:

### **2.2.1 Video Ingestion Service**

Uses **FFmpeg** to convert continuous RTSP video stream into fixed-length segments.

**FFmpeg:**
Open-source command-line tool for video processing (converting, encoding, segmenting).

Example segmentation command:

```bash
ffmpeg -i rtsp://camera-ip/stream \
  -c copy \
  -f segment \
  -segment_time 6 \
  -reset_timestamps 1 \
  /var/edge/segments/CAM-101/segment_%Y%m%dT%H%M%S.mp4
```

Explanation:

| Parameter           | Meaning                     |
| ------------------- | --------------------------- |
| `-c copy`           | No re-encoding, low CPU use |
| `-f segment`        | Enable segmentation feature |
| `-segment_time 6`   | New file every 6 seconds    |
| `-reset_timestamps` | Keeps segments independent  |

Each segment file is around **6 seconds long**.

---

### **2.2.2 Encryption Service**

Every segment is encrypted **locally** before being stored or uploaded.

#### **Encryption concepts used:**

* **CEK (Content Encryption Key)**
  Temporary key used to encrypt one segment (or small group of segments).

* **KEK (Key Encryption Key)**
  Long-term key stored on the edge and on-prem, used to encrypt/wrap CEKs.

* **AES-256-GCM**
  AES encryption with:

  * **256-bit key** (very strong)
  * **GCM mode** (provides confidentiality + integrity)

#### **Process:**

1. Generate 256-bit CEK (random key).
2. Encrypt segment using AES-GCM.
3. Get:

   * IV (Initialization Vector)
   * Authentication Tag
4. Wrap CEK using KEK.

### **2.2.3 Hashing Service**

Compute **SHA-256 hash** of the encrypted segment:

* Hash = unique 256-bit fingerprint.
* ANY modification to segment changes the hash completely.

### **2.2.4 Local Metadata (SQLite Database)**

Metadata stored:

| Field                 | Description                     |
| --------------------- | ------------------------------- |
| segment_id            | Unique segment name             |
| camera_id             | Which camera produced it        |
| time_start / time_end | Timestamp range                 |
| segment_hash          | SHA-256 of encrypted bytes      |
| enc_algorithm         | AES-256-GCM                     |
| iv                    | 12-byte random IV               |
| auth_tag              | GCM integrity tag               |
| wrapped_cek           | CEK encrypted with KEK          |
| wrapped_cek_iv        | IV for CEK-wrapping             |
| wrapped_cek_tag       | Auth tag for CEK-wrapping       |
| sync_status           | PENDING / SENT                  |
| local_path            | Where encrypted video is stored |

---

## **2.3 On-Premise Core Server**

The center of the system, running:

1. **Satellite Service** (Distributed Storage Coordinator)
2. **Central Metadata Database** (PostgreSQL/MySQL)
3. **Playback API Server** (Node.js)
4. **Key Management Service** (KEK storage)
5. **Blockchain Batcher** (Merkle tree + blockchain writes)

### **2.3.1 Satellite Service (Critical Coordination Node)**

The satellite:

* Keeps track of **all storage nodes**.
* Assigns piece placement for uploaded segments.
* Handles **erasure coding** instructions.
* Performs **health checks and repair tasks**.

Satellite stores information:

| Table         | Stores                            |
| ------------- | --------------------------------- |
| nodes         | List of all edges (storage nodes) |
| objects       | One row per segment/object        |
| pieces        | Which node stores which piece     |
| health_checks | Availability + timestamps         |

### **2.3.2 Distributed Storage Nodes (Storj-style)**

Each edge device is a **storage node**:

* Stores pieces under directory:

  ```
  /storj/pieces/{object_id}/{piece_index}
  ```
* Exposes two HTTP endpoints:

  * `PUT /piece` (upload from other edges)
  * `GET /piece` (download when reconstructing)

Nodes never see keys or plaintext files.
They only receive **encrypted pieces**.

---

# **3. Detailed Write Path (Camera → Edge → Satellite → Nodes)**

---

## **Step 1 — Ingest Segment on Edge**

FFmpeg creates a 6-sec file:

Example filename:

```
segment_20251129T121006.mp4
```

---

## **Step 2 — Encrypt the Segment**

### **Generate CEK:**

```js
const CEK = crypto.randomBytes(32); // AES-256
```

### **Encrypt video:**

```js
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', CEK, iv);
```

Pipeline:

```
Raw segment → AES-GCM → Encrypted segment (.enc)
```

---

## **Step 3 — Compute SHA-256 Hash**

```js
const hash = crypto.createHash('sha256');
```

Hash (hex string) becomes segment fingerprint.

---

## **Step 4 — Wrap CEK with KEK**

```js
const kek = loadKEK(camera_id);
const wrapCipher = crypto.createCipheriv('aes-256-gcm', kek, wrap_iv);
```

Result:

* `wrapped_cek`
* `wrapped_cek_iv`
* `wrapped_cek_tag`

---

## **Step 5 — Insert into Local SQLite as PENDING**

Local DB record:

| Field        | Value                                  |
| ------------ | -------------------------------------- |
| sync_status  | PENDING                                |
| enc_path     | e.g. `/var/edge/encrypted/CAM-101/...` |
| segment_hash | SHA-256 value                          |
| wrapped_cek  | bytes                                  |
| iv           | AES-GCM IV                             |
| ...          | etc                                    |

---

## **Step 6 — Request Storage Plan from Satellite**

Request:

```json
{
  "object_key": "/tenant/t1/site/s1/camera/c101/date/2025-11-29/segment/20251129T121006.mp4.enc",
  "size": 1048576,
  "hash": "e3b0c44298f...",
  "camera_id": "CAM-101",
  "site_id": "SITE-01",
  "edge_id": "EDGE-01"
}
```

Satellite responds:

```json
{
  "object_id": "obj_abc123",
  "k": 20,
  "n": 40,
  "pieces": [
    {"piece_index": 0, "node_id": "EDGE-01", "upload_url": "https://edge01/piece"},
    {"piece_index": 1, "node_id": "EDGE-08", "upload_url": "https://edge08/piece"},
    ...
  ]
}
```

---

## **Step 7 — Edge Splits File into Pieces (Erasure Coding)**

Using **Reed-Solomon** library:

* Input: encrypted segment bytes.
* Output: 40 pieces (`n = 40`).

Any 20 (`k = 20`) pieces can recreate the file.

---

## **Step 8 — Upload Pieces to Nodes**

Each Upload:

```http
PUT /piece
X-Object-Id: obj_abc123
X-Piece-Index: 0
X-Piece-Hash: <sha256>
Content-Type: application/octet-stream
```

Node stores as:

```
/storj/pieces/obj_abc123/0
```

After ≥ 20 successful uploads:

* Satellite marks object as **AVAILABLE**.

Edge updates SQLite:

* `sync_status = 'SENT'`
* `object_id = obj_abc123`

---

# **4. Read Path (Playback Flow)**

---

## **Step 1 — User Requests Playback**

API:

```http
GET /api/v1/playback?camera=CAM-101&from=12:00&to=12:30
```

Backend fetches segments in order.

---

## **Step 2 — Backend Requests Download Plans from Satellite**

```json
{
  "object_id": "obj_abc123"
}
```

Satellite returns list of nodes that store pieces.

---

## **Step 3 — Backend Downloads Pieces**

Backend pulls pieces:

* Parallel downloads (fast).
* Verifies each piece hash.
* Stops after `k` valid pieces (e.g., 20).

---

## **Step 4 — Reconstruct Encrypted Segment**

Use Reed-Solomon decode:

```
pieces → reconstructed encrypted bytes
```

Verify:

```
SHA-256(reconstructed) == segment_hash
```

---

## **Step 5 — Decrypt Segment**

1. Decrypt CEK using KEK.
2. Decrypt segment using AES-GCM.

Output: raw video.

---

## **Step 6 — Stream to Browser via HLS**

Backend uses FFmpeg to convert decrypted video to **HLS**:

* `.m3u8` playlist
* `.ts` or `.m4s` chunks

Browser streams video normally.

---

# **5. Blockchain Integrity Layer**

This ensures nobody can secretly modify or delete video evidence.

---

## **5.1 Batching Segments**

Every 10–15 minutes:

1. Collect 500–2000 unbatched segments.
2. Extract their `segment_hash` values.

---

## **5.2 Build Merkle Tree**

* Leaves = segment hashes.
* Parents = `SHA-256(left + right)`.
* Continue until one root remains.

**Merkle root** represents entire batch.

---

## **5.3 Write Merkle Root to Blockchain**

Smart contract stores:

* `merkle_root`
* `batch_id`
* `segment_count`
* `timestamp`

One transaction secures thousands of segments.

---

## **5.4 Future Verification**

To verify a segment:

1. Recompute hash.
2. Get its Merkle path.
3. Recompute root.
4. Compare with blockchain value.

If match → segment is tamper-free.

---

# **6. Offline Operation (Critical Design Requirement)**

### If Edge Goes Offline:

* FFmpeg still segments.
* Segments still encrypted & hashed.
* Metadata stored in SQLite.
* `sync_status = PENDING`.

### When Back Online:

* Sync worker pushes pending segments.
* Distributed storage updates.
* Blockchain batching includes them.

This supports:

* **mobile surveillance vans**
* **remote sites with bad connectivity**

---

# **7. Summary Table**

| Component           | Function                                                      |
| ------------------- | ------------------------------------------------------------- |
| Camera              | Produces RTSP live stream                                     |
| Edge Device         | Segments, encrypts, hashes, stores locally, runs storage node |
| Distributed Storage | Stores encrypted pieces across multiple edges                 |
| Satellite           | Coordinates nodes, erasure coding, repairs                    |
| Backend             | Playback, metadata API                                        |
| Blockchain          | Long-term integrity proof                                     |
| SQLite              | Local edge metadata DB                                        |
| PostgreSQL          | Central metadata DB                                           |
