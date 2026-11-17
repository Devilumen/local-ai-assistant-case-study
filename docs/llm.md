# Local LLM Deployment and Evaluation

This document expands on the language-model deployment described in the [main case study](../README.md). It records a local experiment, not a general performance benchmark.

## Goal

Run a quantized conversational model locally alongside speech synthesis on a consumer GPU, while retaining enough context for an assistant that uses memory and tools.

The evaluation focused on instruction following, conversational style, tool use, resource consumption, and handling long context.

## Test Configuration

| Parameter | Setting |
| --- | --- |
| GPU memory | 16 GB |
| Model size | 27B parameters |
| Model weights | Quantized |
| Context window | 65,536 tokens |
| KV cache | Q8 |
| Model placement | 50 GPU-offloaded layers as counted by the inference engine, including the output layer; remaining layers on CPU |
| Interface | OpenAI-compatible API |
| Additional model reasoning | Compared in separate chat tests; disabled in the integrated run |
| Speech synthesis | Local, tested alongside the language model |

The context window is the configured capacity, not a guarantee of reliable retrieval or answer quality at every position within it.

## Tests and Observations

- Tested dialogue and instruction following, including consistency of conversational style.
- Compared responses with additional reasoning enabled and disabled. Some responses were overly brief or used repetitive phrasing.
- Tested tool use in the integrated Telegram workflow and confirmed delivery of a generated voice message.
- Sent a synthetic request containing approximately 60,000 input tokens while speech was generated separately.
- During the combined long-context and voice test, sampled GPU memory usage peaked at 15,604 MiB out of 16,303 MiB.

The memory figure represents a sampled observation from one test, not a guaranteed maximum. The limited remaining headroom means that other GPU workloads, longer voice messages, or different input patterns may lead to memory pressure or failure.

## Context Overflow

An earlier request exceeded the configured context size. The application repeatedly retried the rejected request and appended additional instructions, making the request larger.

Changes were made to bound retries, stop on HTTP errors, trigger memory saving during a tool sequence, and preserve temporary context when summaries are empty or truncated. Fourteen application-component tests passed after the changes, including a regression test for repeated empty responses.

The current threshold uses token usage reported by the previous successful response. It does not yet perform an exact preflight token count for every new request, and the complete automatic compaction workflow has not been validated across all long-running conversations.

## Future Work

- Add accurate preflight token accounting.
- Evaluate response quality and tool selection across longer conversations.
- Explore ways to reduce memory pressure when language and speech workloads overlap.
- Publish a separate reproducible latency and GPU-memory benchmarking tool.
