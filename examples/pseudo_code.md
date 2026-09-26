# Video Streaming Implementation Pseudo-code

This document provides sample pseudo-code for the critical operations: GOP chunking, parallel FFmpeg worker execution, and client ABR bitrate selection.

---

## 1. Transcoding Worker: Splitting and FFmpeg Chunk Encoding (Python)

```python
import subprocess
import os

class TranscoderWorker:
    def __init__(self, s3_client, redis_client):
        self.s3 = s3_client
        self.redis = redis_client

    def process_chunk_task(self, video_id: str, chunk_index: int, start_time_sec: float, duration_sec: float, profile: dict):
        """
        Transcodes a single GOP segment into target profile with exact keyframe alignment.
        """
        lease_key = f"lock:chunk:{video_id}:{chunk_index}:{profile['resolution']}"
        worker_id = os.getenv("HOSTNAME", "worker_node_1")

        # 1. Acquire Redis Lease (60s TTL)
        if not self.redis.set(lease_key, worker_id, nx=True, ex=60):
            print(f"Task already owned by another worker, skipping.")
            return

        input_chunk_path = f"/tmp/{video_id}_input.mp4"
        output_ts_path = f"/tmp/{video_id}_{profile['resolution']}_seg{chunk_index:04d}.ts"

        try:
            # 2. Transcode chunk using hardware accelerated FFmpeg with closed GOP
            cmd = [
                "ffmpeg", "-y",
                "-ss", str(start_time_sec),
                "-t", str(duration_sec),
                "-i", input_chunk_path,
                "-vf", f"scale={profile['width']}:{profile['height']}",
                "-c:v", "libx264",
                "-b:v", f"{profile['bitrate_kbps']}k",
                "-maxrate", f"{int(profile['bitrate_kbps'] * 1.2)}k",
                "-bufsize", f"{int(profile['bitrate_kbps'] * 2)}k",
                "-g", "120",                # GOP Size: Force I-frame every 4 seconds (30fps)
                "-keyint_min", "120",
                "-sc_threshold", "0",        # Disable dynamic scene cut keyframes
                "-c:a", "aac",
                "-b:a", f"{profile['audio_bitrate_kbps']}k",
                "-f", "mpegts",
                output_ts_path
            ]
            
            subprocess.run(cmd, check=True)

            # 3. Upload chunk directly to Processed Media S3 Bucket
            s3_key = f"videos/{video_id}/{profile['resolution']}/segment_{chunk_index:04d}.ts"
            self.s3.upload_file(output_ts_path, "processed-media-bucket", s3_key)

            # 4. Mark chunk complete in Redis Task Hash
            self.redis.hincrby(f"job:video:{video_id}", "completed_chunks", 1)
            print(f"Successfully processed chunk {chunk_index} for {profile['resolution']}")

        finally:
            if os.path.exists(output_ts_path):
                os.remove(output_ts_path)
            self.redis.delete(lease_key)
```

---

## 2. Client-Side Adaptive Bitrate (ABR) Selection Algorithm (JavaScript / TypeScript)

```typescript
interface RenditionVariant {
  resolution: string;
  bandwidthBps: number; // e.g. 4500000 for 1080p
}

class AdaptiveBitrateEngine {
  private variants: RenditionVariant[];
  private safetyHeadroomFactor: number = 0.8; // Use max 80% of measured bandwidth

  constructor(variants: RenditionVariant[]) {
    // Sort renditions in descending order of bandwidth
    this.variants = variants.sort((a, b) => b.bandwidthBps - a.bandwidthBps);
  }

  public selectOptimalVariant(
    measuredBandwidthBps: number,
    bufferDurationSec: number
  ): RenditionVariant {
    // 1. Buffer Danger Rule (Avoid playback stalls)
    if (bufferDurationSec < 5.0) {
      // Pick lowest available quality immediately to prevent freeze
      return this.variants[this.variants.length - 1];
    }

    // 2. Safe Bandwidth Calculation
    const effectiveBandwidth = measuredBandwidthBps * this.safetyHeadroomFactor;

    // 3. Find highest quality that fits inside effective bandwidth
    for (const variant of this.variants) {
      if (effectiveBandwidth >= variant.bandwidthBps) {
        return variant;
      }
    }

    // Default to lowest resolution if bandwidth is extremely constrained
    return this.variants[this.variants.length - 1];
  }
}
```
