
# 🎧 Email Audio Agent

An AI agent that reads my Gmail, pulls out action items, and turns the briefing into an MP3 I can listen to. It runs automatically every morning at 7 AM.

![Architecture](email-audio-agent-architecture.png)

## What it does

1. **Reads** recent inbox emails with **read-only** Gmail access (`gmail.readonly`). It can't send or delete anything.
2. **Summarizes** them with the Gemini API (or Claude, optional) into a spoken-style summary plus a list of action items with sender, due date and priority.
3. **Speaks** the briefing with gTTS and saves it as `digests/YYYY-MM-DD.mp3`, alongside `.json` and `.txt` copies.

## Design notes

- **Resilience:** free APIs fail. Requests retry with backoff on 429/5xx, then fall back through a list of Gemini models if one is busy or unavailable.
- **Untrusted input:** an inbox is attacker-controlled text. The system prompt tells the model to treat emails as data, not instructions, and the agent only has read-only access, so the worst case is a bad summary.
- **Mock mode:** `--mock` runs the AI and audio steps on fake emails, so you can test without setting up Gmail.
- **Cost control:** each email body is trimmed to 4,000 characters, and up to 25 emails are read per run.

## Setup

**1. Install**

```bash
git clone https://github.com/buildwithsd/email-audio-agent.git
cd email-audio-agent
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

**2. Add your API key**

```bash
export GEMINI_API_KEY="your-key"   # free key from Google AI Studio
```

**3. Try it without Gmail**

```bash
python email_audio_agent.py --mock
```

**4. Connect Gmail**

1. In Google Cloud Console, create a project and enable the **Gmail API**.
2. Create an **OAuth client ID** (type: Desktop app), download it and save it as `credentials.json` in this folder.
3. Run `python email_audio_agent.py`. A browser opens for login the first time; after that `token.json` is reused.

`credentials.json`, `token.json`, `.env` and `digests/` are in `.gitignore`, so they're never committed.

## Run it every morning at 7 AM

Use cron (macOS/Linux). Run `crontab -e` and add:

```cron
0 7 * * * cd /path/to/email-audio-agent && GEMINI_API_KEY=your-key .venv/bin/python email_audio_agent.py >> digests/cron.log 2>&1
```

Your computer must be awake at 7 AM for cron to run.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | none | Needed for Gemini (default provider) |
| `LLM_PROVIDER` | `gemini` | `gemini` or `claude` |
| `ANTHROPIC_API_KEY` | none | Needed if `LLM_PROVIDER=claude` |
| `CLAUDE_MODEL` | `claude-sonnet-5-5` | Claude model to use |
| `GMAIL_QUERY` | `newer_than:1d in:inbox -category:promotions -category:social` | Which emails to read |
| `MAX_EMAILS` | `25` | Max emails per run |

## Privacy

Email content is sent to the LLM provider you choose. On free tiers, providers may use inputs to improve their products, so check their terms before using this with sensitive mail.

## Stack

Python · Gmail API · Gemini API · gTTS

## Roadmap

- [ ] Prompt-injection test set and PII redaction before the LLM call
- [ ] Rebuild as a LangGraph workflow with a human approval step for draft replies
- [ ] Deliver the MP3 straight to my phone
