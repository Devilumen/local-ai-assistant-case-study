# Sandbox Evaluation: Latency, Reasoning Effort, and Blind Judging

This document expands on the sandbox evaluation in the [main case study](../README.md). It records local experiments on one machine. It is not a general benchmark.

## Why a Sandbox

Measuring a conversational assistant in real chats has three problems: results cannot be repeated, private conversations get involved, and every test changes the assistant's memory.

The sandbox runs the assistant's real turn loop and the same local model, with the following changes:

- The chat is simulated. Each scenario is a list of owner messages, and replies are recorded instead of sent.
- Every scenario starts with empty memory, and real memory is never read or written.
- The sandbox exposes the same tools the model uses for replies. Voice notes are recorded as text, without synthesis.
- Every model request is logged with its timing, token counts, the tools it called, and the size of its reasoning block.

The same scenario set can be re-run after any change, and the reports are compared directly.

## Baseline and Noise

Three identical runs of the baseline set (4 scenarios, 12 turns and one memory save):

| Run | Total | Memory save |
| --- | --- | --- |
| 1 | 401 s | 70 s |
| 2 | 415 s | 55 s |
| 3 | 488 s | 74 s |
| Earlier single run | 494 s | 119 s |

Run-to-run spread is about ±10%. Server timings were stable. The variation came from the model: the same turn could take anywhere from 2 to 7 model requests, depending on whether it decided to search memory or to split its reply.

The memory save got faster after a fix. A validity check used to match an unrelated word and threw away a correct result, so the step was generated twice.

## Reasoning-Effort Matrix

The model's chat template supports several reasoning-effort levels. Each level was run on the same 7 scenarios (16 turns). They cover ordinary conversation, care, commands, pressure, a voice-note request, a self-description question, and two memory checks: recalling something said earlier, and refusing to confirm an event that never happened. The server was restarted for each level.

| Level | Average turn | Model requests | Output tokens | Reasoning text (chars) |
| --- | --- | --- | --- | --- |
| Off | 38 s | 82 | 6,060 | 0 |
| Low | 64 s | 54 | 12,362 | 34,244 |
| Medium | 80 s | 64 | 15,556 | 40,709 |
| Extra-high | 153 s | 66 | 31,699 | 100,990 |

With reasoning enabled, the model made about a third fewer requests: it called fewer tools at random and split replies into fewer messages. Still, the time spent reasoning outweighed the saving.

### Blind judging

Transcripts were anonymized. Timings and request counts were removed, so they could not reveal the level, and the four variants of each scenario were shuffled and labelled A–D. Two LLM judges from a different model family scored every variant from 1 to 5 on four criteria: logic, tool use, language, and character consistency. A written rubric described the intended character.

| Level | Judge 1 | Judge 2 | Combined (max 280) |
| --- | --- | --- | --- |
| Off | 99 | 96 | 195 |
| Low | 108 | 101 | 209 |
| Medium | 113 | 107 | 220 |
| Extra-high | 117 | 97 | 214 |

Both judges ranked "off" lowest. Its main failure was **invented memories**: it confirmed a past conversation that never took place and "remembered" habits it had no record of. In the false-memory scenario, both judges picked "low" as the best variant. The second judge penalized "extra-high" for tool use: unnecessary calls and repeated content.

### Calibration against a human

Before the judges saw the pack, I chose the better variant in 5 blind pairs myself. Judge 1 agreed with me in 3 of 5 pairs, judge 2 in 2 of 5. The disagreements were systematic: the judges were stricter about factual accuracy, and I weighed liveliness and character more. Judges are therefore used for repeatable scoring, while character decisions stay with a person.

### Decision

**Low** was chosen as the default. It removed the worst failure of "off" at about 1.7× the latency. Medium scored slightly higher but cost 2.1×, and extra-high cost 4×.

Calls that do not talk to the user, such as memory search, summaries and image descriptions, can switch reasoning off per request. That change is implemented but not yet measured.

### Limitations

- One run per level, on a small scenario set. The baseline shows ±10% noise in time, and quality per turn varies too.
- Memory is empty in the sandbox, so long-term memory use and the assistant starting a conversation on its own are not tested.
- LLM judges have their own preferences. Their agreement with a human was moderate.

## Speculative Decoding

The model is also available with a multi-token prediction head for speculative decoding. On this setup, with 50 of 65 layers on the GPU and the rest on the CPU, it gave **no speedup**:

| | Generation speed | Draft acceptance |
| --- | --- | --- |
| Standard | 13.8 tok/s | — |
| With prediction head | 12.9 tok/s | 63% |

The drafts were accepted often enough, but checking a batch of drafted tokens on the CPU-resident layers cost as much as it saved. The technique is worth re-testing once the whole model fits on the GPU.

## What Changed in How I Evaluate

- Every change to settings or instructions is first run in the sandbox, against a recorded baseline.
- Quality comparisons are blind: the person or model that scores does not know the configuration.
- Latency is reported per turn together with the number of model requests, because the request count explains most of the variation.
