# intent-engine-buy-side-agent-skill

An agent skill for buying through the Intent Network: the agent posts what its user wants, merchants answer with private offers, and the agent accepts one and hands the checkout to a payment skill such as Clink. The agent reads [SKILL.md](SKILL.md); this file is for people.

## Install

- **HelloMinds:** add `https://github.com/daniellinuk/intent-engine-buy-side-agent-skill.git` to `SKILLS_REPOS`, or install it with the `skills` tool, and add `intent-engine-buy-side-agent-skill` to the mind's skill list. The payment skill (`agentic-payment-skills`) must be installed too.
- **Other runtimes:** clone the repo into the runtime's skills folder under its own name, for example `~/.claude/skills/intent-engine-buy-side-agent-skill`. The folder name must match the skill's `name`.

## Requirements

- x86_64 Linux. `bin/intent` runs the bundled static CLI, `bin/intent-linux-x86_64`, and prints a JSON error on any other platform.
- Network access to intent-service. `bin/intent` uses `https://api.yentin.xyz` unless `INTENT_SERVICE_URL` is set; set it to `http://localhost:7700` for a local stack.
- A key per agent. `intent key init` creates `~/.config/intent/key`, or set `INTENT_SECRET_KEY`. The key is the agent's identity: its intents and offers belong to it.
- Optional: `JEV_API_KEY` lets `intent offers` judge whether each offer fits the request.
- For the temporary demo funding step: the user's AIR email, on the service's demo list, and a Clink instruction signed through the payment skill. Nothing checks that instruction with Clink yet, so any UI that shows funding must say so.

## Update the CLI

The CLI is bundled until it is published. From an intent-engine checkout:

```bash
just bundle <path to this repo>
```

`bin/intent --version` shows the intent-engine commit it was built from.

## Layout

```
SKILL.md                 what the agent reads
references/commands.md   every buyer command, its flags, output and errors
references/payment.md    from the handoff to a paid order, and funding from a Clink instruction
bin/intent               runs the bundled CLI
bin/intent-linux-x86_64  the static intent CLI
```
