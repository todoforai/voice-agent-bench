# voice-agent-bench

A black-box latency & behavior benchmark for **any** voice agent. A scripted "person"
(pre-generated speech, byte-identical every run) talks into a virtual mic, the agent's speaker
output is recorded, and every score is derived from **audio alone** — voice→voice latency,
barge-in stop time, stalls, false barge-ins, echo leakage.

**If it makes sound, it can be benchmarked. Zero integration required.**

## Leaderboard

Three scenarios: **clean** (smalltalk), **hesitating user** (mid-sentence pauses), **echo**
(agent's own voice fed back into the mic, no AEC). Latency = voice→voice median, pooled over
5 conversations × 6 turns (n=30).

| system | clean | hesitation | overlap → talked through | echo | cut itself |
|---|---|---|---|---|---|
| OpenAI Realtime \* | 870ms | 1290ms | 12/30 → **0** (yields in 130ms) | 790ms | **17/30** |
| **voiceloop** · deepgram + EL flash | 980ms | 1400ms | 2/30 → **0** (420ms) | 930ms | **0** |
| **voiceloop** · deepgram + Piper (free TTS, local) | 970ms | 1400ms | 0/30 → **0** | — | — |
| **voiceloop** · webspeech + Piper (zero-key, browser STT) | 2110ms | — | — | — | — |
| Pipecat 1.8.1 · same providers | 1050ms | 1290ms | 12/30 → **2** (200ms) | 1320ms | **20/30** |
| ElevenLabs ConvAI | 1450ms | 1810ms | 2/30 → **0** (490ms) | 1410ms | 0 |

\* speech-to-speech, its own LLM — every cascade system shares the fixed mock LLM; Realtime is
the disclosed exception.

Speed rankings barely move across scenarios — **what changes is who recovers**. Overlapping the
user is not a failure (people do it constantly); riding over them when they keep talking, or
cutting your own reply because you heard your own echo, is. Latency and behavior must be read
together — neither column alone ranks a system.

Full per-scenario tables, sub-metric splits and caveats: [`results/RESULTS.md`](results/RESULTS.md).

## How it works

```
scenario.json ──► driver.js ──► bench_mic ──► [ your agent ] ──► bench_spk ──► analyze.js ──► report
 (script)      (plays "person"                (any process                    (records &
                audio, fixed)                  or browser)                     scores audio)
```

- The "person" audio is generated **once** (ElevenLabs) and replayed byte-identically every run.
- Every cascade system gets the **same fixed mock LLM** (scripted responses, fixed TTFT and
  token rate) — the numbers measure the voice pipeline, not the language model.
- Latency numbers are **median + p95** (ms), never mean-only, and single runs are never
  reported: `blackbox/run-n.js` pools N conversations (±300ms network/WASM jitter has flipped
  single-run conclusions).

## Quickstart

1. Create the virtual mic/speaker pair (PulseAudio/PipeWire):

   ```sh
   blackbox/audio-setup.sh up
   ```

2. Generate the "person" audio (once, needs `ELEVENLABS_API_KEY`):

   ```sh
   node blackbox/gen-audio.js smalltalk
   ```

3. Start the bench server (static files + fixed mock LLM):

   ```sh
   node server.js smalltalk
   ```

4. Point your agent at the devices — it must listen on `bench_mic` and speak on `bench_spk`.
   For a browser agent, e.g. voiceloop itself:

   ```sh
   PULSE_SOURCE=bench_mic PULSE_SINK=bench_spk google-chrome --user-data-dir=/tmp/sut \
     --use-fake-ui-for-media-stream --autoplay-policy=no-user-gesture-required \
     http://localhost:7777/bench/blackbox/agent.html
   ```

5. Run the conversation and score it:

   ```sh
   node blackbox/driver.js smalltalk mylabel      # the virtual person talks
   node blackbox/analyze.js results/bb-….json     # → markdown report
   ```

For pooled results (the only kind we publish), `blackbox/run-n.js <scenario> <label> [N=5]`
drives Chrome over CDP and runs the whole loop N times.

## What we measure

One scripted conversation, five times, all turns pooled (median + p95, n=30). Headline timing
and behavior come from the recorded audio; internal splits (EOT, STT partial, TTS first audio)
only exist for instrumented SUTs and *explain* the audio-truth numbers, never replace them.

### Latency (ms, lower is better)

| number | meaning |
|---|---|
| **voice→voice** | person stops speaking → first audible agent audio. The headline. |
| **barge-in stop** | person starts interrupting → agent audio actually stops. |
| **first content word** | person stops → first agent word from the actual answer (a "Hmm," head start doesn't count). |
| EOT delay | person stops → agent commits the turn (endpointing decision). |
| STT first partial | person starts → first partial transcript. |
| TTS first audio | first LLM token → first audible synthesis. |

### Behavior (counts over 30 turns, 0 is perfect)

| number | meaning |
|---|---|
| **overlap** | turns where the agent started talking while the user still had the floor. Allowed — people overlap too. |
| **talked through** | of those, turns where it kept going instead of backing off when the user resumed. The failure. |
| **yield** | user resumes → agent audio stops (a latency number, listed with the behavior it belongs to). |
| **self-interruptions** | agent cut its own reply with nobody talking, or delivered <80% of it (hearing itself as the user). |
| false barge-ins | agent stopped for something that wasn't the user interrupting. |
| stalls | >250ms silent gap inside one reply. |
| echo words | agent's own words leaking into its *user* transcript. |
| WER / spoken ratio | transcript error rate / fraction of the reply actually delivered (uninterrupted replies only). |

### What we do NOT measure

Model quality. STT/TTS models have their own rigorous benchmarks (LibriSpeech, TTS Arena…);
this bench measures the realtime quality of a voice agent *implementation*. The WER/fidelity
columns only check that the implementation feeds the models properly — no truncated turns, no
echo in transcripts, no clipped replies.

## Two modes, one scorecard

Both feed the same scorer (`metrics.js`):

- **Black-box** (any agent): audio in, audio out — zero integration. See [`blackbox/`](blackbox/).
- **Instrumented** (agents that emit milestone events): internal splits (EOT / TTFT /
  TTS-first-audio) merged into the audio timeline via epoch clock.

## Fairness rules

Every system under test gets:

- the **same fixed mock LLM** — scripted responses, fixed TTFT, fixed token rate;
- the **same audio devices** (`bench_mic` / `bench_spk`);
- the **same scenario script**, byte-identical person audio.

Exceptions (e.g. speech-to-speech systems that can't use the mock LLM) are always disclosed
next to their numbers.

## Add your agent

The whole integration is "listen on `bench_mic`, speak on `bench_spk`". The full contract —
interface, fairness rules, working example SUTs (process, Pipecat, ElevenLabs ConvAI, OpenAI
Realtime), cloud tunneling, PR checklist — is in [`ADDING_A_SUT.md`](ADDING_A_SUT.md).

PRs with new SUTs or scenarios welcome.

## Repo layout

| path | what |
|---|---|
| `server.js` | static server + fixed mock LLM + result saving (zero deps) |
| `metrics.js` | the scorer — one scorecard for both modes |
| `scenarios/` | conversation scripts (`smalltalk`, `hesitation`, `echo`, `noise`) |
| `audio/` | pre-generated "person" speech per scenario |
| `blackbox/` | the rig: virtual devices, driver, analyzer, run-n pooling, example SUTs |
| `results/` | published runs + [`RESULTS.md`](results/RESULTS.md) |

## Origin

Developed alongside [voiceloop](https://github.com/todoforai/voiceloop)
([live demo](https://todoforai.github.io/voiceloop/)) — it lives at `bench/` there and is split
to this repo — but the rig is agent-agnostic.
