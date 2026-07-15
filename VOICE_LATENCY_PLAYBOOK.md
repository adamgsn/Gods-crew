# John — Expert Latency Playbook (Telephony Voice Agent)
### Founder/architect-grade. Two stacks: (A) Vapi cascade, (B) Grok native S2S.

> Provenance note (2026-06-12): the web-research/citation stage of the generating
> workflow failed mid-run due to an API outage. The technique set, structure, and
> stack-specific actions are synthesized from an architecture-completeness critic +
> domain expertise + this session's *earlier* web-verified findings (TTFT governs
> perceived voice latency; reasoning models are a TTFT trap; Groq 8b-instant is the
> speed-optimal usable model). **Treat the ms figures as grounded estimates, not
> fresh measurements.** Primary references to chase for citations: LiveKit
> "Understand and Improve Agent Latency", Pipecat docs (Cartesia input streaming,
> preemptive generation), Twilio core-latency guide, AssemblyAI turn-detection,
> arXiv 2410.00037 (Moshi), vLLM/SGLang prefix caching.

> Scope: this goes **beyond** the table-stakes already shipped (8b-instant, fast
> endpointing, pre-cached opener, ambiance, capped tokens, no BYOK hop). Everything
> below is the next 100–600 ms that separates "fast" from "uncanny-fast", plus the
> two flagged pains: per-turn latency attribution and self-repetition.

---

## 1. Mental model — the time-to-first-audio (TTFA) budget

The only number a caller feels is **TTFA: end of their speech → first phoneme of John's reply leaving the Twilio media stream.** Everything else is bookkeeping.

```
Caller stops talking
   │
   ├─[A] Endpoint decision delay   ← VAD/smart-endpointing waits to be SURE they're done
   │        (often the SINGLE biggest controllable chunk)
   ├─[B] STT finalization TTFB     ← final transcript emitted (partials are "free")
   ├─[C] LLM TTFT                   ← time to FIRST token (not full completion)
   ├─[D] TTS TTFB                   ← time to FIRST audio chunk (not full synthesis)
   ├─[E] Network/RTC + jitter buffer← WS hops, Twilio media, playout buffering
   │
First audio reaches caller's ear  → TTFA
```

### Typical per-stage cost (telephony, well-tuned cascade)

| Stage | What it is | Good | Sloppy | Notes |
|---|---|---|---|---|
| **A. Endpoint delay** | silence wait before "user done" fires | 250–400 ms | 800–1200 ms | **Usually the #1 lever.** A deliberate wait, not compute. |
| **B. STT final TTFB** | final transcript after endpoint | 50–150 ms | 300 ms+ | nova-3 streaming is fast; finalization is the cost, partials free |
| **C. LLM TTFT** | first token | 150–350 ms | 600 ms+ | 8b-instant on Groq is excellent; prompt size & cold cache dominate |
| **D. TTS TTFB** | first audio chunk | 80–250 ms | 500 ms+ | speech-02-turbo / Sonic stream; first-chunk is what matters |
| **E. Network/RTC** | WS + PSTN + jitter buffer | 80–200 ms | 400 ms+ | geo distance, extra proxy hops, jitter-buffer depth |
| **TOTAL TTFA** | | **~700–1100 ms** | **2.5–4 s** | sub-800 ms feels human; >1.5 s feels like a bot |

### ACTUAL vs PERCEIVED latency (the distinction that matters most)

You can win 300+ ms **without making anything faster** by attacking *perception*:

- **TTFA is what's felt, not total response time.** Once audio starts, the rest streams under the caller's own listening time. Optimize first-chunk, not total synthesis.
- **Filler/acknowledgment audio** ("Sure—", "Right,") played *the instant the endpoint fires* masks the entire LLM+TTS pipeline behind it. Perceived latency collapses to endpoint+STT only.
- A pipeline that's actually 200 ms faster but has a 600 ms silent gap before audio feels **slower** than a slower pipeline that grunts "mm-" immediately. Humans measure the silence, not the compute.

**Rule:** measure actual ms per stage (§5) but **spend effort where it changes the silence the caller hears.**

---

## 2. Ranked levers — biggest real ms first

| # | Lever | Mechanism | Realistic impact | Effort |
|---|---|---|---|---|
| 1 | **Semantic / dynamic endpointing** | End turn earlier when transcript is *semantically complete*; extend only mid-sentence | **150–500 ms** | M |
| 2 | **Instant filler on endpoint-fire** | Play "Sure," at endpoint, before LLM starts — masks C+D entirely | **200–600 ms perceived** | M |
| 3 | **LLM→TTS streaming at first clause** | Start TTS on first sentence/comma, don't wait for full completion | **150–400 ms** | M |
| 4 | **Prompt-prefix / KV cache warm** | Static system+pitch+FAQ as cached prefix; only turn-delta is fresh | **100–300 ms TTFT** | L–M |
| 5 | **Geo co-location + hop elimination** | STT, LLM, TTS, SIP same region; kill every extra WS/proxy hop | **100–300 ms** | M |
| 6 | **TTS first-chunk / smaller frames** | Tune chunk size for fast first frame; μ-law 8k for PSTN, no resample | **80–200 ms** | L |
| 7 | **Speculative LLM start on partials** | Begin generating on stable partial before final fires | **100–250 ms** | H |
| 8 | **Prompt diet (tokens = TTFT)** | Cut system prompt; FAQ as short retrievable lines not megaprompt | **50–150 ms TTFT** | L |
| 9 | **Jitter-buffer / playout tuning** | Minimum viable jitter buffer on the media path | **50–150 ms** | M |
| 10 | **Warm / pre-opened sockets** | Keep STT/LLM/TTS WS warm; avoid TLS+cold-start on turn 1 | **50–200 ms first turn** | M |

> The top 3 are ~90% of the available win and all attack **perceived** latency. Do them first. Item 7 and the frontier stuff are sub-200 ms each and carry real complexity/repetition risk.

---

## 3. The three tiers

### TABLE STAKES — John already has (don't re-litigate)
- Smallest fast **non-reasoning** model (llama-3.1-8b-instant) — reasoning models are a TTFT disaster.
- Fast LiveKit smart endpointing (~0.3 s wait).
- Pre-cached **instant opener** (first-turn TTFA ≈ 0).
- Office ambiance (masks micro-gaps, kills "are you there?").
- Capped `maxTokens` (180).
- Dropped the BYOK proxy hop (−1 network round trip).

### EXPERT MOVES — do next (highest ROI, low frontier-risk)
1. **Filler/acknowledgment audio on endpoint-fire** — the single best perceived-latency win not yet shipped. Pre-synthesize 5–8 short ack clips ("Sure—", "Right,", "Okay so—", "Mm,"), play one *immediately* when user-done fires, while the LLM is still thinking. The ambiance covers the seam.
2. **Semantic / dynamic endpointing** — short wait (~150–200 ms) when the transcript looks complete ("yes", "I'm not interested", a finished sentence); longer (~600 ms) only when it ends mid-clause. **Biggest real-ms lever.**
3. **Stream LLM→TTS at the first clause boundary.** For the **verbatim airline pitch** and **fixed FAQ lines** go further: these are static — **pre-synthesize them as audio assets** and play the cached audio, skipping LLM+TTS entirely (TTFA ≈ network). This also *guarantees* verbatim delivery and removes a repetition vector.
4. **Prompt-prefix / KV caching** — keep the static system+pitch+FAQ block as a stable cached prefix.
5. **Prompt diet** — move fixed FAQ answers out of the megaprompt into short injected lines keyed by intent.
6. **Geo co-location + warm sockets** — pin STT/LLM/TTS/SIP to one region; keep WS sessions warm.

### FRONTIER BETS — measure twice (§7)
- Full-duplex S2S models (Moshi, Sesame CSM) replacing the cascade.
- Speculative dialogue: generate on partials, commit/discard on final.
- Sub-500 ms unified architectures.
- For John, the Grok worker is *already* the frontier bet — extract its wins, don't open a third architecture.

---

## 4. Action plan by stack

### (A) Vapi cascade
1. **Endpointing.** Move from fixed `waitFunction ~0.3s` to **dynamic/semantic**: short wait (~150–200 ms) on complete transcripts; extend to ~500–700 ms only on mid-clause endings. Tune punctuation/confidence, not a flat timer. #1 real-ms win.
2. **Filler audio.** Fire a short ack at start-of-turn, before LLM TTFT. Clips ~150–300 ms, EQ'd into the office-ambiance bed.
3. **Static-turn shortcut.** Serve **pre-rendered audio** for the verbatim pitch and fixed FAQ lines instead of round-tripping LLM+TTS. Guarantees verbatim + removes repetition risk + near-zero TTFA. (Do NOT touch `firstMessage` — this is pitch/FAQ turns only.)
4. **TTS streaming.** Confirm MiniMax/Cartesia in streaming first-chunk mode; small audio frames; μ-law 8k end-to-end (no needless resample).
5. **Prompt.** Cut to minimum; FAQ lines short injected strings; keep static block stable for prefix caching.
6. **Topology.** Align Deepgram / Groq / TTS / Twilio-SIP regions. Each misaligned provider = a transcontinental WS round trip *per turn*.
7. **maxTokens.** 180 is fine; add a soft 1–2 sentence style instruction so completions *end* sooner (shorter completion → less barge-in-induced restart, a repetition cause).

### (B) Grok native S2S worker
S2S collapses STT+LLM+TTS into one hop — different knobs:
1. **VAD discipline (memory).** Keep `0.5 / 200 ms / 300 ms`. Below that, one-word replies drop — a correctness *and* repetition trigger. Don't chase latency below this floor.
2. **Warm the session during the opener clip (memory).** First reply ~3 s cold. Fire `session.update` + warm the realtime session *while the cached opener plays*; `create_response` ON up front.
3. **Deterministic-content escape hatch.** S2S paraphrases — fatal for the verbatim pitch/FAQ. Detect trigger → **pause realtime model, play pre-cached verbatim clip, hand control back.** Enforces verbatim + prevents re-improvisation/repetition.
4. **Single-responder guarantee.** Native S2S + any cascade fallback = two brains = double replies. Exactly one active responder; never let a fallback speak over the realtime model.
5. **Output gain only via documented path (memory):** `GROK_OUTPUT_GAIN ≤ 1.5` in `realtime_audio_output_node`. No volume API; don't add latency hunting one.
6. **Deploy auth (memory).** `lk` has no cloud auth — pass `--url/--api-key/--api-secret` from `.env` (wss→https) or it says "no projects configured."

---

## 5. Measurement & instrumentation — attribute every millisecond

Upgrade per-turn trigger logging to a **per-turn latency ledger** so every TTFA decomposes into named stages. You can't optimize what you can't attribute.

### One structured event per turn (monotonic timestamps, not wall clock)
```jsonc
{
  "call_id": "...", "turn": 7, "ts": 1234567890,
  "t_user_speech_end":     0,      // VAD/endpoint anchor (t=0)
  "t_endpoint_fired":     310,     // [A] endpoint decision delay
  "t_stt_final":          395,     // [B] STT finalization TTFB
  "t_llm_first_token":    640,     // [C] LLM TTFT
  "t_llm_last_token":    1180,
  "t_tts_first_chunk":    760,     // [D] TTS TTFB
  "t_first_audio_out":    840,     // [E] TTFA  ← THE NUMBER
  "t_filler_audio_out":   330,     // perceived TTFA if filler used
  "stt_provider":"deepgram-nova3","llm":"groq-8b-instant","tts":"minimax-02-turbo",
  "prompt_tokens": 612, "completion_tokens": 48,
  "endpoint_mode":"semantic","filler_played": true,
  "barge_in": false, "response_id":"r_7a", "superseded": false, "retry": false
}
```

### Derived per-stage metrics (track p50 / p95 / p99 — means lie, tails hang up)
- **endpoint_delay** = `t_endpoint_fired − t_user_speech_end`
- **stt_ttfb** = `t_stt_final − t_endpoint_fired`
- **llm_ttft** = `t_llm_first_token − t_stt_final`
- **tts_ttfb** = `t_tts_first_chunk − t_llm_first_token`
- **rtc_out** = `t_first_audio_out − t_tts_first_chunk`
- **TTFA** = `t_first_audio_out − t_user_speech_end`
- **perceived_TTFA** = `t_filler_audio_out − t_user_speech_end`

### How to trace & attribute
- **Single trace/span id per turn** (`call_id + turn`), one span per stage → a slow turn shows *which* stage blew up.
- **Tag every turn type:** `opener | pitch_verbatim | faq_fixed | freeform | transfer | dnc`. Verbatim/FAQ turns should show near-zero LLM/TTS if the audio-cache shortcut is firing — if not, it isn't.
- **Log `prompt_tokens` next to `llm_ttft`** — a rising prompt is a stealth TTFT regression and a repetition risk.
- **p95/p99 by stage, segmented by turn type and region.** A single p99 hides "FAQ great, freeform p99 = 2.4 s".
- **Alert on tail, not mean.** SLOs: TTFA p50 < 900 ms, p95 < 1.4 s. Page on p95 drift.
- **Grok worker:** same ledger, stages collapse — log `t_user_speech_end → t_response_audio_start` as one S2S TTFA + cold-vs-warm flag (proves the opener-warming win).
- **Repetition counter:** increment when a `response_id` is superseded, a retry fires, or two audio streams overlap. Early-warning that the duplicate-speech bug is back.

---

## 6. The "AI repeats itself" problem — root causes & fixes

Repetition is almost never the model "deciding" to repeat — it's a **turn-taking / concurrency / context bug.**

| # | Root cause | Symptom | Fix |
|---|---|---|---|
| 1 | **Double responders** | Same line twice / two voices overlapping | Two paths can speak (S2S + cascade fallback, or function-call + text). **Exactly one active responder** per turn via hard mutex/state flag. Never let a fallback speak over the realtime model. |
| 2 | **Turn-taking race / premature endpoint** | John restarts a line after a "uh-huh" | Endpoint fired on a backchannel. **Semantic endpointing** + treat short acks ("mm","yeah") as non-turn-ending. |
| 3 | **Barge-in restart** | Caller interrupts; John replays from the top | On barge-in **discard the in-flight response, not pause-and-resume.** Cancel TTS stream + LLM generation; start fresh. Resuming a cancelled buffer = "repeats the first half". |
| 4 | **Stale / duplicated context** | Re-asks an answered question / repeats a pitch line | Transcript appended twice, or assistant turn not written back before next turn. **Single source of truth**; write assistant turns to history *before* the next user turn; dedupe appends. |
| 5 | **Retries replaying a committed response** | Reply spoken, hiccup, retry speaks it again | Idempotent delivery: tag each response `response_id`; suppress audio for an already-played id. Never retry after first-audio-out. |
| 6 | **Filler + real response collision** | "Sure, sure, …" | When using filler, instruct the LLM not to re-emit an opener token, or strip leading acks from the completion. Filler + completion = one designed utterance. |
| 7 | **Provider resend / WS reconnect** | Whole turn duplicates after a blip | On STT/TTS WS reconnect, don't replay last final transcript or re-submit last completion. Gate on `response_id`. |
| 8 | **Stage-direction leakage (Grok, memory)** | Model "speaks" instructions | S2S voices anything after a quote / any stage direction. Keep prompt tight, no parentheticals. |

**The two that bite hardest in John's setup:** #1 (double responders — you run *two* implementations) and #3 (barge-in restart). Instrument both via the §5 repetition counter so a regression pages you instead of a caller noticing.

---

## 7. Frontier — full-duplex, native S2S, speculation, sub-500 ms

**Full-duplex models (Moshi/Kyutai, Sesame CSM):** listen and speak on the same stream, no explicit turn-taking, collapse the whole cascade into one model. Upside: sub-500 ms, human overlap, no endpoint-delay lever at all. Downside for John: you **lose deterministic control** of the verbatim pitch and fixed FAQ — they improvise by nature, violating two hard constraints.

**Native S2S vs cascade:**
- *Native S2S (Grok worker):* lower latency (one hop), better prosody/interruption, **weak determinism**, harder tool-calling — exactly the friction already logged (paraphrasing, stage-direction leakage, cold-start).
- *Cascade (Vapi):* higher floor, but **total control** — verbatim audio injection, fixed FAQ, clean tools, easy logging. For a compliance-shaped cold-call agent, control is worth latency.

**Speculative dialogue:** generate on the stable partial before endpoint; commit if final matches, discard if not. Upside 100–250 ms; downside doubles a repetition/double-speak risk — only after §6 idempotency is rock-solid.

**Honest take — is it worth it for John?**
- **No** third architecture (Moshi/CSM) now. John's hard constraints (verbatim pitch, fixed FAQ, DNC/transfer tools) favor *control*; full-duplex fights you on exactly those. You'd trade two product guarantees for ~200–400 ms.
- **The Grok S2S worker is already your frontier bet.** Extract its wins (sub-second warm replies) *behind* a determinism layer: pre-cached verbatim/FAQ audio injected over the realtime stream, strict single-responder.
- **Spend the next week on EXPERT moves (§3), not frontier:** semantic endpointing, filler-on-endpoint, audio-cached static turns, prefix caching. **~400–800 ms of perceived latency** for medium effort and **zero** new product risk.

**Bottom line:** John's ceiling is *perceived* TTFA ≈ **350–450 ms** (filler + cached static turns) on the cascade, and **~700–900 ms warm** on Grok — without surrendering a single product guarantee. Frontier models would shave maybe 200 ms more and cost you the guarantees. Not worth it yet.
