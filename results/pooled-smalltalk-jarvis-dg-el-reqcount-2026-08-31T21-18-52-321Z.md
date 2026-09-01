# pooled: jarvis-dg-el-reqcount / smalltalk — 2 runs, 12 turns

> COST PROBE (2 runs, 12 turns) — measures LLM requests per turn at the bench server with
> `prefetchMs: 0` on deepgram: **92 requests / 12 turns = 7.7 per turn**. Pair with
> jarvis-dg-el-basecount (the same probe without the fix). v→v here is incidental.

| metric | pooled median (p95, range, n) |
|---|---|
| voice→voice | 864 (p95 1113, 439–1113, n=12) |
| first content word | 864 (p95 1113, 439–1113, n=12) |
| EOT delay | 436 (p95 942, -7–942, n=12) |
| TTS first audio | 421 (p95 684, 154–684, n=12) |
| STT first partial | 762 (p95 912, 600–912, n=12) |
| barge-in stop | 1017 (p95 1078, 510–1078, n=4) |
| per-run v→v medians | 864, 905 |
| stalls / false barge-ins (total) | 8 / 0 |
| agent-stalled-by-noise (total) | 0 |
| echo words / self-interruptions / echo drops (total) | 0 / 0 / 0 |
| user-interrupted (total) | 0 |