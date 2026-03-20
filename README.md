# cc-pixel-office

Pixel-art office simulation plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Visualize any folder structure as a living, breathing pixel-art office. Characters walk, talk, deliver documents, and rest — all driven by real file changes.

## Install

```bash
# Add marketplace
/plugin marketplace add shuntailor/cc-pixel-office

# Install plugin
/plugin install pixel-office@cc-pixel-office
```

## Usage

```
/pixel-office
```

Select a folder to visualize (e.g. `.company/`), and the office starts automatically.

## Features

- Folders become rooms, files determine employee count
- Real-time file watching — save a file, characters react
- HP system — employees get tired, visit cafeteria or sleep room
- Qwen LLM integration — contextual chat bubbles and daily briefings
- Office shop — buy furniture with in-game currency
- Collaboration graph — visualize inter-department connections

## Requirements

- Node.js 18+
- Ollama + Qwen 2.5 (optional, for AI chat)

```bash
# Optional: Install Ollama + Qwen for AI features
brew install ollama
ollama serve &
ollama pull qwen2.5:1.5b
```

## Works with

- **[cc-company](https://github.com/shuntailor/cc-company)** — Create the virtual company structure that Pixel Office visualizes

## Cost

**$0/month.** Runs entirely locally. No API keys required.

## License

BSL-1.1 (Business Source License)

- Personal & educational use: Free
- Commercial use: Contact inabe.609@gmail.com

Copyright (c) 2026 Shuntaro Inabe
