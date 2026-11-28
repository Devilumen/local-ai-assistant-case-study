# Scaling Intelligence, Not Latency: 125B MoE on a 16 GB GPU

*A 5× larger model, 3× faster replies, 2× the context and no loss in blind-judged quality — on the same 16 GB GPU.*

## Result

Same consumer GPU with 16 GB of VRAM, same assistant. Only the model and the inference engine changed:

| | Before | After | Change |
| --- | --- | --- | --- |
| **Model** | 27B parameters | **125B** parameters | **4.6× larger** |
| **Reply per turn** (sandbox) | 51 s | **17.5 s** | **2.9× faster** |
| **Generation speed** | 13.8 tok/s | **45–65 tok/s** | **3.3–4.7×** |
| **Context window** | 64K tokens | **128K** | **2× larger** |
| **Voice message** | 12–52 s | **1–3 s** | **4–17× faster** |
| **Memory save** | up to 7.7 min | **~2 min** | **3.8× faster** |
| **Quality** (blind judges) | 282 / 360 | **291 / 360** | **no loss, persona +14%** |
| **Human pick** | 1 of 5 | **4 of 5** | in favor of the new model |
| **Stability** | at the VRAM limit | **~13 h live** | not a single memory failure |

All of it runs together with the speech-synthesis model on the same GPU.

**How:** a mixture-of-experts (MoE) model, where only 6B of the 125B parameters are active for each token, and an engine that keeps the "hot" experts on the GPU and the rest in system RAM. The hardest part turned out to be system RAM, not VRAM: the fastest configuration left no room for speech synthesis and pushed the system into swap. Below is how the working balance was found, what did not work, and where these numbers end.

*How to read the numbers:* "reply per turn" is the average turn in the test sandbox (9 scenarios, the assistant's real turn loop with a simulated chat). In the configuration that is actually used it is 20.5 s, which is 2.5× faster; a typical live turn takes 2.5–10 s. Voice messages got faster because the freed memory made it possible to keep the speech model loaded at all times. Everything was measured on one machine: these are observations, not a benchmark.

## Starting Point

| Parameter | Value |
| --- | --- |
| GPU | RTX 5070 Ti, 16 GB |
| CPU / RAM | Ryzen 7 9700X, 48 GB (part of it assigned to WSL) |
| Model | dense 27B, IQ3_S quantization |
| Placement | 50 of 65 layers on the GPU, the rest on the CPU |
| Generation | 13.8 tok/s |
| Sandbox turn | 51 s on average |
| Peak VRAM with speech synthesis | 15.4–15.6 of 16.3 GB |

Speculative decoding with a multi-token prediction (MTP) head gave nothing on this model: while part of the layers runs on the CPU, the gain is eaten up (12.9 vs 13.8 tok/s). This was expected, but it was measured rather than assumed.

## Why MoE

A mixture-of-experts model uses only a small part of its parameters for each token: here, 6B out of 125B. The compute per token is close to that of a small model, while the knowledge is that of a large one. The only question is where to keep the remaining experts. A specialized engine keeps a cache of the most used experts on the GPU and the rest in RAM, or even streams them from NVMe.

Model: Qwen3.8-Flash-Next (125B MoE, 512 experts, 6B active), IQ3_XXS quantization. Engine: Strata. According to its model card the new model beats the previous 27B on every benchmark listed, but those benchmarks are in English and agent-oriented, while the assistant is a Russian-speaking conversational persona. So it was tested on our own workload.

## Memory Budget: the Fastest Option Does Not Work

A short request while a voice message is synthesized in parallel:

| Configuration | Generation | With voice | Voice message | Peak VRAM | Peak RAM |
| --- | --- | --- | --- | --- | --- |
| A — defaults | 88.5 tok/s | 87.1 | 31 s | 15.85 GB (at the limit) | 47.1 GB (swap) |
| B — 3 GB VRAM reserved for voice | 77.0 | 72.1 | 28 s | 15.7 | 47.1 (swap) |
| **C — B + 20 GiB cap on experts in RAM** | **65.4** | **62.0** | **12 s** | 15.7 | **34.2** |

In configuration A the model took almost all of the VRAM and all of the system RAM. On its own it was very fast, but a voice message took 31 s, and the engine warned that under memory pressure decoding would drop to a few tokens per second. Capping RAM (C) cost 26% of generation speed but removed swapping, and voice became 2.5× faster. What matters for the product is the speed of the whole chain, not a single number.

Later the speech model was kept loaded permanently (a voice message in 1–3 s instead of 12–52 s), and the VRAM reserve for it was raised from 3000 to 3800 MiB.

## Speed: Sandbox and Live Use

**Sandbox:** the assistant's real turn loop with a simulated chat, 9 scenarios, 21 turns:

| | Average turn | Model requests | Generation |
| --- | --- | --- | --- |
| 27B, dense | 51.2 s | 79 | 13.8 tok/s |
| 125B MoE, configuration A | **17.5 s** | 81 | 60–90 tok/s |
| 125B MoE, configuration C (in use) | **20.5 s** | — | not measured in the sandbox |

**Live**, in Telegram: about 3 hours, then an overnight run of almost 10 hours:

- a typical turn takes 2.5–10 s, generation runs at 45–69 tok/s;
- saving the conversation to memory when the context fills up takes about 2 min (up to 7.7 min with the 27B);
- no engine memory failures; RAM 40–42 GB and VRAM 13–14 GB, steady.

## Long Context: 128K Together With Voice

- A 124,130-token prompt was read in **64 s** (about 1,930 tok/s); generation after it ran at **45 tok/s**.
- A voice message is synthesized without memory errors while the context is full.
- A request of about 152K tokens is rejected by the server, so the limit is real, and the memory-save threshold (96K) leaves headroom.

## Quality: Blind Comparison

Nine scenarios, one answer per model. The answers were shuffled and anonymized, two LLM judges scored them against a rubric (up to 180 points each), and the key was revealed afterwards.

| | Judge 1 | Judge 2 | Total / 360 | Persona (sum) | Average turn |
| --- | --- | --- | --- | --- | --- |
| 27B, temperature 0.4 | 149 | 133 | 282 | 57 | 51 s |
| **125B MoE, 0.4** | **151** | **140** | **291** | **65** | **17.5 s** |
| 125B MoE, 1.0 | 131 | 129 | 260 | 60 | 20 s |

- The difference in the total score is small (+3%), honestly within what 9 scenarios can tell. The point is that **quality did not drop** while replies became 2.9× faster, and persona consistency improved (+14%).
- The higher temperature lost with both judges, so 0.4 was kept.
- In paired comparisons a human reviewer chose the new model in **4 of 5** cases.
- Both models sometimes repeated example lines from the prompt word for word. So this is a property of the prompt, not of model size, and a separate task.

## What Did Not Work and What I Learned

1. **The fastest number is not the fastest system.** Configuration A won on tokens per second and lost on voice and stability.
2. **Engine updates must be measured.** The next engine release turned out about 25% slower on this workload (39–41 vs 49–56 tok/s). I rolled back and do not update without a new measurement.
3. **Returning VRAM on the fly** (to give it to speech synthesis) is not available in this mode: the engine switches to its resident mode by itself when the experts do not fit in RAM, and the feature does not work there.
4. **An ordinary desktop costs speed.** With a browser, video, and an IDE open, generation drops to about 50 tok/s from 65 on a clean system: less room for the expert cache.
5. **Saving memory resets the prompt cache.** After compaction the beginning of the context changes, and the engine re-reads 26–33K tokens (about 25 s). This is unavoidable with this memory design; the fix is to save less often, which the 128K context made possible.

## Limitations

- One machine, one workload. The numbers are observations, not a benchmark.
- Nine scenarios are a small sample; LLM judges are not people, which is why they were calibrated against a human.
- IQ3_XXS is a heavily compressed quantization; the model-card benchmarks were not measured on it.
- 48 GB of RAM is the lower bound for this model next to the rest of the stack; the model has to be unloaded while gaming.

## Next Steps

- Check whether the prompt prefix cache is reused between turns (the server does not report the counter yet).
- Re-test MTP now that almost the whole model runs in fast layers.
- Publish the latency and memory benchmarking tool as a separate repository.
