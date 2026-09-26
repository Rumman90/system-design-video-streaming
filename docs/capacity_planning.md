# Capacity Planning & Scale Estimations

A comprehensive mathematical breakdown of compute, network, memory, and storage footprints required to operate an on-demand video platform serving 100 Million Daily Active Users (DAU).

---

## 1. Traffic Assumptions & Base Metrics

| Parameter | Baseline Value | Notes |
| :--- | :--- | :--- |
| **Daily Active Users (DAU)** | 100,000,000 | Global active viewers |
| **Daily Video Uploads** | 500,000 videos/day | ~5.8 uploads/sec average, ~25 uploads/sec peak |
| **Average Video Length** | 10 minutes (600 seconds) | Standard user-generated content |
| **Raw Upload Quality & Bitrate** | 1080p @ 6.5 Mbps | Raw MP4 ingest |
| **Total Daily Views** | 1,000,000,000 views/day | 10 views per DAU average |
| **Average Watch Time** | 30 minutes/user/day | 3 billion total minutes viewed per day |
| **Peak Traffic Multiplier** | 2.5x to 3.0x | Evening primetime spikes |

---

## 2. Ingestion & Storage Mathematics

### 2.1 Ingestion Storage & Bandwidth
* **Raw File Size**:
  $$\text{File Size} = 600\text{ sec} \times \frac{6.5\text{ Mbps}}{8\text{ bits/byte}} = 487.5\text{ MB} \approx 500\text{ MB}$$
* **Daily Ingest Volume**:
  $$500,000\text{ uploads} \times 500\text{ MB} = 250\text{ TB / day}$$
* **Average Ingest Bandwidth**:
  $$\text{Bandwidth} = \frac{250\text{ TB} \times 8\text{ bits/byte}}{86,400\text{ sec}} \approx 23.15\text{ Gbps}$$
* **Peak Ingest Bandwidth (2.5x)**: $\approx 58\text{ Gbps}$

---

### 2.2 Transcoding Storage Multiplier
Each video is transcoded into multiple representations for Adaptive Bitrate Streaming (ABR):

| Resolution | Video Bitrate | Audio Bitrate | Format / Codec | Storage per 10-min Video |
| :--- | :--- | :--- | :--- | :--- |
| **4K (2160p)** *(Optional)* | 15.0 Mbps | 192 kbps | AV1 / VP9 | ~1.14 GB |
| **1080p (FHD)** | 4.5 Mbps | 128 kbps | H.264 / VP9 | ~347 MB |
| **720p (HD)** | 2.2 Mbps | 128 kbps | H.264 | ~174 MB |
| **480p (SD)** | 1.0 Mbps | 96 kbps | H.264 | ~82 MB |
| **360p** | 500 kbps | 64 kbps | H.264 | ~42 MB |
| **240p** | 250 kbps | 48 kbps | H.264 | ~22 MB |
| **Total per Video** | — | — | **All Profiles** | **~667 MB** |

* **Daily Transcoded Storage**:
  $$500,000 \times 667\text{ MB} \approx 333.5\text{ TB / day}$$
* **Combined Daily Storage (Raw + Processed)**: $\approx 583.5\text{ TB / day}$
* **5-Year Cold/Warm Storage Projection**:
  $$\text{5-Year Storage} = 583.5\text{ TB/day} \times 365 \times 5 \approx 1,065\text{ PB} \approx 1.06\text{ Exabytes}$$

---

## 3. Streaming Egress Bandwidth & CDN Capacity

### 3.1 Peak Concurrent Viewers
* **Total Daily Viewing Minutes**: $100\text{M users} \times 30\text{ min} = 3\text{ Billion minutes/day}$
* **Average Concurrent Streams**:
  $$\text{Avg Streams} = \frac{3,000,000,000\text{ min}}{1,440\text{ min/day}} \approx 2,083,333\text{ streams}$$
* **Peak Concurrent Streams (3.5x peak evening ratio)**:
  $$\text{Peak Streams} = 2.08\text{M} \times 3.5 \approx 7,300,000\text{ concurrent streams}$$

### 3.2 Total Outbound Bandwidth
* Assuming viewing distribution:
  * 40% at 1080p (4.5 Mbps)
  * 35% at 720p (2.2 Mbps)
  * 20% at 480p (1.0 Mbps)
  * 5% at 360p/240p (0.4 Mbps)
* **Weighted Average Bitrate**:
  $$\bar{R} = (0.40 \times 4.5) + (0.35 \times 2.2) + (0.20 \times 1.0) + (0.05 \times 0.4) \approx 2.79\text{ Mbps}$$
* **Peak Egress Bandwidth**:
  $$\text{Peak Egress} = 7,300,000\text{ streams} \times 2.79\text{ Mbps} \approx 20.36\text{ Terabits per second (Tbps)}$$

---

## 4. CDN Cache Sizing (80/20 Rule)

According to the Pareto principle, **20% of the popular videos generate 80% of daily views**.
* Daily viewed data for top 20% videos:
  $$20\% \times 333.5\text{ TB} \approx 66.7\text{ TB / day}$$
* **Target CDN Hit Ratio**: $\ge 95\%$
* **Edge CDN Memory/SSD Working Set**:
  Distributing a 66.7 TB hot working set across 200 global CDN Point-of-Presence (PoP) locations requires:
  $$\frac{66.7\text{ TB}}{200\text{ PoPs}} \approx 335\text{ GB NVMe / RAM per PoP}$$
  This is extremely cost-effective to cache completely in edge NVMe/RAM buffers.

---

## 5. Transcoding Compute Cluster Sizing

* Total video duration ingested per day:
  $$500,000\text{ videos} \times 10\text{ min} = 5,000,000\text{ video minutes / day}$$
* **Hardware Transcoding Speedup Factor**:
  Modern GPU transcoders (e.g., NVIDIA T4 / L4 NVENC) can transcode 1080p video at ~6x realtime ($1\text{ minute of video takes } 10\text{ seconds}$).
* Total GPU-minutes required per day across all 5 profiles:
  $$\text{GPU Minutes} = 5,000,000 \times \left( \frac{5\text{ profiles}}{6\text{ speedup}} \right) \approx 4,166,666\text{ GPU minutes / day}$$
* Total Active GPUs needed:
  $$\text{GPUs Needed} = \frac{4,166,666}{1,440\text{ min/day}} \approx 2,893\text{ GPUs}$$
* **With 2x Peak Buffer & Redundancy**: Auto-scaling cluster of **~5,800 GPU worker instances**.
