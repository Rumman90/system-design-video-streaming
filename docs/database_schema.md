# Database Schema & Storage Architecture

This document defines the storage layers, data models, relational schemas, and caching strategies.

---

## 1. Storage Technology Choices

| Data Domain | Storage Engine | Justification |
| :--- | :--- | :--- |
| **Video Metadata & User Data** | PostgreSQL / Amazon Aurora | ACID transactions for channel management, video metadata, visibility permissions. |
| **Video Playback Logs & Telemetry** | ClickHouse / Amazon Timestream | High-throughput columnar store for petabyte-scale analytics and bitrate debugging. |
| **Realtime Counters & Leases** | Redis Cluster | Ultra-fast atomic increments (`INCRBY`), HyperLogLog for unique views, and distributed locks. |
| **Video Files, Chunks, Playlists** | AWS S3 / Google Cloud Storage | Unlimited durability (99.999999999%), cheap bulk storage with multi-tier lifecycle rules. |

---

## 2. Relational Schema (PostgreSQL)

```sql
-- 1. Users / Channels Table
CREATE TABLE channels (
    channel_id VARCHAR(64) PRIMARY KEY,
    owner_id VARCHAR(64) NOT NULL,
    handle VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    subscriber_count BIGINT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 2. Videos Table
CREATE TABLE videos (
    video_id VARCHAR(64) PRIMARY KEY,
    channel_id VARCHAR(64) NOT NULL REFERENCES channels(channel_id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    duration_seconds INT NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'UPLOADING', -- 'UPLOADING', 'PROCESSING', 'READY', 'FAILED'
    visibility VARCHAR(20) NOT NULL DEFAULT 'PUBLIC', -- 'PUBLIC', 'UNLISTED', 'PRIVATE'
    raw_s3_uri VARCHAR(512),
    manifest_s3_uri VARCHAR(512),
    thumbnail_url VARCHAR(512),
    views_count BIGINT DEFAULT 0,
    likes_count BIGINT DEFAULT 0,
    dislikes_count BIGINT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    published_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_videos_channel_id ON videos(channel_id);
CREATE INDEX idx_videos_status ON videos(status);
CREATE INDEX idx_videos_published_at ON videos(published_at DESC);

-- 3. Video Transcoding Profiles / Asset Tracks
CREATE TABLE video_renditions (
    rendition_id BIGSERIAL PRIMARY KEY,
    video_id VARCHAR(64) NOT NULL REFERENCES videos(video_id) ON DELETE CASCADE,
    resolution VARCHAR(10) NOT NULL, -- '1080p', '720p', '480p', etc.
    bitrate_kbps INT NOT NULL,
    codec VARCHAR(20) NOT NULL, -- 'h264', 'vp9', 'av1'
    playlist_path VARCHAR(512) NOT NULL,
    total_segments INT NOT NULL,
    total_size_bytes BIGINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (video_id, resolution, codec)
);
```

---

## 3. Distributed Transcoding State Model (Redis / Task Store)

Redis hashes manage active transcoding tasks and worker leases:

```text
Key: job:video:{video_id}
Type: Hash
Fields:
  status: "IN_PROGRESS"
  total_chunks: 120
  completed_chunks: 84
  created_at: 1774590000

Key: lock:chunk:{video_id}:{chunk_id}:{resolution}
Type: String (Value: worker_id, TTL: 60s)
```

---

## 4. Analytical Schema: ClickHouse (Playback Events)

```sql
CREATE TABLE video_playback_events (
    event_timestamp DateTime,
    video_id String,
    viewer_id String,
    session_id String,
    country_code LowCardinality(String),
    device_type LowCardinality(String),
    resolution LowCardinality(String),
    buffer_health_sec Float32,
    dropped_frames UInt16,
    bandwidth_kbps UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_timestamp)
ORDER BY (video_id, event_timestamp);
```
