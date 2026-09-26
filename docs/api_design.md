# API Design: Video Streaming Platform

This document describes the simple HTTP endpoints and WebSocket messages used to upload videos, fetch video details, and stream content.

---

## 1. Video Upload Endpoints

### 1.1 Start Upload Session
Creates a new upload session and returns temporary upload links for each piece (chunk) of the file.

* **Method:** `POST`
* **Path:** `/api/v1/videos/upload-session`

#### Request Body
```json
{
  "title": "Introduction to System Design",
  "description": "A beginner guide to building scalable systems.",
  "filename": "my_video.mp4",
  "file_size_bytes": 104857600,
  "chunk_size_bytes": 10485760
}
```

#### Response (`201 Created`)
```json
{
  "video_id": "vid_12345",
  "upload_id": "upload_abc987",
  "chunk_size_bytes": 10485760,
  "total_parts": 10,
  "upload_urls": [
    {
      "part_number": 1,
      "url": "https://storage.streamhub.com/raw/vid_12345/part1?token=xyz..."
    },
    {
      "part_number": 2,
      "url": "https://storage.streamhub.com/raw/vid_12345/part2?token=xyz..."
    }
  ]
}
```

---

### 1.2 Finish Upload Session
Tells the server that all pieces have been uploaded so it can start processing the video.

* **Method:** `POST`
* **Path:** `/api/v1/videos/{video_id}/complete-upload`

#### Request Body
```json
{
  "upload_id": "upload_abc987",
  "uploaded_parts": [
    {"part_number": 1, "etag": "hash1"},
    {"part_number": 2, "etag": "hash2"},
    {"part_number": 10, "etag": "hash10"}
  ]
}
```

#### Response (`202 Accepted`)
```json
{
  "video_id": "vid_12345",
  "status": "PROCESSING",
  "message": "Upload complete. Video is now being converted."
}
```

---

## 2. Video Playback & Details Endpoints

### 2.1 Get Video Details & Streaming Link
Fetches video information and the link to the playlist file (`master.m3u8`).

* **Method:** `GET`
* **Path:** `/api/v1/videos/{video_id}`

#### Response (`200 OK`)
```json
{
  "video_id": "vid_12345",
  "title": "Introduction to System Design",
  "duration_seconds": 600,
  "thumbnail_url": "https://cdn.streamhub.com/videos/vid_12345/thumbnail.jpg",
  "stream_url": "https://cdn.streamhub.com/videos/vid_12345/master.m3u8",
  "available_qualities": ["1080p", "720p", "480p", "360p"],
  "stats": {
    "views": 52400,
    "likes": 3200
  }
}
```

---

### 2.2 Record View Activity (Heartbeat)
The video player on your phone or computer sends this ping every 30 seconds while you watch to update view counts and watch time statistics.

* **Method:** `POST`
* **Path:** `/api/v1/telemetry/playback-pulse`

#### Request Body
```json
{
  "video_id": "vid_12345",
  "current_time_seconds": 120,
  "current_quality": "720p"
}
```

#### Response (`204 No Content`)

---

## 3. Realtime Notification (WebSocket)

When a creator uploads a video, their browser stays connected via WebSocket to receive live progress updates:

* **Endpoint:** `wss://ws.streamhub.com/v1/notifications`

### Example Progress Event
```json
{
  "event": "CONVERSION_PROGRESS",
  "video_id": "vid_12345",
  "progress_percentage": 65,
  "status": "CONVERTING_720P"
}
```

### Example Video Ready Event
```json
{
  "event": "VIDEO_READY",
  "video_id": "vid_12345",
  "stream_url": "https://cdn.streamhub.com/videos/vid_12345/master.m3u8"
}
```
