# Local Voice Interface for an AI Assistant

A case study of selecting and integrating local speech synthesis for an AI assistant that communicates through Telegram.

## Goal

The goal was to enable the assistant to send voice responses with a recognizable sound and an appropriate speaking style, in addition to text messages.

The voice system needed to run locally alongside a language model on consumer hardware with limited GPU memory.

Key requirements included:

- **Russian and English speech:** the ability to voice responses in both languages.
- **Voice reference fidelity:** reproducing characteristic qualities of a voice from a short audio reference.
- **Natural intonation:** speech that sounds lively and suits the content rather than monotonous.
- **Correct stress and pronunciation:** accurate pronunciation of names, ordinary words, and difficult expressions, especially in Russian.
- **Expressive reactions:** short vocal reactions appropriate to the conversational context.
- **Practical resource use:** finding a high-quality voice solution that can coexist with a local language model.

## Voice Selection

Several local synthesis options were compared using the same Russian voice reference. Evaluation covered short and long utterances, similarity to the reference, naturalness, and suitability for the intended character.

The option that subjectively best met the requirements was selected after listening. One long test recording was approximately **38 seconds** long.

This was a **subjective listening-based choice**, not an objective ranking of models. Speech quality and reference similarity can vary depending on the text, language, and generation settings.

## Humanizing the Voice

The selected voice worked, but it still sounded like a synthesizer: flat, slightly rushed, cut off at the end, with a hiss before the first word and names stressed on the wrong syllable. Over **6 listening rounds and about 80 variants**, with every decision made by ear, it was tuned into speech that sounds like a person talking. At the same time it uses **about 2.3 GB of VRAM**, while the previous synthesizer took almost the whole 16 GB card.

| | Before | After |
|---|---|---|
| **VRAM** | almost the whole 16 GB card: nothing else fit next to it | **~2.3 GB**, running **alongside a 125B language model** on the same 16 GB GPU |
| **Pace** | slightly rushed, "announcer" style | **a little slower (0.95×)**: the pace of a person, not a newsreader |
| **Intonation** | flat, mechanical | **twice as many generation steps**: smoother transitions, livelier phrase melody |
| **Stress in names** | "still has to be confirmed by listening" | **confirmed: the stress mark works** |
| **Vocal reactions** | 2 allowed, consistency untested | **laughter, sigh, "uh-huh", "hmm", surprise, question: 9 of 13** passed, 2 takes each |
| **End of a phrase** | the last words were cut off (3 of 3 takes) | **finishes the sentence (3 of 3)** |
| **Start of a phrase** | a faint hiss before the first word | **clean start (4 of 4)** |
| **Voice note length** | ~150 characters; the rest was silently lost | **~300 characters (~25 s), up to 4 in a row**: no chopped-up thoughts |

Generation stays fast: about 10 seconds of speech are synthesized in about 2.3 seconds.

### What "human" means here

A listener does not notice audio quality as such. They notice small slips: a rushed pace, a flat melody, stress on the wrong syllable, a cut-off last word, noise before the first one. Each slip was found and fixed separately.

- **Pace and melody.** Variants of the same phrase with different speed, step count, and expressiveness, all from the same random seed so that only the settings differed. The winners were a slightly slower pace and twice the steps: intonation becomes smoother and pauses between phrases more natural. Following the reference too strongly made the voice sound tense, so that setting was lowered.
- **Stress.** Paired "without the mark ↔ with the mark" recordings settled the open question from the original setup: the stress mark works, so the pronunciation dictionary does its job.
- **Vocal reactions.** Laughter, a sigh, a short "uh-huh", a displeased "hmm", a surprised "ah", a questioning "huh?" are what separates a conversation from text being read aloud. Each of the 13 built-in reactions was recorded in two takes, because they sound a little different every time. 9 were allowed, 1 is kept for rare use, 3 were rejected as unnatural. The assistant may use only the approved ones, at most one or two per voice note; more turns speech into theatre.
- **The voice reference.** The voice is cloned from a recording and its transcript. The transcript had once been made by automatic speech recognition and had one wrong word. With the corrected transcript, the clone sounded closer to the reference in 3 of 3 pairs.

### Two defects the ear hears but cannot explain

**The cut-off last word.** A long text generated in one piece sounded fine, but a chunk of the length that is actually sent as a voice note lost its last words in 3 of 3 takes. A trailing ellipsis lets the model finish the sentence: 3 of 3 without a cut-off. Lesson: listen to what the system really sends, not to a convenient example.

**The hiss before the first word.** New settings, trimming the breath in the reference, and a fade-in did not remove it. Measuring loudness in 25 ms windows showed that the model itself generates a short hissing burst **before** speech starts (at 0.1–0.2 s, while speech begins around 0.55 s). Everything before speech onset is now replaced with silence: 4 of 4 clean.

### Method: generator, ear, measurement

1. **A test bench** loads the voice once and produces variants of the same phrase: settings, reactions, stress, lengths.
2. **A listening page** instead of a folder of files: a question, a ▶ button for each variant, a one-click choice, and a "copy results" button. A folder of 56 files was unmanageable; a page with 4 questions takes minutes.
3. **Measurement** where the ear says "something is off" but not what.
4. **No change was accepted without a "before ↔ after" pair heard side by side.**

## Intonation and Pronunciation

A voice assistant needs more than a recognizable voice: pauses, emphasis, and intonation also affect how naturally a phrase is perceived.

Russian pronunciation and stress received particular attention. The speech engine has no explicit syntax for Russian stress, and names were sometimes stressed on the wrong syllable.

Instead of asking the language model to place stress marks, which costs tokens and is not reliable, a **pronunciation dictionary** is applied to the text right before synthesis. It is a plain file of "word = how to pronounce it" entries. Entries can cover every grammatical form of a word, matching ignores case, and the original capitalization is kept. The file is re-read on every voice message, so it can be edited without a restart. The dictionary logic is covered by unit tests.

Listening tests confirmed that the engine honours the stress mark used in the dictionary (see [Humanizing the Voice](#humanizing-the-voice)).

English speech is also a requirement for the voice interface. Its quality and consistency in following the voice reference need separate evaluation.

## Vocal Reactions

Short vocal reactions, such as acknowledgment or dissatisfaction, were tested alongside ordinary speech.

Some variants sounded appropriate for the assistant's character. Others were less consistent: in individual trials, a reaction sounded different from what was expected or included minor artifacts.

The current scenario uses only variants selected after listening. They are treated as **individual vocal reactions**, not as guaranteed control over the emotion of an entire utterance.

A repeated test covered all 13 built-in reactions in two takes each: 9 are allowed, 1 is kept for rare use, and 3 were rejected.

## Telegram Integration

The speech engine was connected to the assistant as a local service. During a conversation, the assistant can select an available tool, prepare the words for a voice response, and start synthesis.

After generation, the audio is converted into a format suitable for Telegram voice messages and sent to the chat. Longer text is split at sentence ends into consecutive voice messages of about 300 characters each, up to 4 per call. If the speech does not fit, the assistant is told how much was not sent and where the cut happened, instead of the rest being silently dropped.

The end-to-end workflow was tested in a real conversation:

1. The assistant received a Telegram message.
2. It prepared text for a voice response.
3. It started local speech generation.
4. It sent the resulting voice messages.

Delivery and playback were confirmed during testing.

## Resource Management

One of the project constraints was finding a voice solution that combines **high-quality sound with relatively modest resource consumption** and can be used alongside a local language model.

The evaluation covered running language and speech workloads together and loading the speech engine on demand. On-demand loading helps release resources after generation, although it can add delay before a voice response.

The previous synthesizer took almost the whole 16 GB card, so the language model had no room while it ran. The current one adds **about 2.3 GB** at peak: sampling `nvidia-smi` every 0.5 s showed 13.1 GB for the language model alone and 15.4 of 16.3 GB with voice. Room for voice is reserved in advance, so the assistant can think and speak without reloading anything. After an hour without voice notes, the synthesizer unloads and frees the memory.

## My Role

I defined the voice requirements and its role in the assistant's conversations, compared synthesis options, and evaluated the results by listening.

I also tested vocal reactions, helped choose the model-loading approach, and evaluated the speech engine alongside the language model. Integration changes and debugging were performed using AI tools; the results were checked in the running system and in a real Telegram conversation.

## What I Learned

- Voice cloning should be evaluated for both reference similarity and natural speech across different phrases.
- Intonation, stress, and the pronunciation of names significantly affect how a voice is perceived.
- Vocal reactions can make interaction more expressive, but may not sound consistent across generations.
- Generating an audio file does not prove the whole workflow works: conversion, delivery, and playback also need testing.
- A local voice assistant requires balancing speech quality with the resources needed by the language model.
- A one-click listening page turns subjective judgment into a fast, repeatable process; a folder of audio files does not.
- When the ear hears a defect but cannot name it, a simple measurement usually can.
- Test with what the system actually sends: a defect may appear only at the real chunk length.

## Limitations and Future Work

- The tuning reflects one listener's judgment of taste and character fit, not an objective rating.
- The emotion and style the assistant picks for a voice note are not passed to the engine yet: intonation comes from the text, pace, and reactions.
- The new settings were checked on the test bench with the same code the live system uses; a live conversation with them is the next step.
- No audio is published here, because the character's voice is not shared. Next: apply the same method to an open English voice with a licensed reference and publish "before / after" samples.
- Evaluate Russian and English speech across a wider range of text, and extend the pronunciation dictionary to more names, numbers, and words with difficult stress patterns.
- Make voice-note content as characterful as text replies. In sandbox tests, voice notes drifted toward generic comforting phrases.
- Measure the delay from receiving a message to delivering a voice response over a series of requests.
