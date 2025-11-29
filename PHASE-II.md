## 1. System components

We will implement the system using three main components:

1. **IP Cameras**

   * Network cameras that stream video using protocols like **RTSP**.
   * **RTSP (Real Time Streaming Protocol)**: a network protocol used to deliver live video or audio over IP networks (e.g., `rtsp://camera-ip/...`).

2. **Edge Device / Edge Server** (one per local network or mobile unit)

   * A small server (could be an embedded device, mini-PC, or standard server) placed close to the cameras.
   * It receives video streams, cuts them into small files, encrypts them, computes hashes, stores them locally, and sends **metadata** (small descriptive data) to the on-premise server.

3. **On-Premise Server**

   * A central server located inside the organization’s main data center.
   * It collects metadata from multiple edges, stores it in a database, groups hash values into batches, computes a **Merkle root**, and writes that root into a **blockchain**.

4. **Blockchain Network**

   * A distributed ledger (for example, a private Ethereum or Hyperledger network) where we record tamper-proof proofs of integrity as small transactions.

---

## 2. Detailed data flow

### 2.1 From Camera to Edge: Video Ingestion and Segmentation

**Goal:** Take a continuous live video stream and break it into small, fixed-length files called **segments**.

* Each camera sends a video stream over **RTSP** to the edge device.
* On the edge, we run a process (e.g., a **Node.js** service) that:

  * Subscribes to each camera RTSP URL.
  * Uses a tool such as **FFmpeg** to cut the live stream into short video files.

**FFmpeg** is a widely used open-source command-line program for processing audio and video (converting, segmenting, transcoding, etc.).

**Example FFmpeg command (conceptual):**

```bash
ffmpeg -i rtsp://camera-ip/stream \
  -c copy \
  -f segment \
  -segment_time 6 \
  /var/edge/segments/CAM-101/segment_%Y%m%dT%H%M%S.mp4
```

* `-c copy`: means “don’t re-encode video, just copy the stream,” which is efficient.
* `-f segment`: use FFmpeg’s segmentation feature.
* `-segment_time 6`: create a new file every 6 seconds.
* Output path: segment files are stored in a directory on the edge device.

Each generated file is a **segment**: a short video chunk (e.g., 6 seconds long).

---

### 2.2 Hashing and Encryption on the Edge

After a segment file is created, the edge device processes it as follows.

#### 2.2.1 Generate a Random Content Encryption Key (CEK)

* For each segment, we create a fresh random key, called a **Content Encryption Key (CEK)**.
* A **key** is just a secret number used in encryption.
* The CEK should be strong, e.g., a 256-bit randomly generated value.

This can be done with a secure random generator in the programming language (e.g., Node.js `crypto.randomBytes(32)` for 256 bits).

#### 2.2.2 Encrypt the Segment

We encrypt the segment file using a symmetric cipher such as **AES-GCM**.

* **AES (Advanced Encryption Standard)**: a widely used symmetric encryption algorithm (same key for encrypt and decrypt).
* **GCM (Galois/Counter Mode)**: a mode of AES that provides both confidentiality (hides content) and integrity (detects modification).

The edge:

1. Reads the raw segment file (e.g., `segment_20251129T121006.mp4`).
2. Generates a random **nonce** or **IV (Initialization Vector)** for AES-GCM.
3. Encrypts the file with AES-GCM using the CEK.
4. Stores the resulting encrypted file to disk, e.g.:

   `/var/edge/encrypted/CAM-101/2025-11-29T12-10-06.mp4.enc`

We also keep the encryption metadata:

* Which algorithm (`"AES-256-GCM"`).
* The IV / nonce.
* The authentication tag produced by GCM (used to verify integrity during decryption).

#### 2.2.3 Compute a Cryptographic Hash

Next, we compute a **hash** of the encrypted segment.

* A **cryptographic hash function** takes any data and produces a fixed-size output (called a **hash value** or **digest**).
* Property: a tiny change in the input produces a completely different output, and it’s infeasible to find two different inputs with the same output.

We can use **SHA-256**:

* **SHA-256 (Secure Hash Algorithm 256-bit)**: a commonly used hash function producing 256-bit digests.

So:

```text
segment_hash = SHA-256(encrypted_segment_file_bytes)
```

We recommend hashing the **encrypted content**, not the original raw video, so that the hash corresponds exactly to what is stored.

---

### 2.3 Local Storage on Edge

The edge stores:

1. **Encrypted segment file** on a local filesystem.

   * Example path:
     `/var/edge/encrypted/{camera_id}/{date}/segment_id.mp4.enc`
2. **Local metadata record** for each segment in a small database (e.g., SQLite) that holds:

   * `camera_id`
   * `segment_id`
   * `timestamp`
   * `segment_hash`
   * `encryption_algorithm` (e.g., AES-256-GCM)
   * `iv` (initialization vector used)
   * `wrapped_cek` (explained in the next section)
   * `local_file_path`
   * `size_bytes`
   * `duration_sec`
   * `sync_status` (e.g., `PENDING`, `SENT`, etc.)

A **SQLite database** is a lightweight file-based relational database that is easy to embed on edge devices. It allows simple queries and local persistence without needing a big database server.

---

## 3. Key Management: CEK, KEK, and Wrapped Keys

We need a secure way to send keys from the edge to the on-prem server without exposing them.

### 3.1 Key Encryption Key (KEK)

* A **Key Encryption Key (KEK)** is a long-term key used to encrypt (or “wrap”) other keys.
* We assign a KEK per camera or per edge device.
* The KEK is known only to:

  * The edge device.
  * The on-prem server (stored safely, ideally in a secure module or at least in encrypted form).

### 3.2 Wrapping the CEK

We do **not** send the raw CEK directly to the on-prem server. Instead:

1. Edge uses the KEK to **wrap** (encrypt) the CEK using, for example, **AES Key Wrap (AES-KW)** or AES-GCM again.

2. The result is a **wrapped_cek**:

   ```text
   wrapped_cek = Encrypt(CEK, KEK)
   ```

3. Edge discards the raw CEK from memory once it has:

   * Encrypted the segment
   * Computed `wrapped_cek`

Later, if the on-prem (or an authorized user) wants to decrypt a segment, it will:

```text
CEK = Decrypt(wrapped_cek, KEK)
```

This way, the CEK is never stored or transmitted in plain form, only wrapped.

---

## 4. Sending Metadata from Edge to On-Prem

The edge will periodically send metadata records to the on-prem server over the network.

### 4.1 Transport Layer

* Edge connects to on-prem via a **VPN** (e.g., WireGuard).

  * **VPN (Virtual Private Network)**: creates an encrypted tunnel between networks for secure communication.
* On top of the VPN, we use **HTTPS** (HTTP over TLS).

  * **TLS (Transport Layer Security)**: protocol that encrypts network traffic and provides server identity verification.

So the overall path is:

> Edge → VPN tunnel → HTTPS → On-prem REST API

### 4.2 REST API on On-Prem Server

We implement a **REST API** on the on-prem server, e.g., using Node.js and a framework like Express.

* **REST API (Representational State Transfer)**: A style of web API where resources are accessed via standard HTTP methods (GET, POST, etc.) using URLs.

An example endpoint:

```http
POST /api/v1/segments/metadata
Content-Type: application/json
Authorization: Bearer <token>
```

**Request body (JSON):**

```json
{
  "camera_id": "CAM-101",
  "edge_id": "EDGE-01",
  "segment_id": "2025-11-29T12:10:06Z_0001",
  "timestamp": "2025-11-29T12:10:06Z",
  "duration_sec": 6,
  "size_bytes": 1048576,
  "hash_alg": "SHA-256",
  "segment_hash": "e3b0c44298fc1c149afbf4c8996fb924...",
  "encryption": {
    "alg": "AES-256-GCM",
    "iv": "base64-iv",
    "wrapped_cek": "base64-wrapped-key",
    "key_id": "KEK-CAM-101"
  },
  "storage": {
    "local_path": "/var/edge/encrypted/CAM-101/2025-11-29/...",
    "expected_tier": "cloud-cold"
  }
}
```

* **JSON (JavaScript Object Notation)**: a lightweight text format for structuring data, easy for both humans and machines.

---

## 5. On-Premise Server: Storage and Batching

### 5.1 Database Storage

The on-prem server stores incoming metadata in a central **relational database** such as PostgreSQL or MySQL.

Table `segment_metadata` might have columns:

* `id` (primary key)
* `camera_id`
* `edge_id`
* `segment_id`
* `timestamp`
* `hash_alg`
* `segment_hash`
* `encryption_alg`
* `iv`
* `wrapped_cek`
* `key_id`
* `size_bytes`
* `duration_sec`
* `batch_id` (foreign key to a batching table, initially null)
* `created_at`
* `updated_at`

### 5.2 Batching for Blockchain

We want to avoid writing one blockchain transaction for each segment. Instead, we group many segments into a **batch**.

A **batch** might cover:

* A fixed time window (e.g., every 15 minutes)
* Or a fixed number of segments (e.g., 1,000 segments)

For each batch, we will:

1. Take all unbatched segments (where `batch_id` is null and timestamp in the target window).
2. Put them into a list:

   ```text
   hashes = [segment_hash_1, segment_hash_2, ..., segment_hash_N]
   ```
3. Build a **Merkle tree** from these hashes.
4. Compute the **Merkle root**.
5. Create a record in a `batches` table and assign its `id` to all segments in that batch.
6. Write a single blockchain transaction that stores:

   * `merkle_root`
   * `batch_id`
   * `time_range` or `batch_start` / `batch_end`
   * additional metadata such as number of segments, etc.

---

## 6. Merkle Tree and Merkle Root (Explained Simply)

A **Merkle tree** is a binary tree built from hash values. It lets you efficiently prove that a given item (segment hash) is part of a larger set.

Construction:

1. Start with the list of leaf hashes:
   `h1, h2, h3, h4, ...`
2. Pair them: `(h1, h2)`, `(h3, h4)`, ...
3. For each pair, compute a parent hash:

   ```text
   parent = SHA-256(h_left || h_right)
   ```

   * `||` means concatenation.
4. Repeat pairing and hashing until only one hash remains. That final hash is the **Merkle root**.

Why this helps:

* To prove that a segment is in the batch, you only need a small set of hashes (the **Merkle path**) rather than the whole list.
* The Merkle root represents the entire batch’s hashes in a single 256-bit value, which is ideal to store on a blockchain.

---

## 7. Writing to the Blockchain

The on-prem server includes a **blockchain client module** (e.g., a Node.js script using web3.js or similar) that:

1. Connects to the blockchain network (e.g., a private Ethereum network node).
2. Calls a smart contract or a simple logging function to store:

   * `merkle_root` (32 bytes)
   * `batch_id` or some batch identifier
   * `timestamp`

A **blockchain transaction**:

* Is a signed message sent to the blockchain.
* It includes data (like the Merkle root) and a small fee.
* Once mined and included in a block, the data becomes tamper-evident (any modification is detectable).

This step is what makes the system **auditable**: anyone with access to the blockchain can later verify that the metadata (and therefore the videos) have not been changed.

---

## 8. Handling Offline Mode and Retries on the Edge

We need to support cases where the edge temporarily cannot reach the on-prem server (e.g., network down, mobile unit out of coverage).

On the edge:

1. Each new segment and its metadata are stored in the local SQLite database and local filesystem immediately.
2. A separate “sync worker” runs periodically:

   * Fetches all metadata entries where `sync_status = 'PENDING'`.
   * Attempts to send them to the on-prem REST API.
   * If the call succeeds:

     * Mark `sync_status = 'SENT'` (or similar).
   * If it fails:

     * Keep `sync_status = 'PENDING'` and maybe record the `last_error` and `retry_count`.

This design ensures:

* Video and metadata are never lost when offline.
* As soon as the connection is restored, the worker gradually uploads all delayed metadata.

For mobile surveillance units, the same mechanism works even if they are only online intermittently.

---

## 9. Verification Flow (How Someone Proves Integrity)

Later, an auditor or system component might want to prove:

> “This specific video segment has not been tampered with since it was created.”

Steps:

1. Download the encrypted segment file.
2. Re-compute the hash using SHA-256 on the file bytes.
3. Fetch the corresponding segment metadata from the on-prem database to get the `segment_hash`.
4. Check that `recomputed_hash == segment_hash`.
5. Obtain the batch information for that segment (`batch_id`), including:

   * The Merkle root stored on blockchain.
   * The Merkle path for this leaf (sequence of sibling hashes up the tree).
6. Using the Merkle path, recompute the Merkle root.
7. Compare:

   * `recomputed_merkle_root` vs. `merkle_root_from_blockchain`.
8. If they match:

   * It proves that:

     * The segment hash has not changed.
     * The batch has not been changed.
     * The blockchain record confirms a historical commitment to that batch at the time of the transaction.

---

## 10. Summary of Technologies and Terms Used

* **RTSP**: protocol for streaming video from cameras.
* **FFmpeg**: tool for splitting, converting, and processing video streams.
* **Segment file**: short video chunk (e.g., 6 seconds).
* **CEK (Content Encryption Key)**: one-time key used to encrypt a segment.
* **KEK (Key Encryption Key)**: longer-term key used to encrypt CEKs.
* **AES-256-GCM**: secure symmetric encryption algorithm with integrity protection.
* **SHA-256**: cryptographic hash function for integrity checks.
* **Hash value / Digest**: fixed-size output of a hash function representing data.
* **Wrapped key**: a CEK encrypted with a KEK.
* **SQLite**: small on-device database for local storage on the edge.
* **REST API**: standard web API style using HTTP and JSON.
* **HTTPS + VPN (WireGuard)**: secure communication between edge and on-prem.
* **Merkle tree / Merkle root**: data structure for summarizing many hashes in a single root hash, used for proofs of inclusion.
* **Blockchain transaction**: signed operation recorded on a distributed ledger, used to anchor Merkle roots in a tamper-evident way.
