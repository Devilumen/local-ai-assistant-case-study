# AI Writes. AI Audits. Human Decides.

*Two AI coding agents and one owner: 6 independent audits, 67 findings, 66 confirmed against the code, 0 false alarms.*

I develop my local AI assistant together with AI coding agents. The assistant runs entirely on a home computer: the language model, voice, speech recognition and memory. Almost all of the code in this project was written with AI involvement. The key question in this kind of work is not "can AI write the code" but "how do I know the code is right". This document describes the process I built and what it produced, in numbers. It complements the [main case study](../README.md).

## Roles

- **I am the product owner and architect.** I set the goals and make the technical decisions: which model and quantization to use, how to split video memory, what to measure and which results to trust. I decide which features are needed and how the assistant behaves. I approve every outward action: a push, a publication, a run in the live messenger. The assistant's character and instructions change only with my consent.
- **Implementer and tech lead — Claude Opus 5.5.** Writes and changes code, runs the tests and the sandbox, keeps the work log, checks other agents' findings against the code and writes the briefs for the auditor.
- **Independent auditor — GPT Astra 6.2.** Reads the code and the changes from a written brief. It changes nothing itself and only writes a report with findings and a risk rating.
- **Quality judges — GPT Sol 6.2 and one more model.** They score the assistant's answers blind when settings need to be compared. Details are in [sandbox evaluation](evaluation.md).

Different models have different blind spots. The implementer tends to trust its own code; an auditor that did not write it reads it without that presumption.

## Rules

1. **No outward action without my "yes".** A push, a publication, a run in the live messenger, installing a package: only after explicit consent, separately for each action.
2. **Private data stays closed.** The agents do not read the configuration with keys, the conversations or the assistant's memory. All tests run on synthetic data.
3. **Every finding is checked against the code before it is fixed.** The auditor's report is a hypothesis, not an order. The implementer finds the place in the code, confirms or refutes the finding and rates the risk for what is actually enabled.
4. **The owner decides about features.** When a finding comes down to "keep or remove a feature", the agent prepares the options with the cost of each, and I choose.
5. **Regression after changes.** Unit tests plus a reproducible sandbox run against a recorded baseline.
6. **A log entry after every task.** What was done, which errors were found, what was not verified and why. The log is the shared memory between agents and sessions.
7. **Self-review before the audit, but the auditor reads it after its own report.** The implementer first looks for holes in its own code and fixes what it finds. The auditor has access to that report, but the brief says explicitly: read it only after your own review. Otherwise the auditor starts to see the code through the author's eyes and confirms the author's conclusions instead of looking for its own.

## Why the Owner Has to Understand How Neural Networks Work

An AI agent writes code well, measures and proposes options. But it does not know what the product is willing to give up for speed, and it does not always tell its own confident hypothesis from a verified fact. To give the agent the right tasks and not accept a good-sounding explanation on faith, the owner has to understand the basics:
- tokens and the context window;
- quantization, video memory, the KV cache;
- temperature and sampling;
- reasoning modes;
- embeddings and memory search;
- how a model treats examples in its instructions;
- where LLM judges go wrong.

Here are decisions from this project that would have been made worse without that understanding.

| Situation | What had to be understood | How it was decided |
| --- | --- | --- |
| Answers are slow. Switch to a more "optimized" engine? | Wrappers around an engine do not make it faster. Speed depends on how much of the model is on the GPU, the quantization and the KV cache | Stayed on the same engine and found the real cause: 15 of 65 layers were on the CPU |
| Trim the model's vocabulary to two languages to save video memory | How the vocabulary and the embedding matrix work, and that changing the vocabulary means re-quantizing | Evaluated: small gain, high cost. Postponed until quantizations are compared |
| The voice puts the stress wrong in names | A model's answer is probabilistic and costs tokens. Fixed knowledge is more reliable applied deterministically | Instead of "let the model place the stress" — a pronunciation dictionary before synthesis |
| Which reasoning level to enable | What reasoning costs in tokens and time, and that one run on a small sample is weak evidence | "Low" was chosen from the data, with an explicit note about the sample size |
| Speculative decoding did not help | When guessing tokens pays off, and when the CPU eats the verification | The negative result was accepted and explained; recheck after the model fits on the GPU entirely |
| Who should judge quality | A judge scores against a rubric and does not need the strongest model. But a judge is biased too | The judge is one tier lower, the strong model stays on code review. I calibrated the judges against myself blind: I wrote down my choice before the key was revealed |
| The character became formulaic after the move to a local model | How model size affects improvisation and why a small model copies examples | I brought reference dialogues from a larger model. That separated two causes: "not enough model" and "the prompt gets in the way" |
| Speed things up by cutting the model down | Where software optimization ends and quality loss begins | First only software speed-ups without cutting anything; cut-down versions later, separately, in a comparison |

The agent took part in each of these decisions: it searched, measured and explained. But "what matters more to us" and "is this data enough" were decided by a person who understands what happens inside the model.

## The Loop

```
brief for the auditor → report with findings → check each against the code → fix
          ↑                                                                 ↓
      work log  ←  regression (tests + sandbox)  ←  risk rating for what is actually enabled
```

The brief for the auditor is a separate document: the scope (which commits to read), what is already known, what to focus on, the answer format with a risk scale, and the restrictions (read only, no live run, no private data).

## What It Produced: Six Reviews

| Review | Auditor's findings | Confirmed against the code | False |
| --- | --- | --- | --- |
| Code after memory stabilization | 11 | 10 (+1 not checked: the module was removed later) | 0 |
| Dead code | 9 | 9 | 0 |
| Regression after a large cleanup | 10 | 10 | 0 |
| Move to a new model and engine: start, stop, voice | 14 | 14 | 0 |
| Refactoring after the previous review | 12 | 12 | 0 |
| Core refactor, every step reviewed separately | 11 | 11 | 0 |
| **Total** | **67** | **66** | **0** |

In the fourth and fifth reviews the implementer ran its own review in parallel. Before the last one it found and fixed 6 problems itself; the auditor then found 12 more. In each of the two reviews there were three serious findings the implementer had missed.

Some examples.

- **Regressions that the tests could not see.** The core turn loop was refactored step by step under 52 golden traces that record every model request and memory write. All traces stayed green, yet the auditor found two real regressions at the first new function boundary: other events could slip in between the end of a turn and its finalisation, and a turn's temporary objects outlived its signals. Both were fixed and turned into new traces, recorded on the previous accepted version so that they describe the old behaviour.

- **"Checked, all fine" — and a hole.** The implementer introduced a rule: the API key goes only over HTTPS or to this same computer. In its self-review it wrote separately that the URL parsing had been checked and was fine. The auditor showed a URL like `http://evil.com?x=@localhost/`: the parser took it for localhost, and the key would have gone to a foreign server in clear text. A confident mistake, in exactly the place the author considered closed.
- **A process that outlives its stop.** The voice model is started through a virtual-environment launcher, and the real process with the model is its child. A stop on timeout killed only the launcher, so the process with the model could stay alive and hold 3 GB of video memory. In the first review the implementer had also judged this place safe. Now the bridge gets the PID of the model process itself and does not load a second copy while the first one is alive.
- **Stopping someone else's instance.** The engine stop script first called the general "stop" command and only then checked whether the process was ours. If our engine had already died and another one had been started from the same installation, the other one would have been stopped.
- **The implementer finds its own too.** In its self-review the implementer found the main bug in the new reminders: if the assistant's turn failed (the model was unavailable), the reminder was still marked as done and was lost. The auditor found the neighbouring one: if writing the mark failed, the reminder was sent again every 30 seconds. Together they closed both edges — "lose it" and "repeat it forever".
- **The auditor found the implementer's errors.** Two of the 11 findings in the first review were regressions the implementer itself had introduced. The "already fired" mark for an emergency notification was written before the notification was queued, so after a crash it was never repeated. The forwarding limit counted calls, not posts. An author usually does not notice such things.
- **Silent data loss in a stream.** A model response that arrived over the network in two pieces lost an event. A rare case that a test on a whole response does not catch.
- **A second stop signal cut off saving memory.** The second shutdown signal killed the process in the middle of saving. The fix: one signal and a wait of up to 30 minutes.
- **Dead code with a hidden dependency.** The audit found that the HTTP library being removed was the only provider of TLS for another library. After the removal the dependency is declared explicitly; the result is minus 3,555 lines.
- **Cross-checking both ways.** When some tasks were done by a second agent, the implementer reviewed its commits just as strictly. In one of them a working-memory cleanup removed lines about the emotional state that the instructions explicitly require to keep. It was caught and fixed before it affected behaviour.

Not every finding is fixed at once. The implementer ranks them by risk for what is enabled right now. For example, a leak between chats is critical when the assistant is allowed many chats, but unreachable in "owner only" mode. Such items are written down as a condition: "close before access is widened".

## What Did Not Work Right Away

- **The implementer is confidently wrong about its own code.** Twice in one day it wrote "checked, safe" where the auditor then found a serious hole. Self-review is useful — it found 6 problems — but it does not replace an independent review.
- **The auditor can read outdated documents.** In one review it referred to an old version of the project map. Now the brief names the commits and document versions explicitly.
- **Both agents have usage limits.** Long tasks have to be cut into parts and handed over between sessions through the log and written handover documents.
- **An AI judge does not replace a person on matters of taste.** When the assistant's character was scored, the judges agreed with me in only 3 and 2 pairs out of 5. So judges are used for repeatable scoring, and the decision on the character stays with me.

## What I Learned

- With AI agents, quality is set not by the model but by the process around it: rules, checks, and who makes the decisions.
- An independent auditor from another vendor pays off. Over six reviews — 66 confirmed findings out of 67, and not one false alarm.
- Self-review and independent review catch different things: the author sees well where the code does not do what was intended, the auditor sees where the intention can be bypassed. Both are needed, and the reading order matters: the auditor reads the author's report only after its own.
- "Check every finding against the code" protects in both directions: from the auditor's mistakes and from unnecessary changes made on its advice.
- The work log is not bureaucracy but the only way for the next agent or the next session not to start from zero.
- Outward actions must stay with the person. That is what makes it possible to give AI a lot of freedom inside the project.
- To manage AI agents, you need to understand neural networks no worse than they do. Otherwise you take their explanations on faith and set tasks at random.
