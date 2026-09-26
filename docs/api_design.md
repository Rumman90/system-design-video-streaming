# API Design & Contracts: Video Streaming Platform

This document outlines the RESTful endpoints, streaming manifest contracts, and real-time WebSocket interfaces.

---

## 1. Video Ingestion & Upload APIs

### 1.1 Initiate Upload Session
Initiates a multipart upload session and returns signed chunk URLs.

- **Method**: `POST`
- **Path**: `/api/v1/videos/upload-session`
- **Headers**: `Authorization: Bearer <token>`, `Content-Type: application/json`

#### Request Payload
```json
{
  "title": "System Design Video Streaming Deep Dive",
  "description": "Complete architectural walkthrough of a video streaming platform.",
  "filename": "video_recording.mp4",
  "file_size_bytes": 1073741824,
  "mime_type": "video/mp4",
  "checksum_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "chunk_size_bytes": 10485760
}
```

#### Response Payload (`201 Created`)
```json
{
  "video_id": "vid_9a8b7c6d5e4f",
  "upload_id": "s3_upload_session_xyz890",
  "chunk_size_bytes": 10485760,
  "total_parts": 103,
  "presigned_parts": [
    {
      "part_number": 1,
      "upload_url": "https://raw-storage.streamhub.com/raw/vid_9a8b7c6d5e4f/part1?signature=abc...",
      "expires_at": 1774598400
    },
    {
      "part_number": 2,
      "upload_url": "https://raw-storage.streamhub.com/raw/vid_9a8b7c6d5e4f/part2?signature=def...",
      "expires_at": 1774598400
    }
  ]
}
```

---

### 1.2 Complete Upload Session
Notifies the system that all parts have been transferred directly to object storage.

- **Method**: `POST`
- **Path**: `/api/v1/videos/{video_id}/complete-upload`

#### Request Payload
```json
{
  "upload_id": "s3_upload_session_xyz890",
  "parts": [
    {"part_number": 1, "etag": "\"d41d8cd98f00b204e9800998ecf8427e\""},
    {"part_number": 2, "etag": "\"0cc175b9c0f1b6a831c399e269772661\""},
    {"part_number": 103, "etag": "\"92eb5ffee6ae2fec3ad71c777531578f\""}
  ]
}
```

#### Response Payload (`202 Accepted`)
```json
{
  "video_id": "vid_9a8b7c6d5e4f",
  "status": "PROCESSING",
  "progress_percentage": 0,
  "estimated_completion_seconds": 180,
  "playback_url": null
}
```

---

## 2. Video Playback & Metadata APIs

### 2.1 Get Video Details & Streaming Links
Fetches metadata, view statistics, and the primary HLS/DASH manifest URLs.

- **Method**: `GET`
- **Path**: `/api/v1/videos/{video_id}`

#### Response Payload (`200 OK`)
```json
{
  "video_id": "vid_9a8b7c6d5e4f",
  "title": "System Design Video Streaming Deep Dive",
  "description": "Complete architectural walkthrough of a video streaming platform.",
  "duration_seconds": 612,
  "channel": {
    "channel_id": "chn_101",
    "name": "Distributed Systems Pro",
    "avatar_url": "https://cdn.streamhub.com/avatars/chn_101.jpg"
  },
  "thumbnails": {
    "default": "https://cdn.streamhub.com/videos/vid_9a8b7c6d5e4f/thumb_720p.jpg",
    "storyboard_vtt": "https://cdn.streamhub.com/videos/vid_9a8b7c6d5e4f/storyboard.vtt"
  },
  "streaming": {
    "hls_manifest_url": "https://cdn.streamhub.com/videos/vid_9a8b7c6d5e4f/master.m3u8",
    "dash_manifest_url": "https://cdn.streamhub.com/videos/vid_9a8b7c6d5e4f/manifest.mpd",
    "available_resolutions": ["1080p", "720p", "480p", "360p", "240p"],
    "codecs": ["h264", "vp9", "av1"]
  },
  "statistics": {
    "views_count": 1420500,
    "likes_count": 89400,
    "dislikes_count": 312
  },
  "created_at": "2026-09-26T08:00:00Z"
}
```

---

### 2.2 Record Playback Heartbeat / View Telemetry
Sent periodically by the client player every 30 seconds during active playback.

- **Method**: `POST`
- **Path**: `/api/v1/telemetry/playback-pulse`

#### Request Payload
```json
{
  "video_id": "vid_9a8b7c6d5e4f",
  "session_id": "sess_live_9981a2",
  "current_playback_time_sec": 124.5,
  "current_resolution": "1080p",
  "buffer_health_sec": 14.2,
  "dropped_frames": 0,
  "client_bandwidth_kbps": 18400
}
```

#### Response Payload (`204 No Content`)

---

## 3. Realtime Processing Notification (WebSocket)

Creators subscribe to processing progress over a persistent WebSocket connection:

- **Endpoint**: `wss://ws.streamhub.com/v1/notifications?token=<jwt>`

### Event Payload: Transcoding Progress
```json
{
  "event": "VIDEO_TRANSCODING_PROGRESS",
  "video_id": "vid_9a8b7c6d5e4f",
  "data": {
    "status": "TRANSCODING",
    "completed_chunks": 45,
    "total_chunks": 60,
    "percentage": 75,
    "current_step": "ENCODING_1080P_AV1"
  }
}
```

### Event Payload: Transcoding Complete (Ready to Stream)
```json
{
  "event": "VIDEO_READY",
  "video_id": "vid_9a8b7c6d5e4f",
  "data": {
    "status": "READY",
    "hls_manifest_url": "https://cdn.streamhub.com/videos/vid_9a8b7c6d5e4f/master.m3u8",
    "published_at": "2026-09-26T08:05:22Z"
  }
}
```
