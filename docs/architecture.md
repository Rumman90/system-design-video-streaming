# System Architecture: Video Streaming Platform

This document explains the main components of the video streaming system and how they talk to each other in simple terms.

---

## 1. The Four Main Parts of the System

To keep things organized, the system is split into four simple layers:

```mermaid
graph TD
    subgraph 1. User & Control Layer
        API[API Gateway]
        Auth[User & Login Service]
        MetadataSvc[Video Details Service]
    end

    subgraph 2. Video Processing Layer
        UploadSvc[Upload Manager]
        TaskQueue[Task Queue]
        Workers[Transcoding Workers]
        Packager[Playlist Packager]
    end

    subgraph 3. Storage Layer
        RawStorage[(Raw Video Storage)]
        ProcessedStorage[(Processed Video Slices)]
        Database[(Metadata Database)]
        Cache[(Redis Fast Cache)]
    end

    subgraph 4. Delivery Layer
        CDN[Global CDN Edge Servers]
        Viewer[Viewer App / Browser]
    end

    API --> MetadataSvc & UploadSvc
    UploadSvc --> RawStorage
    UploadSvc --> TaskQueue --> Workers --> Packager --> ProcessedStorage
    Viewer --> CDN --> ProcessedStorage
```

---

## 2. Part 1: How Videos are Uploaded (Direct Upload)

### Why not upload directly to the main web server?
If a creator uploads a 2 GB video through the normal website server, two bad things happen:
* The web server gets busy holding this huge file in memory and slows down for everyone else.
* If the user's connection drops at 95%, the whole upload is lost.

### The Solution: Direct Multipart Upload
1. The user asks the API for permission to upload a video.
2. The API gives the user special temporary upload links (called presigned URLs) directly to cloud storage (like Amazon S3).
3. The user's browser breaks the video into 8 MB parts and uploads them directly to cloud storage in parallel.
4. Once all parts finish, cloud storage puts the parts together into the final raw file and informs the system.

---

## 3. Part 2: How Videos are Processed (Transcoding)

Transcoding means converting one video into multiple resolutions and formats.

```mermaid
flowchart LR
    Source[Raw 1080p Video] --> Cutter[Cut into 4-Second Slices]
    
    Cutter --> Slice1[Slice 1: 0 to 4s]
    Cutter --> Slice2[Slice 2: 4 to 8s]
    Cutter --> Slice3[Slice 3: 8 to 12s]
    
    Slice1 --> W1[Worker A: Make 1080p, 720p, 480p]
    Slice2 --> W2[Worker B: Make 1080p, 720p, 480p]
    Slice3 --> W3[Worker C: Make 1080p, 720p, 480p]

    W1 & W2 & W3 --> Merger[Create Master Playlist]
```

* **Why cut into 4-second slices?**
  If we try to convert a 2-hour movie on one single computer, it takes hours. But if we split it into small 4-second slices, 50 worker computers can work on different slices at the exact same time. The video gets ready in just a few minutes.
* **Smart Failure Recovery:**
  If Worker B crashes while working on Slice 2, the system automatically gives Slice 2 to another worker without having to restart the whole video.

---

## 4. Part 3: Delivering Videos with CDN

```mermaid
sequenceDiagram
    participant User as Viewer
    participant LocalCDN as Local City Cache (CDN)
    participant CloudStorage as Main Cloud Storage

    User->>LocalCDN: Give me slice 001.ts
    alt Slice is already in local city cache (Cache Hit)
        LocalCDN-->>User: Sends video immediately (No delay)
    else Slice is not in local cache (Cache Miss)
        LocalCDN->>CloudStorage: Fetch slice from main storage
        CloudStorage-->>LocalCDN: Returns slice
        LocalCDN-->>User: Sends video and saves a copy locally for next viewer
    end
```

* **What is a CDN?**
  A Content Delivery Network (CDN) is a network of servers placed in hundreds of cities worldwide.
* When a popular video is uploaded, its slices are cached on servers near viewers. A viewer in Tokyo gets the video from Tokyo, and a viewer in London gets it from London.

---

## 5. Part 4: Counting Views Without Crashing

When a video goes viral and millions of people watch it at the same second, updating a database row (`views = views + 1`) millions of times per second will lock the database and crash the site.

```mermaid
flowchart LR
    Viewer[Viewer watches 30 seconds] --> API[View Tracking API]
    API --> Queue[Temporary Message Queue]
    Queue --> FastCache[(Redis Memory Counter)]
    FastCache -->|Save in bulk every 10 seconds| MainDB[(Main Database)]
```

* View counts are first incremented in super-fast memory (Redis).
* Every 10 seconds, the totals are written in bulk to the main database. This keeps the database fast and healthy.
