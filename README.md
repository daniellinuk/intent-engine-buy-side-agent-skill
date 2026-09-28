# buy-side-agent-skill

An agent skill for buying through the Intent Network: the agent posts what its user wants, merchants answer with private offers, and the agent accepts one and hands the checkout to a payment skill such as Clink. The agent reads [SKILL.md](SKILL.md); this file is for people.

## Install

- **HelloMinds:** add this repo's git URL to `SKILLS_REPOS`, or install it with the `skills` tool, and add `buy-side-agent-skill` to the mind's skill list. The payment skill (`agentic-payment-skills`) must be installed too.
- **Other runtimes:** clone the repo into the runtime's skills folder, for example `~/.claude/skills/buy-side-agent-skill`.

## Requirements

- x86_64 Linux. `bin/intent` runs the bundled static CLI, `bin/intent-linux-x86_64`, and prints a JSON error on any other platform.
- Network access to intent-service. Set `INTENT_SERVICE_URL`; without it the CLI uses `http://localhost:7700`.
- A key per agent. `intent key init` creates `~/.config/intent/key`, or set `INTENT_SECRET_KEY`. The key is the agent's identity: its intents and offers belong to it.
- Optional: `JEV_API_KEY` lets `intent offers` judge whether each offer fits the request.

## Update the CLI

The CLI is bundled until it is published. From an intent-engine checkout next to this repo:

```bash
just bundle ../buy-side-agent-skill
```

`bin/intent --version` shows the intent-engine commit it was built from.

## Layout

```
SKILL.md                 what the agent reads
references/commands.md   every buyer command, its flags, output and errors
references/payment.md    from the handoff to a paid order
bin/intent               runs the bundled CLI
bin/intent-linux-x86_64  the static intent CLI
```
