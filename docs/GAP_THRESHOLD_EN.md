# Gap Threshold: When and Why We Stop Early

This document explains the mechanisms that decide when the stream should stop early instead of hanging indefinitely when:
- Segment downloads become too slow (stalling).
- The playlist stops getting new segments.
- The playable buffer repeatedly collapses or remains negative.

Two related internal mechanisms are used:

- Smart / Local VOD modes: `_smart_threshold_reached()` — “intelligent” multi-rule evaluation (buffer state + segment timing).
- Other modes (segment / targetduration / constant): `_segment_queue_timing_threshold_reached()` — simple “no new segments for too long” rule.

Below is a plain-language explanation (no formulas).

---

## What Is the “Buffer” and How to Read It

- Buffer (seconds) ≈ (total delivered playable time) − (wall-clock time since playback start).
- Positive buffer: we are “ahead” — safe cushion.
- Negative buffer: we are “behind” — playback is catching up faster than we download.

Log examples: `buf=12.3s` or `buf=-0.5s`.

---

## Smart (Live smart) and Local VOD: Stop Rules

The check `_smart_threshold_reached()` runs:
- Right after enqueuing new segments (even if some were enqueued — `queued == True`).
- In Local VOD also during waits between batched enqueue cycles.
- When nothing was enqueued in the current loop.

Segment timing “elapsed” is measured from the start of the HTTP request via monotonic clock and shares the same time base as the completion log:
`+ Segment N complete (time=...)`
So decisions align with what you see in the logs.

There are FOUR rules (evaluated in order). The first matching rule stops the stream.

### 1) Stalled Segment + Negative Buffer → STOP
- If at least one in-flight segment has been downloading longer than its EXTINF duration AND the current buffer is already negative → immediate stop.
- Logs:
  - Continue (buffer still positive):  
    `> Segment N exceeded duration but buffer still positive: elapsed=... > dur=...; buf=...s. Continue...`
  - Stop (buffer negative):  
    `=> Segment N exceeded its duration while downloading: elapsed=... > dur=...; buf=...s (<0). Stopping...`

### 2) Repeated Negative Buffer Entries Within a Time Window → STOP
- The system tracks each time the buffer transitions from non-negative to negative.
- A sliding window (default 60s) collects these “negative entry” events.
- If the count reaches the configured limit (default 3 events) within that window → stop.
- Positive intervals inside the same window DO NOT reset the event count; only window expiry resets it (if the threshold was not reached).
- Logs:
  - Each new negative entry:  
    `=> Negative buffer event #2 in current 60s window`
  - Window expires (no threshold reached):  
    `=> Negative buffer window expired (60s) without threshold; counter reset.`
  - Threshold reached (stop):  
    `=> Buffer entered negative state 3 times within 60s window. Stopping...`

### 3) GAP Rule: Buffer Deficit > Length of Last Segment (Only After Previously Positive) → STOP
- Activates only if the buffer was positive at least once earlier in the session.
- If the buffer is now negative and |buffer| > “last segment length estimate” → stop.
- Example: `buf = -5.01s`, last segment length ≈ `5.00s` → stop.
- Log:  
  `=> Buffer deficit exceeded last segment length: |buf|=5.01s > last_seg=5.00s. Stopping...`

“Last segment length” estimation priority:
1. EXTINF of the last segment currently in playlist.
2. Most recent observed segment duration.
3. Smart base duration (baseline of planning).
4. Playlist targetduration.
5. Minimum floor (MIN_RELOAD_FLOOR = 1.00s).

### 4) Buffer Continuously Negative Longer Than Queue Threshold → STOP
- If the buffer stays negative *continuously* longer than the “queue timing threshold,” stop.
- This “queue timing threshold” is computed exactly the same way as in `_segment_queue_timing_threshold_reached()` (targetduration or mean of recent ~3 segments, not below a minimum, multiplied by `hls-segment-queue-threshold`).
- Disabled if `hls-segment-queue-threshold <= 0`.
- Log:  
  `=> Buffer stayed negative for more than X.XXs. Stopping...`

Technical detail:
- Continuous negativity is tracked from the first moment it dips below zero; the timer resets when it becomes non-negative.

---

## Other Modes (segment / targetduration / constant)

If you are not using Smart or Local VOD modes, only the simple “no new segments” timer `_segment_queue_timing_threshold_reached()` applies:

- Base duration = max( fixed minimum (5s), targetduration, average of last up to 3 EXTINF values (if available) ).
- Multiply by `hls-segment-queue-threshold`.
- If no new segment was queued for longer than this threshold → stop.
- Log:  
  `=> No new segments in playlist for more than X.XXs. Stopping...`

If the factor is 0 → this timeout is disabled.

---

## Mode Differences Summary

| Mode | Planning | Stop Rules |
|------|----------|------------|
| Live smart | Buffer-based adaptive reload interval | 1) stalled+negative, 2) repeated negative entries, 3) gap (|buf|>last seg), 4) continuous negative > threshold |
| Local VOD | Batched enqueue; no playlist reload between segments in a batch | Same four smart rules (checked also during waits) |
| segment / targetduration / constant | Fixed / simple timing logic | Only “no new segments” threshold |

---

## Log Examples

Stalled but buffer positive (continue):
```
> Segment 935641 exceeded duration but buffer still positive: elapsed=10.53s > dur=10.43s; buf=1.3s. Continue...
```

Stalled + buffer negative (stop):
```
=> Segment 935642 exceeded its duration while downloading: elapsed=22.44s > dur=10.43s; buf=-0.5s (<0). Stopping...
```

Repeated negative entries (progress then threshold):
```
=> Negative buffer event #1 in current 60s window
=> Negative buffer event #2 in current 60s window
=> Buffer entered negative state 3 times within 60s window. Stopping...
```

Gap rule (buffer deficit > last segment):
```
=> Buffer deficit exceeded last segment length: |buf|=5.01s > last_seg=5.00s. Stopping...
```

Continuous negative longer than threshold:
```
=> Buffer stayed negative for more than 15.00s. Stopping...
```

Non-smart mode “no new segments”:
```
=> No new segments in playlist for more than 15.00s. Stopping...
```

Window expiry without threshold (diagnostic):
```
=> Negative buffer window expired (60s) without threshold; counter reset.
```

---

## Why “elapsed” and “time” Match Conceptually Now

- “elapsed” in stall warnings and `time=` in completion logs share the same monotonic reference (HTTP request start).
- A warning can appear before completion — so its `elapsed` may be slightly smaller than the final `time=...`. Decisions still use the same base timing you see later.

---

## Practical Interpretation Tips

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| Frequent rule (1) stops | CDN/throttling causing long segment fetches | Inspect network, consider reducing bitrate or checking server health |
| Rule (2) triggers often | Intermittent bursts of inactivity causing repeated buffer collapses | Increase buffer target or investigate upstream stability |
| Rule (3) triggers | Sustained lag after previously healthy buffer | Check segment production delays / ingest latency |
| Rule (4) triggers | Continuous starvation (no full recovery) | Validate feed liveness; maybe switch to fallback source |
| “No new segments” in non-smart | Playlist really stalled or ended | Confirm source availability or intentional end |

---

## Configuration Levers (Most Relevant)
- `hls-playlist-reload-time=smart` — enable smart logic.
- `hls-segment-queue-threshold` — enables gap & continuous-negative timing (and tertiary smart rule).
- `live-buffer-mult` / `vod-buffer-mult` — target cushion sizing (affects buffer dynamics).
- `vod-start`, `vod-process`, `vod-queue-step` — batching behavior in Local VOD.

---

## Summary

Early-stop rules prevent pointless waiting under real stalls or silent playlist freezes. The four smart rules escalate from immediate stall detection to pattern-based instability tracking. Non-smart modes retain a simpler “no new segments” cutoff. Use logs to understand which rule fired and adjust upstream or tuning parameters accordingly.

Enjoy more predictable shutdown behavior and clearer diagnostics.