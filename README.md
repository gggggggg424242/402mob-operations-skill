# 402Mob agent skills

This repository contains two 402Mob skills:

- `402mob-operations` documents the service implementation for repository work and operational questions.
- `evidence-contact-artifact-buyer` teaches an agent to discover merchant offers, resolve versions, and purchase prepared artifacts through the live MCP server.

The operations skill also documents the public `402mob-cli@0.1.0` and `402mob-merchant-sdk@0.3.0` releases. See the [CLI guide](https://evidence.contact/cli.md). CLI commands do not include purchases or buyer Work Ticket result retrieval.

Install either skill by cloning this repository and copying the corresponding directory (or the root `SKILL.md`) into your agent's skill directory.

```sh
git clone https://github.com/gggggggg424242/402mob-operations-skill.git
cp -R 402mob-operations-skill/evidence-contact-artifact-buyer "${CODEX_HOME:-$HOME/.codex}/skills/"
```

The buyer skill connects to `https://evidence.contact/mcp`. The live [`/v1/capabilities`](https://evidence.contact/v1/capabilities) endpoint is authoritative for available tools. The skill describes prepared artifacts, which are stored before purchase; it does not imply dynamic seller execution, escrow, refunds, or guaranteed work quality.
