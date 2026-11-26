# Local AI Assistant: Voice, Vision, Memory, and Behavior

A case study of adapting and evaluating a local AI assistant with long-term memory, voice interaction, and Telegram tools on consumer hardware.

## Project Overview

This project focuses on developing a personal AI assistant that can maintain ongoing conversations through Telegram, use available tools, and take saved context into account.

The work spans assistant behavior and instruction following, language and speech model integration, memory management, tool evaluation, debugging, and hardware optimization.

**Project technologies:** C++20, Python, Telegram/TDLib, local LLM inference, Docker, Windows, and Linux/WSL.

> The assistant's source code, private conversations, prompts, and internal implementation are not published in this repository.

## Assistant Capabilities

Within its configured permissions, the assistant can select and use tools for communication and chat management:

- **Text conversation:** Receive messages and respond using the current conversational context.
- **Telegram actions:** Send and edit messages, use stickers and reactions, and create and send voice messages.
- **Interaction management:** Use available chat and user actions, including blocking, subject to system configuration and restrictions.
- **Memory retrieval:** Search for relevant records about people, events, and previous conversations.
- **Context preservation:** Summarize conversations and update working memory for future use.
- **Voice interaction:** Convert prepared text into speech and send it through Telegram.
- **Speech recognition:** Transcribe incoming voice notes and respond to what was said.
- **Image understanding:** Answer questions about photos and screenshots, taking the caption and the conversation into account.
- **Reminders:** Set reminders and deliver them on time, backed by a real timer rather than a promise in the text.
- **Character consistency:** Follow system instructions and character settings to maintain a distinctive conversational style.
- **Action selection:** Choose, based on available context, whether to respond, consult memory, use a tool, or end the current processing cycle.

Available actions depend on the scenario and configured permissions.

## Behavior and Decision-Making

A separate part of the project evaluates how the language model follows instructions and selects actions when interacting with users and tools.

Testing covered:

- Following system instructions and the intended character in ordinary conversation.
- Maintaining a consistent style across topic changes and ambiguous requests.
- Responding to direct instructions, emotional messages, and requests to take initiative.
- Choosing between responding, consulting memory, and using tools.
- Applying retrieved information without losing track of the conversation.
- Comparing four levels of additional model reasoning in a reproducible sandbox (see below).

The model could maintain the intended style and use tools, but quality depended on context, settings, and instruction wording. Some replies were overly brief or repeated similar phrasing. Work on naturalness and behavioral consistency continues.

Here, **decision-making** means selecting a response or an available action based on the context and instructions provided to the model.

### Finding: example lines get copied

Comparing transcripts with the character instructions showed that the local model often reproduced the example lines from the instructions almost word for word, instead of treating them as an illustration of style. Responses to similar situations therefore followed one fixed pattern.

A revised approach describes the character's conversational *moves* rather than ready-made lines, and marks every example as "the move, not the words". It is being evaluated in the sandbox against the current version; no results are reported yet.

## Memory of Conversation Partners

The assistant can store information about people and previous conversations, then search for related memories during interaction.

Each chat has its own conversation history and its own memory, so information from one conversation is not retrieved in another. When a memory search finds nothing, the assistant now says it does not remember instead of filling in the gaps; this is covered by tests. A separate memory per person, across several chats, is not implemented yet.

## Deployment Challenge

The main technical constraint was running a large language model alongside local speech synthesis on a consumer GPU with **16 GB of VRAM**.

The assistant also needed to preserve context, use tools, and interact through Telegram without depending on a paid cloud LLM API.

The main questions were:

- How much of the model could be placed on the GPU?
- What context size was practical with a quantized model?
- Could the model and speech synthesis share the available VRAM?
- How could repeated requests be avoided when the context overflows?

## Local LLM Deployment

A quantized **27B-parameter language model** was configured for local inference. The evaluation covered:

- Partial GPU offloading, with remaining work handled by the CPU.
- A **65,536-token context window**.
- **Q8 KV-cache quantization**.
- Conversational quality, tool calling, response latency, and memory usage.
- Integration through an OpenAI-compatible API.

In one synthetic test, the model processed approximately **60,000 input tokens** while local speech synthesis was also running.

See [LLM deployment and evaluation](docs/llm.md) for the test configuration and limitations.

## Upgrade: a 5× Larger Model, 3× Faster Replies

The 27B setup above was later replaced, on the same 16 GB GPU, by a **125B mixture-of-experts model** with only 6B parameters active per token. The engine keeps the most used experts on the GPU and the rest in system RAM.

| | Before | After |
| --- | --- | --- |
| Model | 27B | **125B** (4.6×) |
| Reply per turn (sandbox) | 51 s | **17.5 s** (2.9× faster) |
| Generation | 13.8 tok/s | **45–65 tok/s** |
| Context window | 64K | **128K** |
| Voice message | 12–52 s | **1–3 s** |
| Quality, blind judges | 282 / 360 | **291 / 360**, persona +14% |
| Human pick | 1 of 5 | **4 of 5** |
| Live stability | at the VRAM limit | **~13 h** without a memory failure |

Speech synthesis still shares the same GPU. The hardest constraint turned out to be system RAM: the fastest configuration left no room for voice and pushed the system into swap, so the configuration in use trades 26% of generation speed for 2.5× faster voice.

See [125B on a single 16 GB GPU](docs/moe-on-16gb.md) for the memory budget, the measurements, what did not work, and the limitations.

## Reproducible Sandbox and Reasoning-Effort Evaluation

To compare settings without touching real conversations, I used a sandbox. It runs the assistant's real turn loop against the local model, but with a simulated chat, synthetic scenarios and empty memory. Nothing goes to Telegram, and every request is measured.

- **Baseline noise:** three identical runs took 401, 415 and 488 s. The spread came from the model choosing a different number of requests per turn, not from the server.
- **Memory save (sandbox, 27B model):** saving a conversation to memory dropped from 119 s to an average of 66 s after a duplicated generation step was removed. In live use with a full context, the save took up to 7.7 min with the 27B model and about 2 min after the move to the 125B model.
- **Reasoning effort:** the same 16 turns were run with reasoning off, low, medium and extra-high. Quality was scored blind by two LLM judges from a different model family, and I calibrated their scores against my own picks.

| Reasoning | Average turn | Judges' combined score (max 280) |
| --- | --- | --- |
| Off | 38 s | 195 |
| Low | 64 s | 209 |
| Medium | 80 s | 220 |
| Extra-high | 153 s | 214 |

The biggest quality gain came from the first step, off to low: fewer invented memories and fewer unnecessary tool calls. Extra-high cost four times the latency and did not score best. **Low** was chosen as the default.

These are single runs per level on a small scenario set. The judges agreed with my own picks in 3 and 2 of 5 calibration pairs, so on character questions human judgment stays decisive.

See [Sandbox evaluation](docs/evaluation.md) for the method and the full results.

**Character and instruction following.** See [Staying in Character | Staying on Task](docs/prompt-injection-evaluation.md) for two checks: how the assistant handles pressure and whether instructions in search results can steer it away from the user's request.

## Voice and Multimodality

Local speech synthesis was integrated into the Telegram conversation workflow. Generated speech was converted into a format suitable for Telegram voice messages.

The end-to-end flow was tested in a real conversation: the assistant generated and successfully sent voice messages.

**Update: humanizing the voice.** Over 6 listening rounds (~80 variants, every decision by ear), the voice got a natural pace and smoother intonation, confirmed name stress, 9 approved vocal reactions, and no more cut-off endings or hiss at the start. It uses about 2.3 GB of VRAM next to a 125B language model on the same 16 GB GPU; the previous synthesizer took almost the whole card.

See [Local Voice Interface](docs/voice.md) for the voice requirements, evaluation, and limitations.

Speech recognition and image understanding are part of everyday use. Voice notes were transcribed correctly in live conversations, and the assistant answers questions about photos and screenshots with the caption and the conversation in mind. Image descriptions run on the CPU next to the language model and are cached per chat. Measured on 26 November 2026, on the same machine, next to the 125B model:

| | Latency | GPU memory |
| --- | --- | --- |
| Speech recognition, 14 s Russian voice note | **6.3 s** (9.8 s on the first call) | none: runs on the CPU |
| Image question, photo | **2.7 s** | +33 MB: the vision encoder runs on the CPU (~1 GB of RAM) |
| Image question, screenshot | **3.7 s** (up to 5 s on the first call) | same |

Two synthetic examples (no private data): for a close-up photo of a cat it answered "a close-up of a cat's face with a long pink tongue sticking out"; for a flight-status screenshot asked "when is the departure now and from which gate?" it answered "19:55, gate B7", which is correct. Three runs each, reasoning off as in the assistant's image descriptions. Accuracy was not measured on a larger set.

## Architecture Cleanup

After several rounds of experiments, the codebase still carried disabled features, dead code, and one shared memory store for every conversation. I first took a full snapshot and mapped the dependencies. Then I removed three unused subsystems and about 3,600 lines of dead code, gave each conversation its own memory, and restricted tools that could cross between chats. Every finding from an independent code audit was verified before it was fixed.

The codebase ended up slightly smaller (120 → 108 source files) while gaining reliability logic, a sandbox, and tests (279 → 295 at the time of the cleanup; 402 today). In the sandbox, the same scenarios got 16% faster.

Later the core turn loop itself, now called **DEVCORE**, was refactored in six reviewed steps under 52 golden traces that had to match byte for byte after every step. It lost all of its `goto` jumps, and a live comparison with the earlier baseline showed the same behaviour and speed.

See [Architecture cleanup and stabilization](docs/architecture.md) for the approach and the results.

## Working With Two AI Agents

The code is written with AI coding agents: one implements and reviews its own work, a second one from another vendor audits it independently, and I make the decisions and approve every outward action. Every finding is checked against the code before it is fixed. Over six reviews the auditor reported 67 findings: 66 were confirmed, none were false alarms. In two of them it found three serious issues each that the implementer had missed, including one the implementer had explicitly marked as safe. During the core refactor it found two regressions that the golden traces had not caught.

See [Two AI agents and an owner](docs/ai-assisted-workflow.md) for the roles, the rules and the examples.

## Reliability Case: Context Overflow

During testing, the assistant stopped responding when a request exceeded the model's context limit. The application repeatedly sent the rejected request while adding new instructions, causing the context to grow further.

The investigation led to changes that:

- Stop repeated empty or failed model responses from creating an infinite loop.
- Trigger memory-saving operations before the configured context limit is reached.
- Avoid clearing temporary context when a memory summary is empty or truncated.
- Limit retries and preserve information for recovery.

## Reliability Case: Memory That Survives Failures

A code review of the memory pipeline found several ways to lose or corrupt saved context. The fixes:

- **Shutdown:** an interrupt or termination signal now stops taking new events, lets the current step finish without sending new messages, saves memory for every open conversation, and only then exits. This was checked on the running system.
- **Oversized requests:** the request size is estimated before it is sent. Previously, one oversized request could block a conversation permanently, even across restarts. Now an overflow triggers a memory save instead.
- **No duplicates after a crash:** memory saving keeps a journal and resumes from where it stopped instead of writing the same summary twice.
- **Cleaner summaries:** the model's internal reasoning and previously retrieved memory pages are no longer summarized again as if they were new events.
- **Atomic writes:** a corrupted snapshot no longer blocks a conversation, and each turn is capped at a fixed number of model requests.

The application now has **402 automated tests** (26 November 2026), including 52 golden traces of the turn loop. They run without live services. Another 23 integration tests need the running model, speech services or a real Telegram account and are run separately.

## Observations and Limitations

During the combined long-context and speech-synthesis test, sampled GPU memory usage reached **15,604 MiB out of 16,303 MiB**.

This was one test under specific conditions, not a guarantee that every workload will fit. The remaining VRAM headroom was small, so longer voice messages, other GPU applications, or different workloads may require further adjustments.

The size check before each request is a conservative estimate based on the amount of text sent, not an exact tokenizer count.

**Access during testing:** The assistant runs in owner-only mode: it talks to one authorized user.

- **Covered by automated tests:** each chat has its own memory folder and history; image descriptions are cached per chat (a leak between chats through a shared cache was found and closed with a test); the chat list shows only the current chat; forwarding goes only to the owner.
- **Not tested yet:** live use with more than one real person, and the full set of actions involving other users.
- **Not implemented:** a memory per person that spans several chats.

## What I Learned

- Local AI deployment requires balancing model quality, latency, context size, and memory.
- A model responding correctly in a standalone chat does not guarantee reliable tool use inside an application.
- Assistant behavior needs evaluation across different situations, not just one successful conversation.
- Logs and repeatable tests are essential when several services interact.
- Persistent memory requires failure handling, not just a summarization prompt.
- Consumer hardware can support complex local AI workflows, but only with careful measurement and realistic limits.
- More reasoning is not automatically better: a small amount helped, and the maximum mostly added latency.
- A smaller local model follows instruction examples literally, so a character should be described by its behavior rather than by sample lines.
- A speed technique that works in benchmarks may give nothing on a given setup. Speculative decoding showed no gain while part of the model ran on the CPU.
- An LLM judge is useful for blind, repeatable scoring, but needs calibration against a human on the questions that matter.

## Next Steps

- Fit the whole model on the GPU, then re-test speculative decoding and higher reasoning effort.
- Evaluate the behavior-based character instructions against the current version.
- Reduce latency from memory searches that run on almost every turn.
- Develop a mechanism for modeling the assistant's emotional state, influencing its conversational style and reactions according to dialogue context.
