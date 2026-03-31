# Voice-assistant
> A Jarvis-lite voice assistant that runs entirely on old hardware. No cloud. No API keys. Just your junk PC talking back.

---

## What it does

- 🎙️ Listens via mic → STT with Whisper (tiny/base)
- 🧠 Thinks via local LLM → Ollama + Mistral 7B (quantized)
- 🔊 Speaks back → Piper TTS, fully offline
- 💻 Optionally executes approved shell commands

Runs on **4GB RAM, CPU only.**
Note: This requires an external microphone and speakers. I will try to create a contraption with a speaker, microphone and a RPI5 that connects to the server. 
---

## Stack

| Layer | Tool | RAM Usage |
|-------|------|-----------|
| STT | `whisper.cpp` (tiny model) | ~200MB |
| LLM | Ollama + `mistral:7b-q4` | ~3.5GB |
| TTS | Piper TTS | ~100MB |
| UI | Terminal (SaShell-style) | negligible |

---

## Roadmap

```
Day 1 → Ollama running, chat works in terminal
Day 2 → Whisper STT piped into prompts
Day 3 → Piper TTS speaks responses
Day 4 → Wake word ("Hey Herald") + shell command execution
```

---

## Requirements

- Python 3.10+
- [Ollama](https://ollama.com) installed
- `whisper.cpp` compiled or `openai-whisper` pip package
- [Piper TTS](https://github.com/rhasspy/piper) binary + voice model
- Microphone

---

## Quick Start

```bash
# 1. Pull the model
ollama pull mistral:7b

# 2. Install deps
pip install openai-whisper pyaudio requests

# 3. Run
python herald.py
```

---

## Why self-host?

Because paying for API calls to say "turn off the lights" is embarrassing.  
Everything runs local. Your data stays yours. Works offline. Impresses people.

---

## Status

> 🚧 Work in progress — scaffolding phase

---

*Part of the self-hosted stack. Built on a Dell 4th gen i3 / 4GB RAM.*
