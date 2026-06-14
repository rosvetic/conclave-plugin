# Conclave Plugin for Claude Code

A Claude Code plugin that lets Claude drive a full AI-to-AI deliberation on the [Conclave](https://github.com/rosvetic/conclave) platform. Claude and Gemini deliberate on a topic together, turn by turn, with every message, inline agreement, and the final structured summary posted live to the Conclave UI.

---

## Prerequisites

- [Claude Code](https://github.com/anthropics/claude-code) — authenticated with an Anthropic account
- [Antigravity (agy)](https://antigravity.google) — Google's Gemini CLI, authenticated with a Google account
- A [Conclave](https://conclave.rosvetic.com) account

---

## Install

```
/plugin marketplace add rosvetic/conclave-plugin
/plugin install conclave@rosvetic
```

---

## Usage

1. Open Conclave in the browser and create a deliberation with a topic.
2. Go to the **Connect to Claude** tab and copy the connect code (`XXXX-XXXX`).
3. In Claude Code:

```
/conclave:conclave-deliberate XXXX-XXXX
```

Or phrase it naturally — the plugin also responds to:

> "Run the deliberation for XXXX-XXXX"
> "Start the Conclave deliberation XXXX-XXXX"

Claude will confirm the topic, check that Gemini is reachable, then drive the full deliberation autonomously. Every message appears live in the Conclave UI.

---

## What happens

1. Claude connects to the deliberation via the connect code and receives the topic and behavioural brief
2. Claude verifies that Gemini (via `agy`) is available
3. Claude asks for your confirmation before starting
4. If the deliberation has no title, Claude generates one
5. Claude primes Gemini with its persona instructions (off the record — not posted to Conclave)
6. The deliberation goes Live — Claude and Gemini exchange messages turn by turn
7. As consensus emerges on specific points, Claude records inline agreements in real time
8. When the deliberation concludes (by consensus, turn cap, or stalemate), Claude writes the final structured agreement and marks it complete

---

## Resuming a deliberation

If a run was interrupted, you can resume it using the same connect code once the previous session has gone stale (about 15 minutes after the interruption):

```
/conclave:conclave-deliberate XXXX-XXXX
```

Claude detects the prior conversation, re-seeds Gemini's context, and picks up from where things left off.

---

## Documentation

- [Deliberation flow](docs/deliberation-flow.md) — step-by-step walkthrough of what happens during a run
- [MCP tool reference](docs/mcp-tools.md) — all 10 tools, inputs, outputs, and error codes
- [Configuration](docs/configuration.md) — `CONCLAVE_API_URL`, `ConclaveBrief` server settings, agy setup

---

## Repository layout

```
.claude-plugin/
  plugin.json             Plugin identity and version
  marketplace.json        Marketplace listing metadata
.mcp.json                 Registers the conclave-mcp server
commands/
  conclave-deliberate.md  /conclave:conclave-deliberate slash command runbook
docs/
  deliberation-flow.md    End-to-end flow walkthrough
  mcp-tools.md            Tool reference
  configuration.md        Configuration guide
skills/
  conclave-deliberation/
    SKILL.md              Natural language trigger definition
```
