# Capacity Planning & Scale Estimations

This document explains the math behind storage, bandwidth, and server requirements for a platform with 100 Million daily active users.

---

## 1. Basic System Assumptions

| Assumption | Number | Explanation |
| :--- | :--- | :--- |
| **Daily Active Users (DAU)** | 100 Million | People who use the platform every day |
| **Daily Video Uploads** | 500,000 videos | New videos uploaded each day |
| **Average Video Length** | 10 minutes | Standard video length |
| **Daily Video Views** | 1 Billion views | On average, 10 videos watched per active user |
| **Average User Watch Time** | 30 minutes | Total viewing time per user per day |

---

## 2. Storage Math (How much disk space is needed?)

### 2.1 Raw Upload Storage
* Average size of a raw 10-minute 1080p video = **500 Megabytes (MB)**.
* Daily raw storage needed:
```text
Daily Raw Storage = 500,000 uploads × 500 MB
                  = 250,000,000 MB
                  = 250 TB / day
```

### 2.2 Converted Formats Storage
To allow smooth playback on all devices, each video is converted into 4 common qualities:
* **1080p (Full HD):** ~300 MB
* **720p (HD):** ~150 MB
* **480p (SD):** ~70 MB
* **360p (Low):** ~35 MB
* **Total converted size per video:** ~555 MB per video

* Daily converted storage needed:
```text
Daily Transcoded Storage = 500,000 uploads × 555 MB ≈ 277 TB / day
Total Combined Storage   = 250 TB + 277 TB ≈ 527 TB / day
```

---

## 3. Bandwidth Math (How much network speed is needed?)

### 3.1 Peak Concurrent Viewers
* Total watch time every day:
```text
Total Daily Watch Time = 100 Million users × 30 minutes = 3 Billion minutes / day
```
* During evening peak hours, about **7 Million users** watch videos at the exact same second.

### 3.2 Total Outbound Bandwidth
* Average internet speed required to stream a typical mix of 720p and 1080p video = **2.5 Mbps**.
* Peak bandwidth across the entire network:
```text
Peak Network Egress = 7,000,000 users × 2.5 Mbps
                    = 17,500,000 Mbps
                    = 17.5 Terabits per second (Tbps)
```

> This massive amount of network traffic is handled by global Content Delivery Networks (CDNs), which distribute the load across thousands of servers worldwide.

---

## 4. Server & Worker Sizing

* **Total video time uploaded daily:**
```text
Total Video Minutes = 500,000 videos × 10 minutes = 5,000,000 minutes of video / day
```
* Fast GPU computers can convert video at **6x speed** (1 minute of video takes 10 seconds of processing time).
* To process all uploads on time with room for peak hours, the system maintains a pool of approximately **2,500 to 4,000 processing worker servers** that automatically scale up when uploads increase.
