# agent-skills

A collection of AI agent skills for automating personal workflows. Uses the common [Agent Skills](https://agentskills.io/home) format supported by different agentic AI tools: Claude Code, Cursor, Gemini CLI, Codex, OpenClaw and so on.

## Setup

1. Clone this repo into a folder your agentic AI tool uses for skills.
2. Run `cp .env.example .env` and fill in your values in the `.env` file.

## Skills

### estonia-events-add

Add tech events to the [Tallinn.dev](https://www.tallinn.dev) community events calendar from any URL.

| Variable | Description |
| --- | --- |
| `ESTONIA_EVENTS_CALENDAR_ID` | Target Google Calendar ID |
| `GOPLACES_API_KEY` | Google Places API key for venue resolution |

### estonia-events-coda

Label and maintain the [Tallinn.dev](https://www.tallinn.dev) events database in Coda.

| Variable | Description |
| --- | --- |
| `CODA_API_TOKEN` | Coda API authentication token |
| `CODA_DOC_ID` | Coda document ID |
| `CODA_TABLE_ID` | Coda table ID |

### lhv-investment-report

Automate the annual LHV investment account tax report (Investeerimiskonto aruanne) and submit to MTA.

| Variable | Description |
| --- | --- |
| `LHV_USERNAME` | LHV internet bank username |
| `LHV_ISIKUKOOD` | Estonian personal ID number (isikukood) |
| `LHV_ACCOUNT_INVESTMENT` | LHV investment account IBAN |
| `LHV_ACCOUNT_EXTERNAL` | External account IBAN that is used to make transfers from/to investment account |

### linkedin-connect

Send LinkedIn connection requests (without a note) to people in search results, page by page, until the weekly limit is reached.

| Variable | Description |
| --- | --- |
| `LINKEDIN_CONNECT_URL` | LinkedIn people search URL to iterate over |

## Security disclaimer

All secrets should be in env variables. Please check skills content and run them at your own risk.
