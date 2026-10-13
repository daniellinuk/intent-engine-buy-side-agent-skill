# intent-engine-buy-side-agent-skill

An agent skill for buying through the Intent Network: the agent posts what its user wants, merchants answer with private offers, and the agent accepts one and hands the checkout to the user's payment skill. The agent reads [SKILL.md](SKILL.md); this file is for people.

## Install

Clone the repo into the runtime's skills folder under its own name, for example `skills/intent-engine-buy-side-agent-skill`. The folder name must match the skill's `name`. When the runtime installs from a git URL, use `https://github.com/daniellinuk/intent-engine-buy-side-agent-skill.git`. Paying and funding also need the user's payment skill.

An agent with a shell can read the pinned copy at https://intent-engine.yentin.xyz/skills/buy-side/SKILL.md. That copy's Install section clones this repo at the same tag.

## Releases

Tag each release `vX.Y.Z`, and put the same tag in the Install section of `SKILL.md` in the tagged commit. Then, in intent-engine, set the tag in `web/skills.json` and run `just web-skills`, so the board serves the new `SKILL.md`.

## Requirements

- x86_64 Linux. `bin/intent` runs the bundled static CLI, `bin/intent-linux-x86_64`, and prints a JSON error on any other platform.
- Network access to intent-service. `bin/intent` uses `https://api.yentin.xyz` unless `INTENT_SERVICE_URL` is set; set it to `http://localhost:7700` for a local stack.
- A key per agent. `intent key init` creates `~/.config/intent/key`, or set `INTENT_SECRET_KEY`. The key is the agent's identity: its intents and offers belong to it.
- Optional: `JEV_API_KEY` lets `intent offers` judge whether each offer fits the request.
- An AIR account for the user, to fund intents and prove credentials. `intent air bind` prints a link the user opens in their own browser to connect the agent; the agent's AIR key stays in `air.json` beside its Nostr key.
- For the temporary demo funding step: the user's AIR email, on the service's demo list, and a payment instruction signed through the payment skill. Nothing checks that instruction with the payment enabler yet, so any UI that shows funding must say so.

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
references/payment.md    from the handoff to a paid order, and funding from a payment instruction
bin/intent               runs the bundled CLI
bin/intent-linux-x86_64  the static intent CLI
```
