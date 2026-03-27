# MediPal

**Your AI-powered medical assistant for overcoming all barriers to quality healthcare.**

[View the Pitch Deck](https://emcee.github.io/MediPal/)

## Inspiration

MediPal was born from personal travel experiences. Running out of medication in a foreign country where you don't speak the language is frustating, stressful and potentially dangerous. We wanted to build something that could bridge that gap — a tool that doesn't just translate words, but *interprets* medical context so you get the right medication and assistance, not just a rough translation.

## What It Does

MediPal is a personal agent that acts as real-time bridge between you and a pharmacist/doctor/waiter abroad. You store your medications and health profile, and when you walk into a pharmacy or hospital, MediPal:

- **Interprets** your medical needs to the pharmacist/doctor in their language — with voice output so you can simply hand over your phone
- **Surfaces** active compounds, drug classes, and equivalents so they understand exactly what you need
- **Translates** their response back to you, flagging safety concerns like dosage mismatches, drug class differences, or interactions

## How We Built It

We ideated the concept together as a team, then split into focused roles: one teammate handled the **presentation and project administration** (scoping, slide deck, demo scripting, documentation), while the other drove the **technical implementation** (architecture, API integrations, data engineering).

Our stack:
- **Nanobot** as base agentic framework (https://github.com/HKUDS/nanobot)
- **Telegram Bot API** for a zero-install, globally accessible interface
- **OpenCode API** for multilingual medical reasoning and interpretation
- **ElevenLabs** for natural text-to-speech in the pharmacist's language
- **Python** tying it all together

## Challenges We Ran Into

- **The agent prompting beyond our scope:** The agent would sometimes go into lengthy explanations that were outside the role of our interpreter. Reining the agent in to strictly interpret and surface stored profile data — without overstepping — took careful prompt engineering and iteration.

- **Speaking in two languages in the same prompt:** Getting the model to produce clean, well-separated output in two different languages (e.g., English for the user and Thai for the pharmacist) within a single response was tricky. The languages would bleed into each other, or the model would default to one language for both sections. 

- **Voice latency:** Generating voice via the ElevenLabs API and delivering it as a Telegram voice note introduced noticeable delay in the conversation flow. For a real-time pharmacy interaction, every second counts. 

## What We Learned

We learned how to integrate voice capabilities with an AI agent — something neither of us had worked with before. Combining text-based LLM reasoning with speech synthesis into a coherent, real-time conversational flow was a new challenge that pushed us to think about UX beyond just text on a screen. We also gained a deeper appreciation for prompt engineering as a craft — small changes to the system prompt had outsized effects on the quality and safety of the bot's output.

## What's Next for MediPal

- Auto-Research on diet and alternative treatments
- OCR on pill packaging for instant identification
- Ingredient scanning for food safety abroad
- Official Health Records Integration
- Multi-Party Sessions & Personal Health Node
- Group chats with family doctor and local pharmacist