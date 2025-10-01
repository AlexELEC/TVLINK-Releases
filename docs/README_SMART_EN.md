# Smart Live & Local VOD Modes and Gap/Threshold Behavior (User Guide)

This guide explains two enhanced playback control modes (Smart Live and Local VOD) and the gap/threshold shutdown logic, plus two connection/closing features. It is written for practical operators (no deep math). You will learn:
- Why these modes exist and the benefits.
- How adaptive timing and buffering decisions are made.
- What log messages mean (so you can reason about behavior).
- When and why the stream stops early (protective thresholds).
- What “Close segment connection” and “Async close” options do.

---

## 1. Smart Live Mode

### 1.1. Purpose & Benefits
Traditional HLS polling uses fixed or simplistic intervals (e.g. targetduration). Under unstable networks or irregular segment production, this can cause:
- Over-polling (wasted requests).
- Under-polling (buffer underruns / freezes).
- Slow recovery after hiccups.

Smart Live mode (enabled by setting `hls-playlist-reload-time=smart`) adapts playlist reload timing based on how much real playback time you have already downloaded versus real elapsed wall time (“buffer ahead”). It:
- Speeds up polling when the buffer is low (reducing freezes).
- Slows down polling when the buffer is comfortably filled (reducing load).
- Smooths transitions using short “grace” waits to include just-finished segments before recalculating.
- Detects stalled downloads or persistent negative buffer conditions and stops early instead of hanging indefinitely.

### 1.2. Core Concepts (Plain Language)
- “Buffer ahead” = how many seconds of playable media have been fully downloaded but not yet “consumed” by real time.
- A “capacity target” is computed internally from recent average segment duration and a multiplier (`hls-live-buffer-mult`). Think: “desired safe cushion”.
- Mode classification:
  - GROWTH: buffer is low → poll sooner.
  - NORMAL: buffer in comfortable range → poll at about last segment duration.
  - SLOW: buffer is high/excessive → gently lengthen delay (but not too much).
- Short “grace waits” (up to ~1s, bounded) are inserted if segments are just about to finish downloading—this prevents premature reload before the new segment appears.

You do NOT have to tune formulas: just adjust broad knobs if needed (`live-buffer-mult` for target size, or revert to legacy modes by changing `hls-playlist-reload-time`).

### 1.3. Important Options (Live)
- `hls-playlist-reload-time=smart` — activates Smart Live logic.
- `hls-live-buffer-mult` — multiplier for target buffer (default ~3× average segment length).
- `hls-segment-queue-threshold` — gap timing factor for non-smart modes (also reused by one tertiary smart rule).
- `hls-live-edge` — initial starting edge (how far from the live tip you begin).
- `hls-live-restart` — force starting from the beginning if applicable.

### 1.4. Key Protective Conditions (Smart)
Smart mode can stop early (rather than hanging forever) when:
1. A segment download exceeds its own duration AND the computed buffer goes negative.
2. The buffer becomes negative multiple times in a sliding time window (default: 3 negative entries within 60s).
3. The negative deficit exceeds roughly the size of one segment (indicates drift / loss).
4. The buffer stays continuously negative longer than a derived timing threshold (re-uses gap threshold logic).

These are *safety exits*, not failures: they avoid futile waiting under real stalls or provider gaps.

### 1.5. Typical Debug Log Lines (Smart Live)
You will see lines like:
- `- Adding segment N to queue` — a segment was scheduled for download.
- `+ Segment N complete (time=...ms retries=R)` — fully downloaded (R = partial retry count).
- `<< Planning Reload [smart] in X.XXs ... mode=growth|normal|slow` — next playlist poll timing decision.
- `>> Reloading Playlist [smart]: base=... planned=... | waited=...` — actual reload execution.
- `LARGE JUMP: skipped ... segments` — sequence gap detected; internal stats reset.
- `Negative buffer event #K` — buffer just dropped below zero again in current window.
- `Segment N exceeded its duration while downloading ... Stopping...` — stall exit.
- `Buffer deficit exceeded last segment length ... Stopping...`
- `Buffer stayed negative for more than Ts. Stopping...`

Use these to verify mode changes (growth/slow), pre-grace waits, or early-stop reasons.

### 1.6. Partial Read Resilience
If a segment download is truncated (network hiccup), you may see:
- `Partial/incomplete segment N ... retrying...`
The system retries (resuming with byte ranges if possible) up to a configured limit (default 2 extra attempts). Successful completion resets failure counters.

---

## 2. Local VOD Mode

### 2.1. Purpose & Benefits
Some “VOD-like” playlists behave like rolling time-shift windows (catch-up / rewind) rather than infinite live streams. Local VOD mode activates automatically when the playlist signals VOD or an `#EXT-X-ENDLIST` (or recognized catch-up scheme). It:
- Enqueues segments in **batches** from the current window.
- Adapts pacing based on actual download speed and buffer fill.
- Advances to a new window URL when current content ends (for time-shift/catch-up sources).
- Avoids reloading the playlist for every segment; instead it emits segments then waits adaptively.

### 2.2. Window & Catch-Up Schemes
Detected from original (anchor) URL:
- “shift” query (`utc=` or `lutc=`),
- “append” style (`offset=` or `utcstart=`),
- “flussonic” path pattern (`-timeshift_rel-`),
- Or plain VOD (no modification needed).
When all segments of a window are consumed, a new URL is constructed (if supported) and the next window is fetched.

### 2.3. Batch Enqueue Parameters
- “VOD Limit segments on startup” (vod-start) — initial burst count (first window).
- “VOD Limit segments in progress” (vod-process) — initial burst on subsequent windows.
- “VOD segments queue Step” (vod-queue-step) — number of segments per subsequent enqueue cycle after burst is consumed.
This reduces delays at startup while smoothing load later.

### 2.4. Adaptive Wait (Local VOD)
Between batches (or single-step enqueues) the system:
- Maintains a **robust average** segment duration (median + a high-percentile blend).
- Tracks a smoothed “fill” fraction (buffer ahead vs target).
- Maintains an EMA of download-time / segment-duration (fetch ratio) to detect slowing network.
- Enters a “steady mode” temporarily if a streak of slow fetches occurs, stabilizing pacing.
- Soft-throttles if buffer ahead exceeds a high watermark (pauses before next enqueue).
All of this decreases API/query load and reduces bursty network usage.

### 2.5. Protective Logic (Local VOD)
Local VOD inherits smart-like early-stop checks (negative buffer rules, stalled segments). Additional:
- Empty window protection: if multiple successive window reloads are empty (config limit), it stops.
- Numbering reset: if the new window starts over without forward segments, it logs and restarts from beginning of that window.

### 2.6. Typical Debug Log Lines (Local VOD)
Look for:
- `<< Local VOD: next enqueue adaptive in X.XXs (seg=... buf=.../cap=...)`
- `VOD local: advancing window url -> ...`
- `VOD local: numbering reset/no forward segments...`
- `VOD local: empty window after advance, closing stream.`
- Same threshold warnings as Smart (negative buffer, segment exceeded duration, etc.)

---

## 3. Gap Threshold & General Early-Stop Logic

### 3.1. Purpose & Benefits
In non-smart modes (segment/targetduration/default), endlessly polling when no new segments appear wastes time and resources. The gap threshold provides a *maximum “silence” time*—if no new segments arrive within that computed window, the stream stops gracefully. In smart mode, a similar timing value is reused for one of the tertiary negative-buffer rules.

### 3.2. How Gap Threshold Is Calculated (Plain)
If `hls-segment-queue-threshold` is a positive factor:
1. Determine a base duration:
   - Take the maximum of:
     - A minimum floor (5 seconds)
     - Playlist target duration (if set)
     - Average of last up to 3 segment durations (if available; else the last one)
2. Multiply base by the factor → threshold seconds.
3. If no new segment is queued before the current time exceeds last-enqueue-time + threshold → stop with a warning.

Example (conceptual):
- Floor = 5s, target = 4s, recent average = 6s → base = 6s
- Factor = 1.5 → threshold = 9s
- If no new segment in 9s → “No new segments... Stopping...”

### 3.3. Smart Mode Differences
Smart mode does NOT rely on this for routine polling (it has its buffer-based planner). However:
- The *same threshold duration* is reused as a tertiary condition to abort if the buffer stays negative continuously longer than that threshold.
- Other smart-specific rules (stalled segment > its duration, multiple negative events, etc.) may trigger first.

### 3.4. Log Lines (Gap)
- `=> No new segments in playlist for more than X.XXs. Stopping...`
- In smart mode you may instead see negative buffer warnings or specific stall reasons before a basic gap timeout occurs.

---

## 4. Connection & Closing Behavior

### 4.1. Close Segment Connection (`hls-segment-conn-close`)
Purpose:
- Forces adding `Connection: close` to segment/key/playlist HTTP requests.
- Beneficial if the upstream server or an intermediate proxy mishandles persistent (keep-alive) connections, causing stalls or partial reads.

Pros:
- Reduces risk of “sticking” connections / idle TCP issues.
Cons:
- Increases connection setup overhead (TLS handshakes, etc.).
Use it only if you experience unexplained stalls with defaults.

### 4.2. Async Close (`hls-close-async`)
Purpose:
- Speeds up application shutdown when a stream is being closed.
- Offloads the stream teardown (aborting writer, cancelling futures, GC cleanup) to a background thread.
Benefits:
- UI / controlling process becomes responsive faster.
- Avoids blocking on resource cleanup, especially with slow network sockets or large buffers.

Behavior:
- On close, a background thread triggers a “fast abort” in the writer:
  - Cancels pending futures.
  - Stops starting new HTTP requests.
  - Clears caches (keys/maps/futures) to free memory promptly.
- Log noise about thread endings is filtered (so normal users are not spammed).

Choose async close if you manage many parallel HLS streams or need rapid session recycling. Enabled by default.

---

## 5. Interpreting Key Log Patterns (Cheat Sheet)

| Log Fragment (Starts With) | Meaning / Actionable Insight |
|----------------------------|------------------------------|
| `<< Planning Reload [smart]... mode=growth` | Buffer low; faster reload planned. |
| `<< Planning Reload [smart]... mode=slow` | Buffer high; slower reload to reduce load. |
| `+ Segment N complete (time=... retries=R)` | Segment fully downloaded; R signals partial-retry resilience used. |
| `Partial/incomplete segment N ... retrying...` | Network hiccup; automatic resume or refetch. |
| `Segment N exceeded its duration ... Stopping...` | Protection triggered: stall/slow path. |
| `Negative buffer event #K` | Buffer fell below zero again (risk of freeze). |
| `Buffer deficit exceeded last segment length ... Stopping...` | Chronic lag; abort to avoid hang. |
| `=> No new segments ... Stopping...` | Gap threshold exceeded in non-smart mode. |
| `Local VOD: next enqueue adaptive in X.XXs` | Adaptive pacing between segment batches. |
| `LARGE JUMP: skipped ... segments` | Playlist numbering advanced sharply (possible provider gap). |
| `Filtering out segments and pausing stream output` | Certain segments were skipped (e.g. name-based filtering), output paused. |
| `Retry fetch failed for segment N` | Final refetch attempt failed; segment abandoned. |
| `VOD local: advancing window url -> ...` | Catch-up shift to next window. |
| `VOD local: empty window after advance, closing stream.` | No new content; exit. |

---

## 6. Practical Tips

- If you see frequent early stops with smart mode: check upstream stability; consider falling back to `targetduration` mode for diagnosis.
- If logs show many partial retries: network path may be unstable; enabling `hls-segment-conn-close` can sometimes help.
- For high-latency environments, consider increasing `live-buffer-mult` to grow a larger cushion.
- For aggressive, low-latency live following, reduce `live-buffer-mult` (but risk more growth cycles).
- Local VOD pacing knobs (`vod-start`, `vod-process`, `vod-queue-step`) let you control initial burst vs sustained bandwidth smoothing.

---

## 7. Summary

Smart Live and Local VOD modes dynamically adapt to real conditions to balance responsiveness, stability, and resource usage. Gap thresholds and protective early-stop rules prevent indefinite hanging on broken or stalled feeds. Connection and async-close options offer robustness and fast teardown when needed. Use the logs as a transparent window into decisions—almost every adaptive step prints a concise line so you can tune or trust behavior.

Enjoy smoother streaming operations with fewer mysteries and clearer diagnostics.