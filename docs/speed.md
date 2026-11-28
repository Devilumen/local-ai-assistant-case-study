# Breaking the Latency Barrier

*From minutes to seconds: a local AI assistant 10× faster on the same 16 GB GPU — with a 5× larger model, a 100×
faster first token, and every wrong guess along the way.*

## The Result

| | Before (Nov 17) | After (Nov 27) |
| --- | --- | --- |
| First reply to a message | up to **1 min 45 s** | median **9.5 s** |
| Model request | 5–7 per reply, 12–18 s each | median **5.3 s** with up to 70K tokens of context |
| Generation | 13 tok/s | **45–65 tok/s** |
| Model on the same 16 GB GPU | 27B | **125B** (MoE, 6B active) |
| First token on a 19.6K-token prompt | ~12 s | **0.12 s** |
| Service request (memory summary, image description) | 3.2 s | **0.7 s** |
| Video analysis | 108 s | **46 s** |
| Voice message | 12–52 s | **1–3 s** |

I keep two measures apart on purpose:

- **First reply**: from an incoming message to the first outgoing one. This is what the other person feels.
- **Model request**: a single LLM call. One reply can take several, plus a memory search, speech synthesis and a
  deliberate "typing…" pause, the way a person types.

## The Path, Step by Step

### 1. Find where the time goes first

The complaint was simple: "it takes almost two minutes to answer." The first guess was a slow memory search. The
server logs said otherwise:

- generation ran at 13 tok/s, because part of the 27B model did not fit on the GPU;
- one reply took **5–7 requests** to the model;
- in every request the model first wrote a "thoughts" block of 150–250 tokens, **12–18 s** per request.

The second guess was that side requests evicted the main conversation from the cache. **It did not hold:** after a
side request the main one started in 1.6 s, because the server restored its cache from RAM.

**Conclusion:** the time went into generation length and the number of requests per reply, not into search or cache.

### 2. How much to reason

The same set of turns ran with reasoning off, low, medium and extra-high. Quality was scored blind by two LLM judges
from another model family, calibrated against my own picks:

- off → low brought the biggest quality gain: fewer invented memories and fewer unnecessary tool calls;
- extra-high was four times slower and did not score best.

**Low** became the default. Method and full results: [The Cost of Thinking](evaluation.md).

**Did not work:** MTP (speculative decoding) while part of the layers stayed on the CPU: 12.9 vs 13.8 tok/s.

### 3. A 5× larger model, 3× faster

The switch to a 125B mixture-of-experts model: only 6B parameters work per token, the most used experts stay on the
GPU and the rest in system RAM. A turn in the sandbox, same scenarios: **51 → 17.5 s**; generation **13.8 → 45–65
tok/s**. The bottleneck turned out to be system RAM, not VRAM. Memory budget and measurements: [125B on a single
16 GB GPU](moe-on-16gb.md).

### 4. A turn without a wasted second

After the core refactoring, every turn writes one log line with its stages: model, tools, voice, memory, and the time
to the first visible action. No conversation text goes into it. Those lines exposed a detail: turn durations were
always round numbers. The Telegram client's answers were polled **once per second**, so every turn waited up to an
extra second. Polling now runs every **50 ms**.

A live run against thresholds approved **before** the test: 21 regular turns, median request **5.3 s** against a
5.6 s threshold, p95 20.6 s, zero errors — while the context grew to 54–70K tokens.

### 5. The cache a metric was hiding

In the per-request metrics, the "cached tokens" field was always 0. That looks like "the cache does not work" — but
then a 60K-token request would take ~30 s, not 5. A probe on the server:

- a 19.6K-token prompt: first token in **11.9 s**;
- the same prompt again: **0.12 s**, 19,625 of 19,632 tokens cached;
- same start, new ending: almost all of it cached;
- a new start: no cache, as expected.

The cache had worked all along. The metric read the wrong field of the response. Fixed.

**Lesson:** before fixing speed, check that the metric measures what you think it does.

### 6. Service requests and media

- Memory summaries and image and video descriptions need no reasoning. A check on the server confirmed that the flag
  really turns it off: **3.2 → 0.7 s** on a short request.
- Video used to be described with one request per frame. Now all frames go in one request, with a per-frame fallback
  when the answer is off-format: **108 → 46 s** on the test video. An endless retry on an empty answer went away too.
- Voice messages: 12–52 → 1–3 s after the synthesizer switch, details in [Humanizing the Machine](voice.md).

## What Did Not Work — and Why That Counts Too

| Idea | What the measurement showed |
| --- | --- |
| Memory search is the bottleneck | no: generation and the number of requests were |
| Side requests evict the cache | no: the server restores the cache from RAM in 1.6 s |
| MTP speeds up generation | not while layers stay on the CPU: 12.9 vs 13.8 tok/s |
| More reasoning is better | no: extra-high was 4× slower and not the best |
| Two requests in parallel | the server handles them almost one after another: +20% at best |
| Tell the model in the tool description to send several messages in one call | no effect: the model follows the habit in the conversation history, and in a clean context it already does this |

## How It Was Measured

- **Sandbox:** the real turn loop, synthetic scenarios, empty memory, nothing reaches people. The same scenarios
  before and after — see [The Cost of Thinking](evaluation.md).
- **Live metrics without text:** one line per request (duration, tokens, cache, tool names) and one line per turn
  (stages). A separate script summarizes a run.
- **Thresholds set in advance:** "no worse than before" is defined by numbers approved before the test, not fitted
  afterwards.
- **Synthetic probes:** every hypothesis gets a small run on the real model before any code changes.
- **Two agents:** one writes and checks the code, a second one from another vendor reviews it independently — see
  [AI Writes. AI Audits. Human Decides.](ai-assisted-workflow.md).

## What's Next

- A benchmarking tool anyone can run: memory, latency, GPU temperatures and configuration, as a separate open
  repository.
- The forced memory search before a reply — the most visible remaining extra step.
- Voice calls: a different latency budget, where a second of silence is more noticeable than in a chat.

**More from this project:** [Rebuilding Without Breaking](architecture.md) · [Intelligence Under Constraint](llm.md) · [Scaling Intelligence, Not Latency](moe-on-16gb.md) ·
[The Cost of Thinking](evaluation.md) · [Humanizing the Machine](voice.md) · [Staying in Character](prompt-injection-evaluation.md) ·
[AI Writes. AI Audits. Human Decides.](ai-assisted-workflow.md) · [README](../README.md)
