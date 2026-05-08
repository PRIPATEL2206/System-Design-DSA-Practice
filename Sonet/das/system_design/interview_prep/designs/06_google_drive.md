# Design: Google Drive / Dropbox (File Storage)

## Problem Statement
Design a cloud file storage service where users can upload, store, sync, and share files across devices.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Any file type and size? Max file size?" → Any type, up to 10 GB per file
- "File versioning (undo changes)?" → Yes, last 30 days
- "Real-time collaboration (like Google Docs)?" → No, just file sync
- "Sharing: view-only and edit access?" → Yes
- "Desktop sync client needed?" → Yes (mention delta sync)
- "Scale?" → 50M DAU, 10 PB total storage

### Functional Requirements
- Upload, download, delete files and folders
- Sync files across devices (desktop and mobile)
- Share files/folders with other users (view/edit)
- File versioning (restore previous versions, 30-day history)

### Non-Functional Requirements
- High durability: files must never be lost (99.999999999% = 11 nines, like S3)
- Eventual consistency for sync is acceptable
- Upload/download bandwidth optimisation (delta sync)
- Low metadata query latency (< 100ms for directory listing)

---

## Step 2: Capacity Estimation

```
DAU = 50 million
Files uploaded/day = 100M (avg 2 per user)
Avg file size = 500 KB (mix of small docs + larger media)

Storage:
  Daily new data = 100M × 500 KB = 50 TB/day
  5-year total = 50 TB × 365 × 5 ≈ 90 PB

Upload bandwidth:
  100M × 500 KB / 86,400 ≈ 579 GB/sec = 4.6 Tbps

Metadata:
  File record ≈ 500 bytes
  1 billion files → 500 GB metadata (fits in MySQL)

Deduplication benefit:
  If 30% of files are duplicates → 30% storage savings
  (Hash-based deduplication)
```

---

## Step 3: High-Level Architecture

```
[Client App (Desktop/Mobile)]
        |
        | 1. Upload file chunks
        ↓
[Upload Service / API Gateway]
        |
        | 2. Store raw chunks
        ↓
[Block Storage (S3)]          ← Actual file data
        |
        | 3. Event: file uploaded
        ↓
[Kafka]
        |
        | 4. Process event
        ↓
[Metadata Service] ──────────> [Metadata DB (MySQL)]
                                (file tree, versions, shares)
        |
        ↓
[Sync Notification Service]
        |
        ↓
[WebSocket / Long Poll]        ← Notify other devices to sync
        |
        ↓
[Client App (other devices)] → downloads updated file
```

---

## Step 4: File Upload — Block-Level Chunking

### Why Chunk Files?
```
Large file (2 GB) uploaded as one piece:
  - Resume not possible on failure
  - Must re-upload entire file even for 1 byte change

Solution: Split files into 4 MB blocks.

Block deduplication:
  - Hash each block (SHA-256)
  - Check if block already exists in block store
  - If yes: skip upload (just update metadata reference)
  - If no: upload the new block
  
Result:
  - Only changed blocks are uploaded on edit ("delta sync")
  - Identical blocks across files stored once ("deduplication")
  - Bandwidth savings: 70-90% for typical sync workload
```

### Upload Flow
```
1. Client splits file into 4 MB blocks
2. For each block:
   a. Compute SHA-256 hash
   b. POST /check { hash } → server says "exists" or "upload needed"
   c. If needed: PUT /blocks/{hash} → upload block to S3
3. POST /files { name, path, version, block_hashes[] }
   → Server assembles file metadata from block list
```

---

## Step 5: Delta Sync (Only Upload Changes)

```
Original file: [Block A] [Block B] [Block C] [Block D]
              (hashes:   h1        h2        h3        h4)

User edits middle of file:
Modified file: [Block A] [Block B'] [Block C] [Block D]
              (hashes:   h1         h2_new    h3        h4)

Client diff:
  h1 exists on server → skip
  h2_new does not exist → upload only this block (4 MB)
  h3 exists → skip
  h4 exists → skip

Upload cost = 4 MB instead of entire file!
```

---

## Step 6: File Versioning

```
Each file save creates a new version (new set of block_hashes).
Old versions kept for 30 days.

versions table:
  file_id | version_id | block_hashes[] | created_at | size | created_by

Restore: 
  User selects version → server rebuilds file from that version's blocks
  → Blocks still in S3 (garbage collector doesn't delete blocks with active version refs)

Garbage Collection:
  Background job runs daily:
  - Find blocks not referenced by any version in last 30 days
  - Delete from S3 to free storage
```

---

## Step 7: Cross-Device Sync

```
Sync Challenge: User edits file on laptop → phone should see update

Solution:
1. Laptop uploads changed blocks → updates metadata DB
2. Metadata Service publishes "file_updated" event to Kafka
3. Sync Notification Service consumes event
4. Finds all devices of this user via WebSocket registry
5. Pushes notification: "file /docs/report.pdf changed"
6. Device downloads changed blocks from S3 via CDN

WebSocket for real-time sync:
  Each device maintains a WebSocket connection (or long poll)
  On connect: register device_id + user_id in Redis
  On file change: lookup user's connected devices → push notification
```

---

## Step 8: File Sharing

```
Share modes:
  - View-only (read permission)
  - Edit (read + write permission)
  - Share link (public, no auth required)

Permissions table:
  resource_id | resource_type (file/folder) | user_id | permission (view/edit) | expires_at

Folder inheritance:
  Sharing a folder → all children inherit permission
  Implemented via recursive permission check or stored inherited flag

Share links:
  Generate unique token → store { token → file_id, permission, expires_at } in DB
  GET /share/{token} → verify token → serve file
```

---

## Step 9: Data Model

### files table
```sql
CREATE TABLE files (
  id            BIGINT PRIMARY KEY,
  user_id       BIGINT NOT NULL,
  parent_folder BIGINT,                  -- NULL = root
  name          VARCHAR(255),
  type          ENUM('file','folder'),
  current_version BIGINT,               -- FK to versions
  is_deleted    BOOLEAN DEFAULT FALSE,   -- soft delete
  created_at    DATETIME,
  updated_at    DATETIME
);
-- Index: (user_id, parent_folder) for directory listing
```

### versions table
```sql
CREATE TABLE versions (
  id          BIGINT PRIMARY KEY,
  file_id     BIGINT NOT NULL,
  size_bytes  BIGINT,
  block_hashes JSON,                    -- ["sha256_1", "sha256_2", ...]
  created_at  DATETIME,
  created_by  BIGINT
);
```

### blocks table (block registry)
```sql
CREATE TABLE blocks (
  hash        CHAR(64) PRIMARY KEY,     -- SHA-256 hex
  s3_key      VARCHAR(500),             -- path in S3
  size_bytes  INT,
  created_at  DATETIME
);
```

---

## Step 10: API Design

```
POST /api/v1/blocks/check
Body: { "hashes": ["abc", "def"] }
Response: { "upload_needed": ["def"] }   ← only upload what's missing

PUT  /api/v1/blocks/{hash}
Body: binary block data
Response: 201 Created

POST /api/v1/files
Body: { "name": "report.pdf", "path": "/docs/", "block_hashes": [...] }
Response: { "file_id": "123", "version_id": "456" }

GET  /api/v1/files/{id}
GET  /api/v1/files/{id}/versions
POST /api/v1/files/{id}/restore?version_id=456
GET  /api/v1/folders/{id}/contents
POST /api/v1/files/{id}/share
```

---

## Interview Tips

- Lead with: "The key insight is block-level chunking with deduplication — dramatically reduces bandwidth and storage."
- Always mention: "Files are never stored on app servers — client uploads directly to S3 via pre-signed URLs."
- Delta sync is the differentiator: "On every edit, only changed 4MB blocks are synced, not the whole file."
- Versioning: "Each version is just a pointer to a list of block hashes — cheap to store."

## Common Follow-up Questions
- "How to handle concurrent edits?" → Last-write-wins with conflict detection; show "conflicted copy" to user
- "How to handle very large files (10 GB)?" → Multipart upload to S3; blocks uploaded in parallel
- "How does search work?" → Elasticsearch index on file names + metadata; content search via Tika/Lucene
- "How to encrypt files?" → Client-side encryption before upload; key management service; server never sees plaintext
