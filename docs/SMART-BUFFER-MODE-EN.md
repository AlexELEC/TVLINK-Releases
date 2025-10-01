# Smart buffer-based pacing (Live smart) and Local VOD

This document explains in simple terms how the player now decides when to:
- speed up,
- keep normal pace,
- or slow down

for both:
- Live “smart” reload planning, and
- Local VOD segment enqueueing.

The logic in both modes is aligned and logs look similar.

---

## Key ideas in plain words

- “Segment length” (seg) is the duration of one media piece. It’s your natural “metronome”.
- “Buffer ahead” (buf) is how much already-delivered playtime you have in stock (in seconds) compared to real time.
- “Capacity” (cap) is the target buffer we aim to keep. It’s based on a typical segment duration multiplied by a coefficient (different for Live and VOD).
- “Band” is a small safety margin around that typical segment duration (used to decide modes more robustly).

We constantly look at:
- How much buffer we have (buf),
- What our target is (cap),
- And how long one typical segment is (seg, and band).

From that we pick a mode:
- growth — speed up if the buffer is too low
- normal — stay at the regular pace
- slow — slow down if the buffer is too high

and schedule the next wait accordingly.

---

## Shared terminology

- seg: baseline segment length (seconds)
  - Live smart: last segment’s duration.
  - Local VOD: last segment’s duration (with light outlier protection).
- band: typical segment length used as safety band
  - Live smart: mean of recent durations.
  - Local VOD: “robust” mean of recent durations (median blended with a high percentile), or a simple mean if not enough history.
- buf: playable buffer ahead (seconds). It’s delivered media time minus real time since start. Can be > cap.
- cap: target buffer (seconds)
  - Live: mean segment duration × live_buffer_mult
  - VOD: robust mean × vod_buffer_mult
- MIN wait floor: we never wait less than 1.00s to avoid excessive polling.

---

## Modes: when do we speed up, keep normal, or slow down?

We use the same rule for both Live smart and Local VOD:

- slow — if buffer is “well above” the target:
  - buf > cap + band (and not during the very first planning moments)
- growth — if buffer is “too low”:
  - buf < seg OR buf < cap − band (and not during the very first planning moments)
- normal — otherwise

Startup grace: at the very beginning, we treat the first plan as normal to avoid overreacting.

Why two checks for growth?
- buf < seg — if your buffer is smaller than one typical segment, it’s too skinny → speed up.
- buf < cap − band — even if buffer is a bit larger than a segment, but still noticeably below the target (cap), we also speed up.

---

## How long to wait next (the pacing)

Same formula style for both modes:

- normal: wait = seg
- growth (speed up when low on buffer):
  - wait scales with fill: wait ≈ seg × clamp(buf/cap, 0..1), but not less than 1s
  - The lower the buffer vs capacity, the shorter the wait
- slow (slow down when buffer is high):
  - wait ≈ seg × max(1, buf/cap), capped by cap, and not less than 1s
  - The higher the buffer vs capacity, the longer we wait, but not longer than cap

In other words:
- If you’re low on buffer, we poll sooner than one segment.
- If you’re overfilled, we poll later than one segment (but no more than cap).
- If you’re “just right”, we poll exactly one segment later.

---

## Pre-grace: tiny waits to catch imminent completions

To plan accurately, just before we decide the next wait, we do a short bounded “pre-grace”:
- If a segment just finished or is about to finish (we have an internal signal from the downloader), we briefly wait (up to a small limit) so the buffer count reflects those just-finished bytes.
- This makes mode decisions and wait times more stable and accurate.
- The bound can stretch slightly using an estimate of download speed, but it’s still capped (by plan_completion_grace_max).

This applies to both:
- Live smart (playlist reload planning),
- Local VOD (segment enqueue planning).

---

## What you’ll see in logs

We show:
- the next wait time we plan,
- seg (baseline segment length),
- buf/cap (current buffer and target),
- thr (throttle) which is the difference from seg and can be positive or negative,
- mode (growth/normal/slow).

Examples:

Local VOD (enqueue pacing):
- “growth” with negative throttle (speeding up)
  - << Local VOD: next enqueue adaptive in 0.85s (seg=6.00s buf=2.0s/18.0s) thr=-5.15s fill=0.11 mode=growth)
- “normal” (exactly segment pace, thr=+0.00s)
  - << Local VOD: next enqueue adaptive in 6.00s (seg=6.00s buf=10.5s/18.0s) thr=+0.00s fill=0.58 mode=normal)
- “slow” with positive throttle (slowing down)
  - << Local VOD: next enqueue adaptive in 10.50s (seg=6.00s buf=25.0s/18.0s) thr=+4.50s fill=1.39 mode=slow)

Live smart (playlist reload planning):
- << Planning Reload [smart] in 1.20s: (seg=6.00s buf=2.0s/12.0s) thr=-4.80s fill=0.17 mode=growth)
- << Planning Reload [smart] in 6.00s: (seg=6.00s buf=6.1s/12.0s) thr=+0.00s fill=0.51 mode=normal)
- << Planning Reload [smart] in 10.00s: (seg=6.00s buf=20.0s/12.0s) thr=+4.00s fill=1.67 mode=slow)

Notes:
- thr shows the net change from “plain segment pace”:
  - thr < 0 means we are faster than one segment (speed up),
  - thr > 0 means we are slower (throttle).

---

## Configuration knobs you might care about

- “HLS Live buffer multiplier” (Live smart): target buffer is mean_segment × live_buffer_mult
  - Typical default: 3.0 (around three segments ahead)
- “HLS VOD buffer multiplier” (Local VOD): target buffer is robust_mean × vod_buffer_mult
  - Typical default: 3.0 (around three “typical” segments ahead)
- plan_completion_grace_max: upper bound for short “pre-grace” waits when we expect an imminent completion
- MIN wait floor: 1.00s (hardcoded), to avoid too frequent wake-ups
- Buffer smoothing windows:
  - Live: uses recent mean segment duration for band/cap calculations
  - VOD: uses “robust” mean (blends median with a high-percentile of recent durations) to avoid being fooled by spikes

Advanced VOD add-ons (work automatically):
- Slow-series detector: if actual downloads are consistently slower than segment length, we may enter a short “steady mode” to keep pace stable for several segments.
- Lightweight outlier protection: if one segment is an outlier, we avoid overreacting.

---

## FAQ

Q: Why can buf be larger than cap?
- It’s fine to go over target for a while. In “slow” mode we allow it to drain gently, never exceeding a maximum wait of cap seconds.

Q: Why does the wait sometimes include small extra pauses?
- That’s the “pre-grace” to catch segments that are just about to finish, so our buffer snapshot is fresh before deciding.

Q: What if network is shaky?
- The system adapts: when segments take longer to download, the buffer signal and slow-series detector naturally steer pacing to stay smooth.

---

## One-liner summary

Both Live smart and Local VOD now use the same, simple rule:
- Low buffer → speed up
- OK buffer → normal pace
- High buffer → slow down
with clear logs showing how far we deviate from “one segment per step” (thr=±...s).
