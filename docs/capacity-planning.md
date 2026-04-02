# Capacity Planning (3-Year)

## Assumptions

| Parameter | Value | Notes |
|-----------|-------|-------|
| Recording tracks | 2 per table | Track A (raw feed, 2 Mbps) + Track B (HUD, 3 Mbps) |
| Proxy stream | 1 Mbps per table | 720p, not stored — live VPN monitoring only |
| Total encode bitrate | 5 Mbps per table | HEVC, 10fps, CRF 28 |
| Storage per table | ~2.2 GB/hour / ~52.7 GB/day | Both tracks combined |
| Segment size | ~7.9 GB per segment | 1-hour file at 5 Mbps average |
| Default retention | 180 days | Hot 30d + Warm 60d + Cold 90d |
| Growth rate | +4–10 tables/year | Planning uses worst case (+10) |

---

## 3-Year Storage Projections

| Stage | Tables | Daily Write | Hot Tier (30d) | Warm Tier (60d) | Cold Tier (90d) | Total Rolling |
|-------|--------|-------------|----------------|-----------------|-----------------|---------------|
| Launch | 4 | 0.21 TB | 6.2 TB | 12.4 TB | 18.5 TB | **37 TB** |
| Year 1 | 14 | 0.72 TB | 21.6 TB | 43.3 TB | 64.9 TB | **130 TB** |
| Year 2 | 24 | 1.24 TB | 37.1 TB | 74.2 TB | 111 TB | **222 TB** |
| Year 3 | 34 | 1.75 TB | 52.5 TB | 105 TB | 157 TB | **315 TB** |

> **Note:** "Total Rolling" is total active storage across all tiers at steady state, assuming 180-day retention and no marked-for-retention content. Marked content will exceed these figures proportionally.

---

## NAS Sizing (Per Studio)

Assuming by Year 3 there are 3 studios with ~11 tables each:

| Year | Tables/Studio | Daily Write | 30-Day Hot Storage | Recommended NAS |
|------|--------------|-------------|-------------------|-----------------|
| Launch | 4 | 211 GB/day | 6.2 TB | Synology DS923+ (4× 8TB, SHR-2) |
| Year 1 | ~5 | 264 GB/day | 7.9 TB | Expand DS923+ to 4× 12TB |
| Year 2 | ~8 | 422 GB/day | 12.7 TB | Migrate to RS1221+ or RS3621xs+ |
| Year 3 | ~11 | 580 GB/day | 17.4 TB | 12-bay, 12× 8TB drives (RAID 6) |

**NVMe SSD Cache:** Required from day one. Two M.2 NVMe SSDs in Read/Write Cache mode allow fast seeks without interrupting the sequential write workload. Minimum 400GB per cache drive; 1TB recommended by Year 2.

---

## Compute Sizing (Per Studio Capture Server)

| Year | Tables | Chrome RAM | Encode Load | Recommended Spec |
|------|--------|-----------|-------------|-----------------|
| Launch | 4 | ~6 GB | Trivial | Intel Core i7 12th gen, 32 GB RAM |
| Year 1 | ~5 | ~8 GB | Light | Same spec, add iGPU for QuickSync |
| Year 2 | ~8 | ~12 GB | Moderate | Xeon/Ryzen + GTX/RTX for NVENC, 64 GB RAM |
| Year 3 | ~11 | ~17 GB | Moderate | RTX 3060+, 64–128 GB RAM |

One mid-range GPU (RTX 3060 or better) handles 11+ simultaneous HEVC encodes at 10fps/4K via NVENC.

---

## Network Requirements

| Stage | Tables | Internal Write | Proxy Streams (VPN) | Required Switch |
|-------|--------|---------------|---------------------|----------------|
| Launch | 4 | 20 Mbps | 4 Mbps | 1GbE sufficient |
| Year 1 | 14 | 70 Mbps | 14 Mbps | 1GbE sufficient |
| Year 2 | 24 | 120 Mbps | 24 Mbps | 1GbE sufficient (upgrade NAS NIC) |
| Year 3 | 34 | 170 Mbps | 34 Mbps | 10GbE NAS NIC + managed switch |

**Control room upstream:** Sum of all proxy streams across all studios. At Year 3 with 34 tables: 34 Mbps sustained upload from each studio to control room. A 100 Mbps WAN connection per studio is sufficient.

---

## AWS Cost Estimates (Steady State)

Estimates use us-west-2 pricing, approximate, and exclude data transfer costs within a VPC.

| Year | S3 Standard-IA | S3 Glacier IR | Total S3/month |
|------|---------------|--------------|----------------|
| Launch | 12 TB × $0.0125 = $150 | 19 TB × $0.004 = $76 | ~$226/month |
| Year 1 | 43 TB × $0.0125 = $538 | 65 TB × $0.004 = $260 | ~$798/month |
| Year 2 | 74 TB × $0.0125 = $925 | 111 TB × $0.004 = $444 | ~$1,369/month |
| Year 3 | 105 TB × $0.0125 = $1,313 | 158 TB × $0.004 = $632 | ~$1,945/month |

> S3 Glacier Deep Archive ($0.00099/GB) for marked-for-retention content beyond 180 days is negligible unless a large volume of content is held. Budget $50–$200/month for this as a buffer.

---

## Hardware Capital Cost (One-Time, Per Studio)

| Item | Launch | Year 2 Upgrade |
|------|--------|----------------|
| Synology NAS | $700 | $1,200 (larger chassis) |
| Hard drives | $500 (4× 8TB) | $1,000 (add drives) |
| NVMe cache pair | $200 | $400 (larger) |
| Capture server | $800 (refurb i7 workstation) | $1,500 (add GPU) |
| 10GbE switch (studio) | $300 | — |
| **Total per studio** | **~$2,500** | **~$4,100 cumulative** |
