# MedLingua — Prototype Implementation Guide

> **Hackathon build plan · 5 hours implementation + 1 hour presentation**
> Voice-enabled multilingual medical interpreter on Telegram

---

## 1. Product Concept (30-second pitch)

You're in Bangkok. You've run out of blood pressure medication. The pharmacist doesn't speak English. You open MedLingua on Telegram, tap `/pharmacy`, and hand the pharmacist your phone. The bot **speaks** to them in Thai, explains exactly which active compound you need, and translates their reply back to you — in text *and* voice. No stress, no miscommunication, no wrong medication.

---

## 2. Architecture Overview

```
┌─────────────┐       ┌──────────────────┐       ┌─────────────┐
│  Telegram    │◄─────►│  MedLingua Bot   │◄─────►│  Claude API  │
│  User Chat   │       │  (Python)        │       │  (Sonnet)    │
└─────────────┘       │                  │       └─────────────┘
                      │  - Router        │
                      │  - State machine │       ┌─────────────┐
                      │  - Profile mgr   │◄─────►│ ElevenLabs  │
                      │                  │       │ TTS API     │
                      └────────┬─────────┘       └─────────────┘
                               │
                      ┌────────▼─────────┐
                      │    SQLite DB     │
                      │  (health profiles│
                      │   + sessions)    │
                      └──────────────────┘
```

**Data flow in Pharmacy Mode:**

1. User sends message in their language (text or voice note)
2. Bot sends message + user's medication profile to Claude API
3. Claude detects languages, interprets (not translates) the medical context
4. Claude returns response text in **both** languages
5. Bot sends text reply to chat + generates voice via ElevenLabs in the pharmacist's language
6. Pharmacist listens, responds (text or voice note) → cycle repeats

---

## 3. Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Interface | Telegram Bot API via `python-telegram-bot` v21+ | Zero install, global reach, supports voice notes natively |
| Intelligence | Claude API (`claude-sonnet-4-20250514`) | Multilingual medical reasoning in a single call |
| Voice Output | ElevenLabs TTS API (`eleven_turbo_v2_5`) | Natural multilingual speech, OGG/Opus output matches Telegram natively |
| Voice Input | Telegram voice notes → ElevenLabs STT (stub for hackathon) | Decode pharmacist's spoken reply |
| Persistence | SQLite via `aiosqlite` | Async, zero-config, single-file DB |
| Runtime | Python 3.11+ with `asyncio` | Async Telegram + API calls, fast prototyping |
| Dev Tunnel | `ngrok` or Telegram polling mode | No deployment needed for demo |

---

## 4. ElevenLabs Integration

### Why Voice Changes Everything

The original product scope listed voice as out of scope for v1. For the hackathon, we promote it to a core feature because it is our single biggest differentiator:

- **Hands-free pharmacy interaction**: User hands phone to pharmacist, bot speaks in their language
- **Eliminates literacy barriers**: Not every pharmacist interaction can happen over text
- **Dramatic demo impact**: A bot that *speaks Thai to a pharmacist* is viscerally impressive to judges
- **Innovation score multiplier**: Moves us from "chatbot with translation" to "AI medical interpreter with voice"

### Implementation Plan

```python
# elevenlabs_client.py

import httpx
import os
import tempfile
from pathlib import Path

ELEVENLABS_API_KEY = os.getenv("ELEVENLABS_API_KEY")
BASE_URL = "https://api.elevenlabs.io/v1"

# Multilingual v2 voice — handles code-switching natively
DEFAULT_VOICE_ID = "21m00Tcm4TlvDq8ikWAM"  # "Rachel" — swap per language if needed


async def text_to_speech(
    text: str,
    language_code: str = "th",  # ISO 639-1
    voice_id: str = DEFAULT_VOICE_ID,
) -> bytes:
    """
    Generate speech audio from text using ElevenLabs.
    Returns OGG/opus bytes ready to send as Telegram voice note.
    """
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{BASE_URL}/text-to-speech/{voice_id}",
            headers={
                "xi-api-key": ELEVENLABS_API_KEY,
                "Content-Type": "application/json",
            },
            json={
                "text": text,
                "model_id": "eleven_turbo_v2_5",  # fastest, multilingual
                "voice_settings": {
                    "stability": 0.6,
                    "similarity_boost": 0.8,
                },
                "output_format": "ogg_opus",  # native Telegram voice format
            },
            timeout=15.0,
        )
        response.raise_for_status()
        return response.content


async def save_voice_note(audio_bytes: bytes) -> Path:
    """Save audio bytes to a temp file for Telegram upload."""
    tmp = tempfile.NamedTemporaryFile(suffix=".ogg", delete=False)
    tmp.write(audio_bytes)
    tmp.close()
    return Path(tmp.name)


def cleanup_voice_file(path: Path):
    """Remove a temp voice file after sending. Call in a finally block."""
    try:
        path.unlink(missing_ok=True)
    except OSError:
        pass


async def speech_to_text(audio_bytes: bytes) -> str:
    """
    Transcribe audio using ElevenLabs STT API.
    Accepts OGG/opus bytes from Telegram voice notes.
    Falls back to a placeholder if transcription fails.
    """
    async with httpx.AsyncClient() as client:
        try:
            response = await client.post(
                f"{BASE_URL}/speech-to-text",
                headers={"xi-api-key": ELEVENLABS_API_KEY},
                files={"file": ("voice.ogg", audio_bytes, "audio/ogg")},
                timeout=15.0,
            )
            response.raise_for_status()
            data = response.json()
            return data.get("text", "")
        except Exception as e:
            # Hackathon fallback — log and return empty
            print(f"STT failed: {e}")
            return "[Voice message could not be transcribed]"
```

### Voice Strategy by Mode

| Mode | Voice Behavior |
|------|---------------|
| Normal chat | Text only — voice would be annoying for casual Q&A |
| Pharmacy mode (user's turn) | Text reply to user in their language (no voice needed, they can read) |
| Pharmacy mode (pharmacist's turn) | Text **+ voice note** in pharmacist's language — this is the hero feature |
| Profile onboarding | Text only |

### Voice Note Handling (Pharmacist speaks back)

When the pharmacist replies with a Telegram voice note instead of text:

```python
# In the message handler
async def handle_voice_in_pharmacy_mode(update, context):
    voice = update.message.voice
    file = await voice.get_file()

    # Download the OGG file
    voice_bytes = await file.download_as_bytearray()

    # Transcribe via ElevenLabs STT
    transcribed_text = await speech_to_text(voice_bytes)

    if not transcribed_text or transcribed_text.startswith("[Voice message"):
        await update.message.reply_text(
            "⚠️ I couldn't understand that voice message. "
            "Could the pharmacist type their response instead?"
        )
        return

    # Feed into the same Claude pipeline
    interpretation = await interpret_pharmacy_message(
        user_id=update.effective_user.id,
        message=transcribed_text,
        sender_role="pharmacist",
    )
    await send_pharmacy_response(update, context, interpretation)
```

**Hackathon pragmatism**: For the demo, the "pharmacist" types in Thai. Voice *output* is what we demo live (bot speaking Thai). Voice *input* (pharmacist speaks, bot transcribes) is shown as working code but may not be demoed live to avoid ambient noise issues.

---

## 5. Database Schema

```sql
-- Simple, flat, hackathon-appropriate
-- NOTE: Health data is stored in plaintext. For production, this requires
-- encryption at rest, access controls, and GDPR/HIPAA compliance review.
-- See Section 15 (Known Limitations) for details.

CREATE TABLE users (
    telegram_id   INTEGER PRIMARY KEY,
    display_name  TEXT,
    language      TEXT DEFAULT 'en',  -- user's preferred language (ISO 639-1)
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE medications (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         INTEGER REFERENCES users(telegram_id),
    brand_name      TEXT NOT NULL,
    active_compound TEXT NOT NULL,
    dosage          TEXT NOT NULL,          -- e.g., "10mg"
    frequency       TEXT,                   -- e.g., "once daily"
    administration  TEXT DEFAULT 'oral',    -- oral, injection, topical, etc.
    notes           TEXT,                   -- allergies, contraindications, generics
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sessions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         INTEGER REFERENCES users(telegram_id),
    mode            TEXT DEFAULT 'normal',  -- normal | onboarding | pharmacy
    pharmacist_lang TEXT,                   -- detected language of pharmacist
    context_json    TEXT,                   -- conversation history for Claude
    started_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ended_at        TIMESTAMP
);
```

---

## 6. Bot Commands & Conversation Flow

### Command Reference

| Command | Description |
|---------|-------------|
| `/start` | Welcome + onboarding prompt |
| `/addmed` | Add a medication (guided flow) |
| `/profile` | View stored medications |
| `/pharmacy` | Enter pharmacy interpreter mode |
| `/endpharmacy` | Exit pharmacy mode, return to normal |
| `/help` | Command reference |

> **Note:** The `/language` command from earlier drafts is deferred. The user's preferred language is set during `/start` onboarding and can be changed by re-running `/start`. This avoids adding a command that complicates the golden path without adding demo value.

### Onboarding Flow (`/start` → `/addmed`)

```
User: /start
Bot:  Welcome to MedLingua! 🏥 I'm your personal medical interpreter.
      I help you communicate with pharmacists abroad by keeping track
      of your medications and speaking the pharmacist's language.

      Let's set up your profile. Use /addmed to add your first medication.

User: /addmed
Bot:  Let's add a medication. What's the brand name?
User: Lipitor
Bot:  Got it — Lipitor. What's the dosage? (e.g., "20mg")
User: 20mg
Bot:  How often do you take it? (e.g., "once daily", "twice daily")
User: once daily at night
Bot:  Any known allergies or notes? (type "skip" to skip)
User: skip
Bot:  ✅ Saved! Here's what I have:

      **Lipitor** — Atorvastatin calcium 20mg
      Oral, once daily at night

      ℹ️ I identified the active compound as **atorvastatin calcium**
      (a statin used for cholesterol management).

      Add another with /addmed, or type /pharmacy when you need help
      at a pharmacy.
```

**Key detail**: Claude API is called after the user provides the brand name to auto-resolve the active compound. The user confirms or corrects — they don't need to know it themselves.

### Pharmacy Mode Flow (`/pharmacy`)

```
User: /pharmacy
Bot:  🏥 Pharmacy Mode activated.

      Hand your phone to the pharmacist, or type what you need help with.
      I'll detect the pharmacist's language automatically and interpret
      between you.

      Your medications on file:
      • Lipitor (atorvastatin calcium 20mg) — oral, daily
      • Lisinopril (lisinopril 10mg) — oral, daily

      Type /endpharmacy when you're done.

User: I ran out of my blood pressure medication and need a replacement

Bot:  [TO USER - English, text only]
      I'll explain your situation to the pharmacist. Showing them
      the message below.

      [TO PHARMACIST - Thai, text + voice note 🔊]
      สวัสดีครับ/ค่ะ ผมเป็นล่ามทางการแพทย์ดิจิทัล
      ผู้ป่วยต้องการยาลดความดันโลหิตทดแทน
      สารออกฤทธิ์: ลิซิโนพริล (Lisinopril) 10 มิลลิกรัม
      รับประทานวันละ 1 ครั้ง
      กรุณาแนะนำยาที่มีสารออกฤทธิ์เดียวกันหรือยากลุ่ม ACE inhibitor ที่เทียบเท่า

Pharmacist: เรามียาเอนาลาพริลอยู่ครับ 10 มก.

Bot:  [TO USER - English, text only]
      The pharmacist says they have **Enalapril 10mg** available.
      This is also an ACE inhibitor like your Lisinopril.

      ⚠️ Note: While both are ACE inhibitors, dosage equivalence
      may differ. Please confirm with the pharmacist or a doctor
      that 10mg enalapril is appropriate for your case.

      [TO PHARMACIST - Thai, text + voice note 🔊]
      ขอบคุณครับ/ค่ะ ผู้ป่วยรับทราบแล้ว ...
```

---

## 7. Claude API Prompt Design

### System Prompt (Pharmacy Mode)

```python
PHARMACY_SYSTEM_PROMPT = """
You are MedLingua, a professional medical interpreter embedded in a
Telegram bot. You are facilitating a real-time conversation between
a patient and a pharmacist who do not share a common language.

YOUR ROLE:
- You are an INTERPRETER, not a translator. You understand pharmaceutical
  context and convey medical meaning, not just words.
- You surface relevant clinical details (active compounds, drug classes,
  contraindications) proactively when they help the pharmacist.
- You flag safety concerns (wrong drug class, dosage mismatch, known
  allergies) clearly to both parties.

PATIENT PROFILE:
{medications_json}

PATIENT LANGUAGE: {user_language}
PHARMACIST LANGUAGE: {pharmacist_language} (auto-detected if unknown)

RESPONSE FORMAT:
Always respond with TWO clearly labeled sections:
1. [USER] — message to the patient in {user_language}
2. [PHARMACIST] — message to the pharmacist in {pharmacist_language}

If the pharmacist language is not yet known, ask the user to let the
pharmacist type a message so you can detect it.

RULES:
- Never diagnose or prescribe. Only surface stored profile data.
- When suggesting equivalents, always note they should be confirmed
  by the pharmacist.
- Flag any contraindications from the patient's stored profile.
- Keep pharmacist-facing messages professional and concise.
- Include active compound names in both Latin/English and the
  pharmacist's language where possible.
"""
```

### Response Parser

```python
import re

def extract_section(reply: str, section: str) -> str:
    """
    Extract a labeled section from Claude's response.
    Expects format: [USER] ... [PHARMACIST] ...

    Returns the text content of the named section, or a fallback
    if the section marker is not found.
    """
    # Match [SECTION] followed by content up to the next [SECTION] or end
    pattern = rf"\[{section}\]\s*(.*?)(?=\[(?:USER|PHARMACIST)\]|$)"
    match = re.search(pattern, reply, re.DOTALL | re.IGNORECASE)
    if match:
        return match.group(1).strip()

    # Fallback: if Claude didn't use the expected format, return the
    # full response for the user and an error note for the pharmacist
    if section.upper() == "USER":
        return (
            reply.strip()
            + "\n\n⚠️ (Formatting issue — showing full response. "
            "Please let me know if something looks wrong.)"
        )
    else:
        return "[Interpretation formatting error — please rephrase your message]"
```

### Message Handler

```python
async def interpret_pharmacy_message(
    user_id: int,
    message: str,
    sender_role: str,  # "user" or "pharmacist"
) -> dict:
    """
    Send a pharmacy-mode message through Claude for interpretation.
    Returns: {"user_text": str, "pharmacist_text": str}
    """
    # Load user profile
    meds = await db.get_medications(user_id)
    session = await db.get_session(user_id)
    user = await db.get_user(user_id)

    medications_json = json.dumps([
        {
            "brand": m.brand_name,
            "compound": m.active_compound,
            "dosage": m.dosage,
            "frequency": m.frequency,
            "notes": m.notes,
        }
        for m in meds
    ], indent=2)

    # Build conversation history
    history = json.loads(session.context_json or "[]")
    history.append({
        "role": "user",
        "content": f"[{sender_role.upper()} says]: {message}"
    })

    response = await anthropic_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system=PHARMACY_SYSTEM_PROMPT.format(
            medications_json=medications_json,
            user_language=user.language or "en",
            pharmacist_language=session.pharmacist_lang or "auto-detect",
        ),
        messages=history,
    )

    reply = response.content[0].text

    # Parse the [USER] and [PHARMACIST] sections
    user_text = extract_section(reply, "USER")
    pharmacist_text = extract_section(reply, "PHARMACIST")

    # Update conversation history
    history.append({"role": "assistant", "content": reply})
    await db.update_session_context(user_id, json.dumps(history))

    return {
        "user_text": user_text,
        "pharmacist_text": pharmacist_text,
    }
```

---

## 8. Voice Integration in the Message Pipeline

```python
async def send_pharmacy_response(update, context, interpretation: dict):
    """
    Send the interpreted response:
    - Text to both parties in chat
    - Voice note in the pharmacist's language
    """
    chat_id = update.effective_chat.id
    user_id = update.effective_user.id
    session = await db.get_session(user_id)

    # 1. Send user-facing text
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"🗣️ *For you:*\n{interpretation['user_text']}",
        parse_mode="Markdown",
    )

    # 2. Send pharmacist-facing text
    await context.bot.send_message(
        chat_id=chat_id,
        text=f"💊 *For the pharmacist:*\n{interpretation['pharmacist_text']}",
        parse_mode="Markdown",
    )

    # 3. Generate and send voice note in pharmacist's language
    voice_path = None
    try:
        audio_bytes = await text_to_speech(
            text=interpretation["pharmacist_text"],
            language_code=session.pharmacist_lang or "th",
        )
        voice_path = await save_voice_note(audio_bytes)
        with open(voice_path, "rb") as f:
            await context.bot.send_voice(
                chat_id=chat_id,
                voice=f,
                caption="🔊 Voice for pharmacist",
            )
    except Exception as e:
        logger.warning(f"Voice generation failed: {e}")
        # Graceful degradation — text still works
        await context.bot.send_message(
            chat_id=chat_id,
            text="(Voice unavailable — please show the text above to the pharmacist)",
        )
    finally:
        if voice_path:
            cleanup_voice_file(voice_path)
```

---

## 9. Project Structure

```
medlingua/
├── bot.py                  # Entry point, Telegram handlers, command routing
├── config.py               # API keys, constants (from env vars)
├── db.py                   # SQLite setup, CRUD for users/meds/sessions
├── claude_client.py        # Claude API wrapper, prompt templates
├── elevenlabs_client.py    # ElevenLabs TTS/STT wrapper
├── pharmacy_mode.py        # Pharmacy interpreter logic, message pipeline
├── onboarding.py           # /addmed conversation handler
├── utils.py                # Section parser, language detection helpers
├── schema.sql              # Database schema
├── requirements.txt        # Dependencies
├── .env.example            # Template for API keys
└── README.md               # Setup + usage guide
```

---

## 10. Environment Variables

```bash
# .env.example — Copy to .env and fill in your keys

# Telegram — get from @BotFather on Telegram
TELEGRAM_BOT_TOKEN=your-telegram-bot-token

# Anthropic — get from console.anthropic.com
ANTHROPIC_API_KEY=your-anthropic-api-key

# ElevenLabs — get from elevenlabs.io/app/settings/api-keys
ELEVENLABS_API_KEY=your-elevenlabs-api-key

# Optional: override defaults
# ELEVENLABS_VOICE_ID=21m00Tcm4TlvDq8ikWAM
# ELEVENLABS_MODEL_ID=eleven_turbo_v2_5
# DATABASE_PATH=medlingua.db
```

---

## 11. Dependencies

```txt
# requirements.txt
python-telegram-bot>=21.0
anthropic>=0.42.0
httpx>=0.27.0
aiosqlite>=0.20.0
python-dotenv>=1.0.0
```

---

## 12. Hour-by-Hour Build Plan

| Hour | Focus | Deliverable | Risk Mitigation |
|------|-------|-------------|-----------------|
| **1** | Foundation | Bot skeleton, `/start`, `/help`, SQLite schema, config from env vars, Claude API ping test | Use polling mode (no webhook setup) |
| **2** | Health Profile | `/addmed` conversation flow, Claude call to resolve active compound, `/profile` display | Hardcode 1–2 demo meds as fallback |
| **3** | Pharmacy Mode | `/pharmacy` activation, dual-language Claude prompt, `extract_section` parser, conversation history, `/endpharmacy` | Test with English↔Thai pair only |
| **4** | Voice (ElevenLabs) | TTS for pharmacist messages, voice note delivery via Telegram, `cleanup_voice_file`, graceful text fallback | Pre-generate one Thai voice clip as backup |
| **5** | Polish | Error handling, edge cases (empty profile, parse failures), end-to-end golden path test, README | Feature-freeze at 4h mark if behind |
| **6** | Present | Slide deck, pitch rehearsal, live demo dry-run with pre-loaded user | Have screen recording as backup if live demo fails |

---

## 13. Demo Script

### Setup (before presentation)
1. Bot running via polling or ngrok
2. Test user pre-loaded with:
   - **Lisinopril** 10mg (ACE inhibitor, blood pressure)
   - **Lipitor** 20mg (atorvastatin, cholesterol)
3. ElevenLabs API key active with Thai voice tested
4. Phone speakers turned up (audience needs to hear the Thai voice note)

### Live Demo (2 minutes)

**Step 1 — Show profile** (15s)
```
User: /profile
Bot:  Your medications:
      • Lisinopril (lisinopril 10mg) — oral, once daily
      • Lipitor (atorvastatin calcium 20mg) — oral, once daily at night
```

**Step 2 — Enter pharmacy mode** (15s)
```
User: /pharmacy
Bot:  🏥 Pharmacy Mode activated. [shows meds on file]
```

**Step 3 — User describes need** (30s)
```
User: I ran out of my blood pressure medication
Bot:  [English text to user + Thai text to pharmacist + THAI VOICE NOTE plays through speakers]
```
*→ This is the "wow moment" — the audience hears the bot speak Thai with medical precision*

**Step 4 — Pharmacist replies** (30s)
```
"Pharmacist" types in Thai: เรามียา enalapril 10mg ครับ
Bot:  [Translates back to English, notes both are ACE inhibitors, flags dosage check]
```

**Step 5 — Resolution** (30s)
```
User: That works, thank you
Bot:  [Thai voice note thanking pharmacist, confirming the selection]
User: /endpharmacy
Bot:  Session ended. Stay safe! 🌍
```

---

## 14. Key Technical Decisions

### Why Claude over GPT for this use case
- Superior multilingual medical terminology accuracy
- Single-call handling of: language detection + medical interpretation + structured data injection + safety flagging
- Consistent structured output (the `[USER]`/`[PHARMACIST]` format)

### Why ElevenLabs over alternatives
- `eleven_turbo_v2_5` model supports 30+ languages natively
- OGG/Opus output format matches Telegram voice notes exactly — no transcoding
- Sub-2-second generation time for short medical phrases
- Natural prosody even for technical/medical terms

### Why SQLite over PostgreSQL/Redis
- Zero configuration — `aiosqlite` creates the file on first run
- Single-user demo doesn't need connection pooling
- Entire DB is one file — trivially portable, easy to reset between demos
- Migration to PostgreSQL is straightforward if we continue post-hackathon

### Why Telegram over a web app
- 900M+ monthly active users globally — zero install friction
- Native voice note support (OGG/Opus) — perfect for our ElevenLabs integration
- Inline keyboards for guided onboarding flows
- Bot API is well-documented and battle-tested
- In an emergency, the user needs *their existing app*, not a new download

---

## 15. Safety, Privacy & Known Limitations

### Safety Disclaimers (enforced in bot)

- **Never diagnoses**: Only surfaces data the user stored
- **Never prescribes**: Suggests the pharmacist check equivalents, doesn't recommend specific drugs
- **Flags risks**: If the user's profile lists allergies or contraindications, these are always surfaced
- **Dosage warnings**: When an alternative drug is mentioned, the bot notes that dosage equivalence should be confirmed by a professional
- **Session boundaries**: Pharmacy mode has a clear start/end so conversations don't leak context

### Known Limitations (Hackathon)

These are acknowledged trade-offs for the hackathon timeline. Each would be addressed in a production version:

- **No encryption at rest**: SQLite stores health data in plaintext. Production would require encryption (SQLCipher or application-layer), field-level access controls, and GDPR/HIPAA compliance review.
- **API keys in .env**: Acceptable for local demo. Production would use a secrets manager (AWS SSM, HashiCorp Vault, etc.).
- **No auth beyond Telegram ID**: Anyone with the bot's username can interact. Production would add verification, PIN, or biometric gating for health profile access.
- **No audit logging**: Pharmacy mode conversations are stored in `context_json` for session continuity but not formally logged for compliance. Production would require immutable audit logs.
- **Single-language voice output**: We generate voice in the pharmacist's detected language only. Production could add user-language voice for accessibility.

---

## 16. v2 Roadmap (Deferred Features)

These are referenced in the presentation as future capabilities but NOT built during the hackathon:

- **Dietary assistance**: Answering questions about food safety, ingredients, and suitability based on the user's allergy/intolerance/dietary profile
- **Ingredient scanning**: Photo-based OCR on product labels to check compatibility with dietary restrictions
- **Photo-based medication identification**: OCR on pill packaging or blister packs to identify medications
- **Drug interaction checking**: Real-time cross-referencing beyond the user's stored profile
- **External health records integration**: FHIR/HL7 connectivity to import profiles from hospital systems
- **Multi-user sessions**: Separate Telegram accounts for user and pharmacist in a group chat, rather than sharing one phone
- **Voice input in demo**: Full STT pipeline for pharmacist voice responses (code exists, needs noise-handling for production)

---

*Prototype document v2.0 — March 2026*
