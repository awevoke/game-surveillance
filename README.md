# Game Surveillance System

Recording, archival, and playback infrastructure for live gameshow HUD streams.

## Purpose

Each gameshow table runs a presenter-facing HUD — a 52" display driven by a web application showing the live video feed, game chat, control room messages, and game instructions. This system captures that HUD (and the underlying raw video feed) for two primary purposes:

1. **Game integrity** — immutable, tamper-proof record of what was displayed and dealt
2. **Presenter review** — production team and central control room can review presenter performance

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | System topology, components, per-studio layout |
| [Recording Pipeline](docs/recording-pipeline.md) | Capture stack: browser → FFmpeg → NAS |
| [Data Retention Policy](docs/data-retention.md) | Tiers, retention windows, immutability rules |
| [Capacity Planning](docs/capacity-planning.md) | 3-year storage and network projections |
| [Playback System](docs/playback.md) | Onsite, VPN, and archive playback modes |
| [Monitoring & Watchdog](docs/monitoring.md) | Health checks, alerting, failure recovery |
| [High-Value Content](docs/high-value-content.md) | Duplicate recording and extended retention |

## Quick Reference

- **Tables at launch:** 4
- **Growth rate:** 4–10 tables/year
- **Retention default:** 180 days, then auto-purge
- **Recording tracks per table:** 2 (raw video feed + HUD screen capture)
- **Format:** HEVC (H.265), 10fps, segmented 1-hour files
- **Primary storage:** Per-studio Synology NAS
- **Archive:** AWS S3 Standard-IA → S3 Glacier (with Object Lock)
- **Remote access:** Tailscale (offsite) / Site-to-site VPN (control room to studios)
