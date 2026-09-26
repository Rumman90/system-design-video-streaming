# Database Design: Video Streaming Platform

This document outlines the database tables, caching keys, and storage components in simple terms.

---

## 1. Where Different Types of Data Live

| Type of Data | Storage System | Why We Use It |
| :--- | :--- | :--- |
| **Video Details & Channels** | PostgreSQL Database | Reliable, structured tables for video titles, descriptions, and user accounts. |
| **Live View Counters & Tasks** | Redis (Fast Memory) | Super fast in-memory storage for real-time counters and worker task locks. |
| **Actual Video Files & Slices** | Cloud Storage (Amazon S3) | High durability, low-cost storage for terabytes of video files. |
| **Analytics & Watch History** | ClickHouse / Columnar DB | Fast queries across billions of daily watch history logs. |

---

## 2. Core Database Tables (SQL)

```sql
-- 1. Channels / Creators
CREATE TABLE channels (
    channel_id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    handle VARCHAR(50) UNIQUE NOT NULL,
    subscriber_count BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Videos
CREATE TABLE videos (
    video_id VARCHAR(64) PRIMARY KEY,
    channel_id VARCHAR(64) REFERENCES channels(channel_id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    duration_seconds INT NOT NULL,
    status VARCHAR(30) DEFAULT 'PROCESSING', -- 'PROCESSING', 'READY', 'FAILED'
    manifest_url VARCHAR(512),
    thumbnail_url VARCHAR(512),
    views_count BIGINT DEFAULT 0,
    likes_count BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. Video Quality Versions (Renditions)
CREATE TABLE video_renditions (
    rendition_id SERIAL PRIMARY KEY,
    video_id VARCHAR(64) REFERENCES videos(video_id),
    resolution VARCHAR(10) NOT NULL, -- '1080p', '720p', '480p', '360p'
    bitrate_kbps INT NOT NULL,
    playlist_url VARCHAR(512) NOT NULL,
    total_slices INT NOT NULL
);
```

---

## 3. Fast In-Memory Keys (Redis)

Redis is used to coordinate live worker tasks and track view counts:

```text
Key: job:video:{video_id}
Type: Hash
Value: {
  "status": "PROCESSING",
  "total_slices": 150,
  "completed_slices": 95
}

Key: views:video:{video_id}
Type: Integer Counter
Value: 10452
```
