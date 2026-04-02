# High-Value Content

## Overview

For sessions designated as high-value, the system maintains a duplicate recording on a separate physical path. This protects against:

- Single-file corruption during the 30-day hot window
- NAS drive failure during a critical session
- Any single point of failure in the capture or storage chain

High-value designation is either **pre-set** (e.g., VIP tables always run in duplicate) or **triggered in real-time** from the control room.

---

## What "High-Value" Means in Practice

Examples:
- VIP or high-wager sessions above a defined threshold
- Sessions involving a regulatory observer or compliance audit
- Any session manually flagged by control room staff
- Tables designated as "certified" for regulatory purposes

---

## Duplicate Recording Architecture

When a session is high-value, the capture container writes Track B (HUD) to two separate paths simultaneously:

```
Primary:    /mnt/nas/{studio}/{table}/track_b_hud/{date}/{hour}.mp4
Duplicate:  /mnt/nas-secondary/{studio}/{table}/track_b_hud/{date}/{hour}.mp4
```

Track A (raw feed) is duplicated identically:
```
Primary:    /mnt/nas/{studio}/{table}/track_a_raw/{date}/{hour}.mp4
Duplicate:  /mnt/nas-secondary/{studio}/{table}/track_a_raw/{date}/{hour}.mp4
```

The secondary path is either:
- A **second volume** on the same Synology (different RAID group, different drives)
- A **second NAS** at the same studio (preferred for highest redundancy)

FFmpeg writes to both simultaneously via two independent output targets — there is no intermediary copy step that could introduce a gap.

```bash
# FFmpeg dual-output for Track B HUD (high-value mode)
ffmpeg -f x11grab -display_name "${DISPLAY_NUM}" \
  -video_size 3840x2160 -framerate 10 \
  -i "${DISPLAY_NUM}" \
  -map 0 -c:v libx265 -preset ultrafast -crf 28 \
  -f segment -segment_time 3600 -strftime 1 \
  "${NAS_PRIMARY}/track_b_hud/%Y-%m-%d/%H-%M-%S.mp4" \
  -map 0 -c:v libx265 -preset ultrafast -crf 28 \
  -f segment -segment_time 3600 -strftime 1 \
  "${NAS_SECONDARY}/track_b_hud/%Y-%m-%d/%H-%M-%S.mp4"
```

---

## S3 Upload for Duplicates

Duplicates are uploaded to a **separate S3 prefix** (optionally a separate bucket, or a separate AWS region for maximum redundancy):

```
Primary:    s3://surveillance-primary/recordings/{studio}/{table}/...
Duplicate:  s3://surveillance-duplicate/recordings/{studio}/{table}/...
```

Both apply S3 Object Lock independently. The duplicate is subject to the same default 180-day retention unless the session is marked for hold.

---

## Discard Behavior

If a high-value session is **not** subsequently marked for retention:

- Both primary and duplicate follow the standard 180-day lifecycle
- At Day 180, both are auto-purged by S3 lifecycle policy
- No manual action required — the duplicate does not extend the retention period

If a high-value session **is** marked for retention:

- The hold tag is applied to both primary and duplicate independently
- Both are moved to Glacier Deep Archive after Day 180
- Retention can only be released by an authorized second user

---

## Activating High-Value Mode

### Pre-set (static, per table)

Configure in the capture container's environment:

```yaml
environment:
  HIGH_VALUE: "true"
  NAS_SECONDARY: "/mnt/nas-secondary"
```

### Real-time trigger (from control room)

The watchdog service exposes a control API:

```http
POST /api/v1/streams/{studio}/{table}/high-value
{
  "enabled": true,
  "reason": "VIP session - player wagering above $10k threshold",
  "triggered_by": "ops@control"
}
```

On activation:
1. Watchdog signals the capture container to add the second FFmpeg output
2. A new set of segments begins on the secondary path from that moment forward (no retroactive duplication)
3. The activation is logged to the audit trail with timestamp, user, and reason

> **Note:** Real-time activation does not duplicate already-recorded segments. If you anticipate a high-value session, activate before it begins or use the pre-set table configuration.

---

## Integrity Verification

For high-value content, checksums are cross-verified between primary and duplicate after each segment closes:

```go
func VerifyDuplicate(primary, duplicate string) error {
    primaryHash := sha256File(primary)
    duplicateHash := sha256File(duplicate)
    if primaryHash != duplicateHash {
        alert("CRIT", "high-value duplicate checksum mismatch", primary)
        return ErrChecksumMismatch
    }
    return nil
}
```

A mismatch triggers an immediate critical alert. The segment is flagged for manual review — it is not deleted, and the discrepancy is preserved in the audit log.
