# HLS Smart mode and its components

This guide is for those who want to understand: “What are these dozens of strange setting names, and which of them should I touch?”

## 1. How to enable everything “smart”
There is a setting: **HLS playlist reload time**.  
If you choose **smart**, the extended “smart” mode is enabled.  
It decides on its own: when to re-fetch the playlist, how quickly to catch up to a live stream, what to do during pauses, how to react to spikes, what to do with VOD (recording), when to stop, and how to temporarily “loop” the last segment so the picture doesn’t freeze.

If you choose:
- segment — we simply wait as long as the last segment duration (#EXTINF tag).
- targetduration — we wait the “target” duration specified for the whole playlist (not individual segments, #EXT-X-TARGETDURATION tag).
- smart — segment length is used as a base, all the “smart” logic below is active.

If neither segment duration nor targetduration is specified in the playlist – a static 6 s interval is used.

## 2. What Smart includes
| Part | Short | Works outside smart? |
|------|-------|----------------------|
| Step3 (three-stage waiting) | Frequent checks when changing, slower when quiet | No |
| “Lag / headroom” (drift) | If we fall behind — speed up checks; if we have buffer — relax | No |
| Batch (segment bursts) | Adjust tempo when many arrive at once | No |
| Auto VOD + acceleration after discontinuities | Different rhythm for recordings + fast checks after breaks | No |
| Replay | Loop last chunk so video doesn’t “freeze” | Almost not |
| Random offset (jitter) | Requests aren’t strictly in sync | No |
| Gap (long silence ⇒ stop) | Stop if no new segments for too long | Yes (simple version) |
| Gap guard “don’t stop too early” | Avoid premature stopping if headroom remains | No |

In other modes only the basic Gap stop timer really remains. Everything else is disabled.

## 3. How to read the descriptions
For each setting below:
- What it does.
- If increased.
- If decreased.
- When to adjust.
- Recommended range.
- Whether it only works in smart mode.

Don’t be scared by the word “segment”: it’s one small video file.  
If it says “fraction of segment duration 0.5” and the segment is 6 seconds — that means about 3 seconds of waiting.
The variables below are located in file “tvlink/libs/streamlink/stream/hls/hls.py”.

---

### 3.1. Core Smart timers

#### reload_early_offset_s (default 0.02)  [smart only]
What: How much earlier than normal we may request the playlist, before the expected “new segment” time runs out.  
Increase: We “peek ahead” more often, more requests.  
Decrease (0): May notice a new segment slightly later.  
When: Rarely.  
Recommendation: 0.0–0.25 (commonly 0.01–0.05).  
Tip: Stream stable? Set to 0.0.

#### reload_jitter_ms (30)  [smart only]
What: Small random “jitter” (milliseconds) in request time to avoid perfectly even cadence.  
Increase: Slightly more varied rhythm.  
Decrease (0): Perfectly identical intervals.  
When: Almost never.  
Recommendation: 10–80.  

---

### 3.2. Step3 — three-stage waiting (all smart only)

(Below “first / second / third and further unchanged” refers to how many consecutive times we’ve seen no new segments added.)

#### step3_changed_ratio (1.00)
What: Max fraction (of segment duration) we wait after a new one appears.  
Increase: Fewer requests, may see following segment a bit later.  
Decrease: More requests, faster reaction.  
Recommendation: 0.8–1.2.  
Touch: Only if you need “even faster”.

#### step3_u1_ratio (0.50)
What: How long to wait (fraction of segment duration) after the first “silence”.  
Increase: Save requests.  
Decrease: Catch resumption faster.  
Recommendation: 0.4–0.7.

#### step3_u2_ratio (0.50)
Same, but for the second consecutive “silence”.  
Recommendation: 0.4–0.7.

#### step3_min_ratio (0.25)
What: Minimum fraction waiting on the third and all subsequent “no new segments”.  
Increase: Lazy checking.  
Decrease: Very frequent requests.  
Recommendation: 0.20–0.35.

#### step3_cv_window (8)
What: How many recent segments we inspect to decide if durations are “stable or jumping”.  
Increase: Smoother but slower to adapt.  
Decrease: Faster but twitchy.  
Recommendation: 5–12.  
When: Rarely.

#### step3_cv_high (0.25)
What: Upper threshold: “duration varies a lot”. Then we do NOT shrink waiting too much.  
Increase: Less often treat stream as chaotic.  
Decrease: More often “cautious”.  
Recommendation: 0.20–0.35.

#### step3_cv_low (0.08)
What: Lower threshold: “very stable”. Then we can slightly lengthen pause (save requests).  
Increase: Less often mark “stable”.  
Decrease: More often.  
Recommendation: 0.05–0.12.

#### step3_cv_floor_ratio (0.40)
What: Floor — we don’t go below this fraction if stream “jumps”.  
Increase: More cautious (no aggressive speeding up).  
Decrease: Allow stronger acceleration.  
Recommendation: 0.35–0.5.

---

### 3.3. Lag and headroom (catch up fast / don’t overshoot)  [all smart only]

(Imagine: we have played N seconds of video. How much real time has passed? If more — we “lag”. If less — we have headroom.)

#### step3_drift_target_factor (1.1)
What: How much forward headroom (factor) we consider normal.  
Increase: More buffer → safer, can notice new chunk a bit later.  
Decrease: Less buffer → faster catch-up, less safety.  
Recommendation: 1.05–1.3.  
Tip: Do not change without reason.

#### step3_drift_aggr_reduce (0.05)
What: Additional wait reduction when lag is already large.  
Increase: More aggressive catching up.  
Decrease: Calmer.  
Recommendation: 0.03–0.08.

#### step3_drift_ahead_increase (0.02)
What: Slight stretching of wait if headroom is already good.  
Increase: May end up slightly behind later.  
Decrease: Relax less.  
Recommendation: 0.01–0.04.

#### step3_drift_lvl1 / step3_drift_lvl2 / step3_drift_lvl3 (0.60 / 1.00 / 1.40)
What: Three levels of “how far behind” (small / noticeable / large).  
Decrease values: Acceleration triggers earlier.  
Usually keep — relative spacing matters.

#### step3_drift_reduce1 (0.10), step3_drift_reduce2 (0.20)
What: How much we cut waiting at levels 1 and 2.  
Increase: Faster catch-up.  
Decrease: Gentler.  
Recommendation: 0.08–0.15 and 0.15–0.30.

#### step3_drift_min_ratio (0.35)
What: How low waiting may drop in “catastrophic” lag.  
Increase: Less aggressive.  
Decrease: Many requests.  
Recommendation: 0.30–0.45.

#### step3_drift_projection_headroom_factor (0.30)
What: Headroom we don’t trim when projecting forward “can we speed more”.  
Increase: More cautious.  
Decrease: Cut harder.  
Recommendation: 0.2–0.4.

#### step3_drift_projection_min_gain_ratio (0.05)
What: Minimal meaningful gain (rough seconds saved) before projection-based reduction is applied.  
Increase: Use projection less.  
Decrease: Use more.  
Recommendation: 0.03–0.08.

---

### 3.4. Batch publication (burst)  [smart only]

#### step3_batch_threshold (3)
What: How many new consecutive segments qualify as a “batch”.  
Increase: Special mode triggers less often.  
Decrease: More often.  
Recommendation: 2–4.

#### step3_batch_follow_ratio (0.60)
What: How fast we check after a batch (smaller = faster).  
Increase: Slower reaction to next wave.  
Decrease: Faster, but more requests.  
Recommendation: 0.5–0.75.

#### step3_batch_follow_cap_s (3.5)
What: Max seconds of waiting after a batch.  
Increase: Could be slower on long segments.  
Decrease: More requests with long segments.  
Recommendation: 2.0–5.0.

---

### 3.5. VOD (recording) — smart logic  [smart only]

#### vod_auto_discont_boost_reloads (2)
What: Number of fast checks after a sharp discontinuity to “recover” sooner.  
Increase: Faster recovery, more requests.  
Recommendation: 2–4.

#### vod_reload_publish_slack_s (0.30)
What: Slack seconds of calm waiting if no new segments.  
Increase: Less nervous.  
Decrease: More frequent checks.  
Recommendation: 0.2–0.6.

#### vod_reload_idle_min (1.00)
What: Minimum pause between checks in calm VOD.  
Increase: Fewer requests.  
Decrease: More requests.  
Recommendation: 0.8–2.0.

---

### 3.6. Gap — stopping on “silence”

Works in any mode, but “smart” protections (below) fully only in smart.

#### gap_stop_min_absolute_s (10.0)
What: Minimum seconds without new segments before stopping.  
Increase: Wait longer.  
Decrease: Risk early stop.  
Recommendation: 8–20.

#### gap_stop_min_growth_samples (3)
What: How many segment duration samples needed for confident estimate.  
Increase: More reliable, slower to “understand”.  
Decrease: Faster, rougher.  
Recommendation: 3–5.

#### gap_small_segment_floor_s (2.0)
What: If segments are very short — don’t count less than this in calculations.  
Increase: Less sensitive to ultra-short.  
Decrease: More sensitive.  
Recommendation: 1.5–3.0.

#### gap_drift_guard_trigger (-2.0)  [smart only]
What: “Don’t stop early” guard. If current forward headroom is still about ≥2 sec (roughly), we do NOT consider it a real dead stop.  
Increase (e.g. -1): Weaken guard — stop earlier.  
Decrease (e.g. -3): More patient.  
Recommendation: -3.0 … -1.0.

---

### 3.7. Segment replay (Replay)  [mostly smart only]

#### event_replay_buffer_min_s (1.0)
What: If forward buffer is below this — allow temporary replay of last segment to avoid a frozen frame.  
Increase: Replay triggers earlier.  
Decrease: Later.  
Recommendation: 0.8–1.5.

#### event_segment_arrival_risk_threshold_s (0.0)
What: Threshold “freeze risk soon”. 0 = automatically half the segment length.  
Increase: Warning later.  
Decrease (>0): Earlier.  
Recommendation: 0 or 0.5–2.0.

#### replay_cooldown_limit (3)
What: How many consecutive replays allowed without new real segments.  
Increase: Longer simulated motion.  
Decrease: Give up sooner.  
Recommendation: 2–4.

---

## 4. Quick tips (what to change if)

| I want | What to do |
|--------|------------|
| See new segments as fast as possible | Lower step3_changed_ratio to ~0.9, u1/u2 to ~0.45 |
| Fewer requests | Raise u1/u2 to ~0.6–0.65, increase changed_ratio to ~1.05 |
| Calm a jittery stream | Raise step3_cv_floor_ratio to ~0.45 |
| Catch up to live faster | Raise drift_reduce1/2 (0.12 / 0.25), lower drift_min_ratio to ~0.30 |
| Fewer false stops | gap_stop_min_absolute_s → 15; gap_drift_guard_trigger → -2.5 |
| Softer replay | replay_cooldown_limit → 2; event_replay_buffer_min_s → 1.2 |

(If unsure — change nothing.)

---

## 5. FAQ

**Do I need to know HLS to tune this?**  
No. Just the idea: we periodically refresh the list of segments.

**Can I leave everything as is?**  
Yes. This is a safe universal set.

**Why so many settings?**  
To balance between “show it fast” and “don’t overload the server”.

**If I broke something?**  
Restore defaults (from this file: “tvlink/libs/streamlink/stream/hls/hls.py”).

**Is Replay a “trick”?**  
No, we simply re-show the already downloaded segment so your screen doesn’t freeze.

---

## 6. Mini sets (ready combinations)

Stable stream (quieter):
```
step3_u1_ratio = 0.55
step3_u2_ratio = 0.55
reload_early_offset_s = 0.0
```

Lagging / want to catch up:
```
step3_drift_reduce1 = 0.12
step3_drift_reduce2 = 0.25
step3_drift_min_ratio = 0.30
```

False “stops” too early:
```
gap_stop_min_absolute_s = 15
gap_drift_guard_trigger = -2.5
```

---

## 7. Summary
1. Choose smart mode — get all capabilities.  
2. Not sure — leave defaults.  
3. Change only if you understand the goal.  

Enjoy watching!