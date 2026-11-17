# Local AI Assistant: Behavior, Memory, and Voice

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
- Comparing modes with additional model reasoning enabled and disabled.

The model could maintain the intended style and use tools, but quality depended on context, settings, and instruction wording. Some replies were overly brief or repeated similar phrasing. Work on naturalness and behavioral consistency continues.

Here, **decision-making** means selecting a response or an available action based on the context and instructions provided to the model.

## Memory of Conversation Partners

The assistant can store information about people and previous conversations, then search for related memories during interaction.

A personal-memory component was used in an earlier version of the project. Its presence and operation in the current version still need to be verified. Reliably associating memories with individual users and testing that information is not mixed between people remain open concerns.

## My Role

I defined the project goals and the intended behavior of the assistant, including its conversational style, interaction boundaries, approach to memory, and voice features.

I selected and evaluated models, configured local inference parameters, compared reasoning modes, and tested responses in real conversations. I also evaluated tool use and voice interaction.

I used AI tools to investigate errors and make integration changes, then checked the results in the running system.

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

## Voice and Multimodality

Local speech synthesis was integrated into the Telegram conversation workflow. Generated speech was converted into a format suitable for Telegram voice messages.

The end-to-end flow was tested in a real conversation: the assistant generated and successfully sent voice messages.

The project also includes work with speech-to-text and visual AI capabilities. Their current configurations require separate verification before performance results can be reported.

## Reliability Case: Context Overflow

During testing, the assistant stopped responding when a request exceeded the model's context limit. The application repeatedly sent the rejected request while adding new instructions, causing the context to grow further.

The investigation led to changes that:

- Stop repeated empty or failed model responses from creating an infinite loop.
- Trigger memory-saving operations before the configured context limit is reached.
- Avoid clearing temporary context when a memory summary is empty or truncated.
- Limit retries and preserve information for recovery.

After the changes, **14 application-component tests passed**, including a regression test for repeated empty responses.

## Observations and Limitations

During the combined long-context and speech-synthesis test, sampled GPU memory usage reached **15,604 MiB out of 16,303 MiB**.

This was one test under specific conditions, not a guarantee that every workload will fit. The remaining VRAM headroom was small, so longer voice messages, other GPU applications, or different workloads may require further adjustments.

The context-management threshold relies on token usage from the previous successful response rather than an exact preflight count before every new request.

**Access during testing:** The assistant was configured to interact with only one authorized user. Multi-chat behavior, separation of memories between people, and the full set of actions involving other users require separate testing.

## What I Learned

- Local AI deployment requires balancing model quality, latency, context size, and memory.
- A model responding correctly in a standalone chat does not guarantee reliable tool use inside an application.
- Assistant behavior needs evaluation across different situations, not just one successful conversation.
- Logs and repeatable tests are essential when several services interact.
- Persistent memory requires failure handling, not just a summarization prompt.
- Consumer hardware can support complex local AI workflows, but only with careful measurement and realistic limits.

## Next Steps

- Improve token counting before requests are sent.
- Continue improving conversational naturalness and reliable tool selection.
- Develop a mechanism for modeling the assistant's emotional state, influencing its conversational style and reactions according to dialogue context.
- Build a separate reproducible tool to measure local LLM latency and GPU memory usage.
