# Clara · Rural credit pre-screening agent

"Crédito rural por voz" for Banco Agrario: an ElevenLabs Conversational AI agent that pre-screens rural credit applicants over WhatsApp (voice notes, text, ID photos) or the web widget, files the case in Notion and books an appointment with a human advisor.

Built as a demo for a Solutions Engineer take-home. Not a production system.

## Repository layout

| Path | What it is |
|---|---|
| `CLARA_SPEC.md` | Full technical specification: workflow, tools, prompts, data model, guardrails, tests |
| `agent_configs/My-Agent.json` | The agent configuration as exported by the ElevenLabs agents CLI |
| `agents.json`, `tools.json`, `tests.json` | ElevenLabs CLI manifests |
| `.env.example` | Environment variables template |

## Running it

The agent is managed as configuration-as-code with the [ElevenLabs agents CLI](https://github.com/elevenlabs/cli):

```bash
npm install -g @elevenlabs/cli
cp .env.example .env   # add your ELEVENLABS_API_KEY
elevenlabs agents push  # sync agent_configs/My-Agent.json to your workspace
```

See section 12 of `CLARA_SPEC.md` for day-to-day operation of the configuration.

No secrets are stored in this repository. Tool credentials are ElevenLabs workspace secrets referenced by id.
