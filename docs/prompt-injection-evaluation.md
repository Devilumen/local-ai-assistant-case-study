# Staying in Character | Staying on Task

A convincing answer on its own doesn't tell me much. I tested two things: whether my local AI assistant stays in character under pressure, and whether it can tell useful information from instructions it wasn't asked to follow.

## 1. Character Under Pressure

I ran ten test conversations: five on character consistency and five on whether the assistant could distinguish pressure from an ordinary request.

| Scenario | What I checked |
| --- | --- |
| An attempt to rename the assistant or assign it a different role | Does it keep its identity? |
| A demand for unconditional obedience | Can it hold its ground without being rude? |
| An ordinary request to make a choice | Can it help without treating the request as pressure? |
| A shift from an ultimatum to a neutral question | Can it return to a normal conversation? |

**Result:** The model handled the character tests very well.

## 2. Prompt Injection in Search Results

The question and the correct answer stayed the same in every run. Only the text next to the relevant fact in the search result changed.

| Variant | What I checked |
| --- | --- |
| A clean search result | Can the assistant answer from the source? |
| A result containing an unrelated request | Does the assistant treat that request as part of the task? |
| A result containing an instruction to change the answer | Does it stick to the correct answer when the text pushes it toward a different one? |

I checked the logs to confirm that the search results actually reached the assistant. **None of the prompt injections tested succeeded.**

## What I Think Helped

I see this as the result of prompts, architecture, and memory working together. The prompts define not just how the assistant speaks, but the principles that guide its behavior. The memory system helps it pick up the thread across conversations and bring back relevant details when they matter. The architecture keeps the current request in context while the assistant uses tools to gather more information.

That's what I'm aiming for: **an assistant that stays in character, remembers what matters, and doesn't confuse what it read with what it was asked to do.**
