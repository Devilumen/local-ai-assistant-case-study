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

## Intonation and Pronunciation

A voice assistant needs more than a recognizable voice: pauses, emphasis, and intonation also affect how naturally a phrase is perceived.

Russian pronunciation and stress received particular attention. During testing, stress marking was tried on difficult words and names, but its reliability has not been established. Correct pronunciation remains an area for further evaluation.

English speech is also a requirement for the voice interface. Its quality and consistency in following the voice reference need separate evaluation.

## Vocal Reactions

Short vocal reactions, such as acknowledgment or dissatisfaction, were tested alongside ordinary speech.

Some variants sounded appropriate for the assistant's character. Others were less consistent: in individual trials, a reaction sounded different from what was expected or included minor artifacts.

The current scenario uses only variants selected after listening. They are treated as **individual vocal reactions**, not as guaranteed control over the emotion of an entire utterance.

## Telegram Integration

The speech engine was connected to the assistant as a local service. During a conversation, the assistant can select an available tool, prepare the words for a voice response, and start synthesis.

After generation, the audio is converted into a format suitable for Telegram voice messages and sent to the chat. Longer text can be split into consecutive voice messages.

The end-to-end workflow was tested in a real conversation:

1. The assistant received a Telegram message.
2. It prepared text for a voice response.
3. It started local speech generation.
4. It sent the resulting voice messages.

Delivery and playback were confirmed during testing.

## Resource Management

One of the project constraints was finding a voice solution that combines **high-quality sound with relatively modest resource consumption** and can be used alongside a local language model.

The evaluation covered running language and speech workloads together and loading the speech engine on demand. On-demand loading helps release resources after generation, although it can add delay before a voice response.

## My Role

I defined the voice requirements and its role in the assistant's conversations, compared synthesis options, and evaluated the results by listening.

I also tested vocal reactions, helped choose the model-loading approach, and evaluated the speech engine alongside the language model. Integration changes and debugging were performed using AI tools; the results were checked in the running system and in a real Telegram conversation.

## What I Learned

- Voice cloning should be evaluated for both reference similarity and natural speech across different phrases.
- Intonation, stress, and the pronunciation of names significantly affect how a voice is perceived.
- Vocal reactions can make interaction more expressive, but may not sound consistent across generations.
- Generating an audio file does not prove the whole workflow works: conversion, delivery, and playback also need testing.
- A local voice assistant requires balancing speech quality with the resources needed by the language model.

## Limitations and Future Work

- Evaluate Russian and English speech across a wider range of text.
- Assess the consistency of reference similarity and natural intonation across utterances of different lengths.
- Test the pronunciation of names, numbers, and words with difficult stress patterns.
- Repeat vocal-reaction tests to assess consistency.
- Measure the delay from receiving a message to delivering a voice response over a series of requests.
- Continue exploring how conversation context can influence vocal reactions and speaking style.
