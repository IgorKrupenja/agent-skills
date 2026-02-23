# agent-skills

A collection of AI agent skills for automating personal workflows. Uses the common [Agent Skills](https://agentskills.io/home) format supported by different agentic AI tools: Claude Code, Cursor, Gemini CLI, Codex, OpenClaw and so on.

## Setup

1. Clone this repo into a folder your agentic AI tool uses for skills.
2. Copy `.env.example` to `.env` and fill in your values.

## Skills

- **estonia-events-add** — Add tech events to the [Tallinn.dev](https://www.tallinn.dev) community events calendar from any URL.
- **estonia-events-coda** — Label and maintain the [Tallinn.dev](https://www.tallinn.dev) events database in Coda.
- **lhv-investment-report** — Automate the annual LHV investment account tax report (Investeerimiskonto aruanne) and submit to MTA.
- **linkedin-connect** — Send LinkedIn connection requests (without a note) to people in search results, page by page, until the weekly limit is reached.

## Security disclaimer

All secrets should be in env variables. Please check skills content and run them at your own risk.
