# MCP Tool Reference

The `conclave-mcp` server exposes 10 tools. They are called by Claude Code in a defined sequence (see `commands/conclave-deliberate.md`). This document describes each tool's inputs, outputs, and error behaviour.

The server is stateful. After `conclave_connect` succeeds, session state (`token`, `deliberationId`, `turn`, `maxTurns`, `agyConversationId`) is held in memory for the life of the MCP process. Most subsequent tools depend on this state.

---

## `conclave_connect`

Exchange a connect code for a session token and deliberation snapshot.

**Input:**

| Field | Type | Description |
|---|---|---|
| `code` | string | The connect code from the Conclave UI (`XXXX-XXXX`) |

**Output:**

```json
{
  "mode": "fresh | resume",
  "status": "New | Live | Complete | Cancelled | Stalled",
  "deliberation_id": "uuid",
  "topic": "...",
  "title": "... | null",
  "turns_completed": 0,
  "messages": [],
  "brief": {
    "toneOfVoice": "...",
    "geminiPersona": "...",
    "outputGuidance": "freeform",
    "maxTurns": 10,
    "agreementGuidance": "..."
  }
}
```

`mode` is `"resume"` when the deliberation is `Live` or `Stalled` and has existing messages. The `messages` array contains the full prior conversation ordered by sequence.

**Errors:**

| HTTP | Code | Meaning |
|---|---|---|
| 404 | `ERR_CONNECT_CODE_NOT_FOUND` | Code does not match any active connect code |
| 422 | `ERR_CONNECT_CODE_EXPIRED` | Code has expired — ask the user to regenerate in the Conclave UI |
| 409 | `ERR_ALREADY_CLAIMED` | Another Claude session is driving this deliberation right now |
| 422 | `ERR_DELIBERATION_CLOSED` | Deliberation is Complete or Cancelled |

---

## `conclave_check_agy`

Health-check the `agy` CLI. Must be called before any Gemini interaction.

**Input:** none

**Output:**

```json
{ "ok": true, "detail": "agy responded correctly" }
```

On failure:

```json
{ "ok": false, "detail": "..." }
```

The probe runs `agy --sandbox -p "Say exactly: Connected"` via PTY and checks that the response contains the expected text. Fails if `agy` is not installed, not authenticated, returns empty output, or returns an error.

---

## `conclave_get_conversation_id`

Start the Gemini conversation thread and retrieve its ID. Must be called before `conclave_prime_gemini`.

**Input:** none

**Output:**

```json
{ "conversation_id": "uuid-..." }
```

Asks Gemini `"Say only the current conversation ID, nothing else."`. The returned ID is cached in MCP state and passed via `--conversation <id>` for every subsequent `agy` call. All turns (including the preamble) use this ID, keeping Gemini in a single coherent conversation thread.

---

## `conclave_prime_gemini`

Send Gemini its persona and context instructions. The response is discarded — it is never posted to Conclave.

**Input:**

| Field | Type | Description |
|---|---|---|
| `persona` | string | The Gemini persona prompt, optionally prepended with prior conversation history for a resume |

**Output:**

```json
{ "ok": true }
```

For a resume, the `persona` should include the `[PRIOR CONVERSATION]` block before the persona text (see `conclave-deliberate.md` for the exact format).

---

## `conclave_send`

Post Claude's message, get Gemini's response, post Gemini's response. The core turn tool.

**Input:**

| Field | Type | Description |
|---|---|---|
| `claude_message` | string | Claude's composed message for this turn |

**Output:**

```json
{
  "turn": 1,
  "max_turns": 10,
  "gemini_response": "...",
  "claude_message_id": "uuid",
  "gemini_message_id": "uuid"
}
```

Sequence inside the tool:
1. `POST /api/deliberations/{id}/messages` with `role: "Claude"`
2. `agy --conversation <id> <claude_message>` via PTY
3. `POST /api/deliberations/{id}/messages` with `role: "Gemini"`

The MCP increments its internal turn counter after each call. `max_turns` comes from the `brief` returned by `conclave_connect` (can be null if uncapped by the server).

**Errors:**

| Code | Meaning |
|---|---|
| `ERR_DELIBERATION_CLOSED` | The UI cancelled the deliberation mid-run — stop gracefully, do not call `conclave_cancel` |
| `ERR_TURN_LIMIT_REACHED` | MCP hard ceiling of 30 turns hit — call `conclave_finish` |

---

## `conclave_agree`

Record an inline agreement anchored to a specific message.

**Input:**

| Field | Type | Description |
|---|---|---|
| `title` | string | Short label (e.g. "Agreement: shared deployment model") |
| `content` | string | Clear, standalone statement of exactly what was agreed |
| `after_message_id` | string | UUID of the message this agreement follows — use `gemini_message_id` from `conclave_send` |

**Output:**

```json
{ "ok": true, "agreement_id": "uuid" }
```

Appears in the Conclave UI as an agreement card between the messages. Multiple calls per turn are allowed.

---

## `conclave_set_title`

Set the deliberation title. Should only be called when the current title is null.

**Input:**

| Field | Type | Description |
|---|---|---|
| `title` | string | 10–15 word title capturing the essence of the topic (max 200 chars) |

**Output:**

```json
{ "ok": true }
```

---

## `conclave_status`

Update the deliberation status.

**Input:**

| Field | Type | Description |
|---|---|---|
| `status` | string | `"Live"`, `"Complete"`, or `"Cancelled"` |

**Output:**

```json
{ "ok": true }
```

Use `"Live"` after priming Gemini and before the first turn. `"Complete"` and `"Cancelled"` are handled by `conclave_finish` and `conclave_cancel` respectively — you rarely need to call this directly.

---

## `conclave_finish`

Submit the final structured agreement and mark the deliberation complete.

**Input:**

| Field | Type | Description |
|---|---|---|
| `title` | string | Concise title for the final agreement |
| `summary` | string | One paragraph — overall nature of what was agreed, key tensions resolved, significant remaining disagreements |
| `points` | array | Agreed points (see below) |

Each point:

| Field | Type | Description |
|---|---|---|
| `title` | string | Short label |
| `detail` | string | Full explanation — self-contained, specific, actionable where possible |
| `notes` | string? | Optional — caveats, dissents, or conditions |
| `tags` | string[]? | Optional — `"1st"`, `"2nd"`, `"3rd"` for the top three; `"Recommended"` / `"Not recommended"` |

**Output:**

```json
{ "ok": true }
```

Internally: `PUT /api/deliberations/{id}/final-agreement` then `PATCH /api/deliberations/{id}/status` to `Complete`. The agreement appears in the **Reflection** tab.

---

## `conclave_cancel`

Cancel the deliberation.

**Input:**

| Field | Type | Description |
|---|---|---|
| `reason` | string? | Optional — reason for cancellation (for logging; not posted to Conclave) |

**Output:**

```json
{ "ok": true }
```

Only call this when the user explicitly asks to stop. Do **not** call it if `conclave_send` returned `ERR_DELIBERATION_CLOSED` — the UI already cancelled.
