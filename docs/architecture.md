# Architecture Cleanup and Stabilization

This document describes how I cleaned up my assistant's codebase after several rounds of experiments: I removed what it no longer needed and made the rest reliable enough to measure and extend. It complements the [main case study](../README.md). Source code is not published; the numbers come from the project's version history.

## Starting Point

Over several iterations the assistant had collected features I had tried and then switched off, along with workarounds that were no longer needed. Before the cleanup, the application had about **12,800 lines** of C++ in 120 source files. Several kinds of problems had built up:

- Features that were switched off but still compiled, configured and loaded: a cloud speech backend, an image-generation pipeline, and a local proxy server with its own HTTP library.
- Commented-out code and debug leftovers. One leftover wrote every analysed video frame to disk as an image file.
- Work that was computed and then thrown away. For example, the code searched memory on every step even when memory injection was disabled.
- A single shared memory store for all conversations, which made it impossible to guarantee that information about one person never surfaced in another chat.
- Shutdown, overflow and crash paths that could lose or duplicate saved memory (see the reliability section of the main case study).
- Part of the work, including the assistant's behaviour files, was not under version control at all.

## Approach

1. **Safety net first.** I archived the project, put every file under version control on a separate branch, and added ignore rules for configuration, private data, logs and caches. Nothing was rewritten before this snapshot existed.
2. **Map before cutting.** A code graph of the project (about 1,400 nodes and 2,100 relations) and a written project map showed which modules depended on what. It also showed what was reachable only from disabled features.
3. **Independent review.** I used two AI coding agents from different vendors. One implemented changes and reviewed them; the other audited the code separately. Every finding was checked against the code before anything was fixed. In the dead-code audit, all 9 findings were confirmed, with no false positives.
4. **Owner decisions for features, not guesses.** Disabled features were not deleted silently. Each one was listed with what it cost and what depended on it, and I decided per feature: remove, keep, or postpone.
5. **Measure after every step.** After each larger change, the unit tests and the reproducible sandbox were re-run against a recorded baseline.

## What Changed

### Removed

- The cloud speech backend and the switch between backends.
- The image-generation pipeline, together with its tool, client, settings and container definition.
- The proxy server and its HTTP library. This exposed a hidden dependency: the messaging library had been getting its TLS library only through the removed HTTP library. That dependency is now declared explicitly.
- Dead code, unused configuration keys, and a stale image-generation prompt that was sent with *every* model request.
- Around 4 GB of model files that only the removed pipeline used.

The dead-code commit alone removed **3,555 lines**.

### Restructured

- **Memory per conversation.** Each chat now has its own memory store. The old shared store was archived, not deleted, because it mixed records about different people.
- **Cross-chat tools restricted.** Chat previews show only the current conversation. Forwarding is limited to the owner and rate-limited. Sticker shortcuts resolve against what the model was actually shown.
- **Read receipts after success.** A message is marked as read only after the turn that answered it succeeded. A failed turn therefore leaves the message to be picked up again after a restart.
- **Streaming robustness.** Responses that arrive split across network chunks are reassembled. A stream that ends without its completion marker is treated as an error, not as a short answer.
- **Behaviour outside the binary.** Character and instruction texts live in separate files, loaded at runtime, so they can be edited without rebuilding.

### Operations

- One start script and one stop script for the whole local stack: language model server, voice bridge, embedding container and assistant. They record process identities, refuse to take over unknown processes, and stop the services in a safe order: the assistant first, then the containers the script started, then the voice bridge, and the model server last.
- The stop script sends the termination signal once and then waits up to 30 minutes, so a long memory save is not cut off by a second signal.
- Infrastructure scripts and documentation got their own repository with an allowlist-based ignore file, separate from the application code.

## Results

| | Before | After |
| --- | --- | --- |
| C++ source files | 120 | 108 |
| C++ source lines | ~12,800 | ~12,300 |
| Unit and component tests | 279 | 295 (272 run without live services) |
| Sandbox turn time, 4 scenarios | 445 s | 374 s (−16%) |
| Memory save | 119 s | 66 s on average |

The application lost about 3,600 lines of dead or unused code. It gained roughly as many lines of reliability logic, isolation, a sandbox and tests, so the codebase ended up slightly smaller while doing more.

### Where the code grew

Two central files actually **grew**:

| File | Role | Before | After |
| --- | --- | --- | --- |
| Application entry point | Messaging client, tool wiring, process lifecycle | 776 lines | 848 lines (+9%) |
| Core turn loop | Event queue, model calls, memory, timers | 695 lines | 1,026 lines (+48%) |

Almost all of the growth came from one reliability change, which touched these two files with about +400 / −50 lines:

- **Signal-driven shutdown.** Interrupt, termination and hangup signals now go through a small async-signal-safe handler. The handler only writes a byte to a pipe, and a watcher thread turns that byte into the same graceful stop as a manual one. A second signal exits immediately.
- **Shutdown sequence.** The loop stops taking new events, the current step finishes without starting new tool calls, and every conversation with unsaved context is written to memory before the process exits.
- **Context budget.** Before each request, the prompt size is estimated from its byte length, using a bytes-per-token ratio calibrated from the token counts the server reports. If the next request would cross the threshold, memory is saved first. If the server still rejects a request as too long, that also triggers a save instead of blocking the conversation.
- **Recovery.** The selected conversation is persisted and restored after a restart, the startup conversation's checkpoint is loaded, and a corrupted checkpoint is quarantined instead of blocking the chat.
- **Thread discipline and limits.** Timer callbacks now run on the application thread instead of touching shared state from another thread, and a single turn is capped at 30 model requests.

The rest came from smaller changes: owner-only forwarding with a note in memory, read receipts after a successful turn, and sticker shortcuts bound to what the model was shown. About 100 lines of dead code went out of the same two files.

The checkpoint format, the size estimator and the memory-chunking rules went into separate small modules with their own tests. The orchestration itself stayed in the turn loop, because it needs the loop's state.

### Next structural step

The turn loop now mixes three jobs: the event loop itself, the lifecycle (shutdown and recovery), and the context policy (budget and memory saving). The plan is to split it along those lines. I postponed this on purpose: the loop is the riskiest part of the system, and I want the sandbox and the tests to cover enough of its behaviour first, so that the split can be checked against a baseline.

## What I Learned

- Removing code is safest when it starts from a snapshot and a dependency map, not from memory of how the code works.
- A second, independent reviewer catches a different class of problems than the author. Checking every finding against the code keeps the fix list honest.
- Disabled features are not free: they still cost build time, configuration, prompt tokens and attention.
- Deleting a library can reveal what quietly depended on it. That is a reason to delete carefully, not a reason to keep the library.
- "Smaller" is not the goal by itself. The useful measure is what the code does per line, and whether its behaviour can be tested and repeated.
