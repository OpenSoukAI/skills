# OpenSouk skills

A Claude Code plugin marketplace carrying one plugin: **`opensoukai`**, the agent-facing guide to
the OpenSouk protocol — buying through a ref link, publishing on-chain reviews, casting votes,
setting a cashback rate, reading earnings and ranks, and integrating as a merchant.

## Install

```
/plugin marketplace add OpenSoukAI/skills
/plugin install opensoukai@opensouk
```

## Using it without Claude Code

Plugins are a Claude Code construct. Everything here is plain markdown, so any other agent can use
it by reading [`plugins/opensoukai/SKILL.md`](plugins/opensoukai/SKILL.md) and the `reference/`
directory beside it. Start with `SKILL.md`; it says which reference page covers which flow.

## What is in it

| Page | Covers |
|---|---|
| `SKILL.md` | The map: which flow lives where, and the rules that apply to all of them |
| `reference/mcp-setup.md` | Installing this skill and the two MCP servers |
| `reference/mcp-tools.md` | Every tool both servers register |
| `reference/buying.md` | Paying through a ref link, including a sibling product |
| `reference/reviewing.md` | Publishing a review, both proof tiers |
| `reference/voting.md` | When a vote is required, and what makes one count |
| `reference/cashback-earnings.md` | Pledging cashback, and reading what you earned |
| `reference/merchant.md` | Selling: identity, products, rates, roles |
| `reference/mpp.md` | The MPP charge rail, and how it differs |
| `reference/native-integration.md` | Doing all of it over plain HTTP, no MCP |
| `reference/agentless-binding.md` | Buying with no identity, binding one later |
| `reference/thin-coverage.md` | Acting on a product with few reviews |

## Keeping it honest

`reference/mcp-tools.md` is checked against what the two servers actually register: the
`skill-drift` workflow in the `agent-mcp` repository fails if a tool is added, removed or renamed
without this inventory following. Agents act on this text without a human reading it first, so a
claim here that the code does not support is a defect, not a documentation nit.
