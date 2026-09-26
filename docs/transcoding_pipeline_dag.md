# Distributed Transcoding Pipeline & DAG Orchestration

This document details how large raw media files are partitioned, dispatched across compute clusters, transcoded in parallel, and stitched into streaming manifests using Directed Acyclic Graph (DAG) workflows.

---

## 1. The Core Engineering Challenge

Monolithic video encoding fails at scale for three fundamental reasons:
1. **Head-of-Line Blocking**: Long videos (e.g., 2-hour podcasts) hog encoder nodes for hours, delaying short viral clips.
2. **Blast Radius of Crashes**: If an encoder crashes at minute 119 of a 120-minute video, the entire 2 hours of compute must be re-run from scratch.
3. **High Turnaround Latency**: A user must wait until the full video is encoded before they can even preview the first 10 seconds.

---

## 2. Chunk-Based Transcoding DAG Architecture

```mermaid
graph TD
    Raw[Raw Ingest Video] --> Inspector[Media Inspector & Demuxer]
    
    subgraph Stage 1: Demuxing & Keyframe Splitting
        Inspector --> SplitTask[GOP Splitter]
        SplitTask --> C0[Chunk 0: 0-10s]
        SplitTask --> C1[Chunk 1: 10-20s]
        SplitTask --> CN[Chunk N: ...]
        Inspector --> AudioTask[Extract Audio Master .wav]
    end

    subgraph Stage 2: Parallel Video Transcoding
        C0 --> E0_1080[1080p Worker]
        C0 --> E0_720[720p Worker]
        C0 --> E0_480[480p Worker]
        
        C1 --> E1_1080[1080p Worker]
        C1 --> E1_720[720p Worker]
        C1 --> E1_480[480p Worker]
        
        AudioTask --> Audio_AAC[AAC-LC 128k]
        AudioTask --> Audio_Opus[Opus 96k]
    end

    subgraph Stage 3: Extraction & AI Tasks
        C0 & C1 --> Storyboard[Storyboard Sprite Generator]
        C0 --> Thumb[Auto-Thumbnail Selector]
        Audio_AAC --> Whisper[Speech-to-Text / Subtitle Generator]
    end

    subgraph Stage 4: Assembly & Packaging
        E0_1080 & E1_1080 --> Playlist1080[Generate 1080p.m3u8]
        E0_720 & E1_720 --> Playlist720[Generate 720p.m3u8]
        Playlist1080 & Playlist720 & Audio_AAC & Whisper --> Master[Master Manifest Packager]
        Master --> Publish[Mark Video READY]
    end
```

---

## 3. Keyframe (GOP) Alignment & Splitting

Video compression uses three frame types:
* **I-Frames (Intra-frames / Keyframes)**: Standalone complete images.
* **P-Frames (Predicted)**: Delta changes based on previous frames.
* **B-Frames (Bi-directional)**: Delta changes based on both preceding and succeeding frames.

> [!IMPORTANT]
> **Splitting Rule**: A video file can **ONLY** be cut on an **I-frame (Keyframe)**. If split on a P/B frame without reference frames, visual artifacts and decoding crashes occur.

### Closed GOP Enforcement
During transcoding, all output chunks are forced to use **Fixed Closed GOP** intervals (e.g., GOP size = 60 frames = exactly 2.000 seconds at 30fps).
This guarantees that:
1. Every segment starts with an IDR (Instantaneous Decoder Refresh) keyframe.
2. Segments across different resolutions (1080p vs 720p) have **identical temporal boundaries**, allowing seamless mid-stream ABR switching.

---

## 4. Worker Queue & Task Scheduling

The DAG is coordinated using a distributed orchestrator (e.g., Temporal / Apache Airflow / custom Kafka workers):

```mermaid
flowchart LR
    Scheduler[DAG Coordinator] -->|Push Tasks| HighPri[GPU Priority Queue: 1080p Chunks]
    Scheduler -->|Push Tasks| LowPri[CPU Spot Queue: 360p / Thumbnails]
    
    HighPri --> GPU_Worker1[GPU Node 1]
    HighPri --> GPU_Worker2[GPU Node 2]
    LowPri --> CPU_Worker1[Spot Node 1]
    
    GPU_Worker1 -->|Report Finished Chunk| Redis[(Redis Task State)]
    Redis --> Scheduler
```

### Worker Lease & Failure Recovery
1. When a worker picks up a chunk task `(video_id, chunk_index, profile)`, it acquires a 60-second heartbeat lease in Redis.
2. If the worker encounters an OOM / node crash, the lease expires.
3. The DAG Scheduler re-queues the chunk task to another node with zero data loss.
4. Finished chunk segments are written directly to S3 under `s3://processed-media/{video_id}/{profile}/segment_{index}.ts`.
