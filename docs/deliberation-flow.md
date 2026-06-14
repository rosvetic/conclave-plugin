# Deliberation Flow

A complete walkthrough of what happens from the moment you run `/conclave:conclave-deliberate XXXX-XXXX` to the final agreement appearing in the browser.

---

## Prerequisites

- Claude Code installed and authenticated
- `agy` (Antigravity) installed and authenticated with a Google account
- The Conclave plugin installed (`/plugin marketplace add rosvetic/conclave-plugin`)
- A deliberation created in the Conclave UI with an active connect code

---

## Step 1 — Connect

Claude calls `conclave_connect` with the code from your command. The MCP exchanges it for a bearer token scoped to that deliberation, and returns:

- `status` — the deliberation's current status (`New`, `Live`, etc.)
- `mode` — `"fresh"` (new run) or `"resume"` (picking up an interrupted one)
- `topic`, `title`, `messages` — snapshot of the deliberation
- `brief` — server-configured behaviour: tone of voice, Gemini persona, max turns, agreement guidance

**If the status is `Complete` or `Cancelled`**, Claude stops immediately — the deliberation is finished.

**If mode is `resume`**, Claude acknowledges it and skips the consent step. The prior messages are used to re-seed Gemini's context in step 6.

---

## Step 2 — Check agy

Claude calls `conclave_check_agy`. The MCP runs a short health probe via the `agy` PTY driver:

```
agy --sandbox -p "Say exactly: Connected"
```

If `agy` doesn't respond — not installed, not authenticated, or returning empty output — Claude stops and tells you what to fix.

---

## Step 3 — Get the conversation ID

Claude calls `conclave_get_conversation_id`. The MCP asks Gemini directly:

```
Say only the current conversation ID, nothing else.
```

Gemini returns its own agy conversation ID (it has accurate access to this). The ID is cached in MCP state and used via `agy --conversation <id>` for every subsequent call — persona preamble, every turn, everything. This keeps Gemini in the same conversation thread throughout.

---

## Step 4 — Consent

Claude shows you the topic and title (or "(no title yet)") and asks for confirmation before starting. If you decline, nothing has been posted to Conclave and you can run the command again later with the same code.

---

## Step 5 — Set the title (if missing)

If the deliberation has no title, Claude generates a concise one (10–15 words that capture the topic's essence) and patches it via `conclave_set_title`.

---

## Step 6 — Prime Gemini

Claude sends Gemini its persona instructions. The MCP calls `agy --conversation <id>` with the persona prompt from `brief.geminiPersona` (already interpolated with the topic by the server).

For a **resume**, the full prior conversation history is prepended in a structured envelope so Gemini re-enters the conversation with full context:

```
[PRIOR CONVERSATION — for context only]
Turn 1 — Claude: ...
Turn 1 — Gemini: ...
...
[END OF HISTORY]

{persona}

You are resuming from turn N. Write your next response directly.
```

Gemini's response to this preamble is discarded — it is never posted to Conclave. It is off the record.

---

## Step 7 — Go Live

Claude calls `conclave_status("Live")`. The deliberation is now officially in progress. The Conclave UI updates immediately.

---

## Step 8 — Turn loop

Repeats until Claude decides to stop.

### Each turn:

**a. Compose.** Claude writes its contribution, following `brief.toneOfVoice`. After turn 1, it engages directly with Gemini's previous reply — making one or two substantive points, challenging or conceding where warranted, raising new angles when the current thread is exhausted. No sycophancy.

**b. Send.** Claude calls `conclave_send({ claude_message })`. The MCP:
1. Posts Claude's message to Conclave → gets back `claude_message_id`
2. Calls `agy --conversation <id>` with Claude's message
3. Posts Gemini's response to Conclave → gets back `gemini_message_id`
4. Returns `{ turn, max_turns, gemini_response, claude_message_id, gemini_message_id }`

Both messages appear in the browser immediately.

**c. Inline agreements.** Claude assesses whether the turn produced genuine unanimous agreement on a specific point — not polite acknowledgement, not agreement in principle with caveats, but explicit and unambiguous convergence on a claim. For each such point, Claude calls `conclave_agree`:
- `after_message_id` — the `gemini_message_id` from the turn
- `title` — short label (e.g. "Agreement: shared deployment model")
- `content` — a clear, standalone statement of exactly what was agreed

Multiple `conclave_agree` calls are fine in a single turn.

**d. Decide.** Claude stops the loop if:
- Genuine consensus has been reached — the deliberation has nothing productive left to explore
- `turn` equals `max_turns` — the cap has been hit
- The last 2–3 turns have covered the same ground with no new points (stalemate)
- The user asks to stop (Claude calls `conclave_cancel` then stops)
- `conclave_send` returns `ERR_DELIBERATION_CLOSED` — the UI cancelled mid-run; stop gracefully, no cancel call needed

**Approaching the cap:** two turns before `max_turns`, Claude steers toward conclusions. One turn before, Claude makes final points and explicitly seeks agreement on outstanding items.

---

## Step 9 — Final agreement

Claude synthesises all agreed points into a structured final agreement and calls `conclave_finish`:

```json
{
  "title": "...",
  "summary": "one paragraph — overall nature, key tensions resolved, any significant remaining disagreements",
  "points": [
    {
      "title": "short label",
      "detail": "full explanation — self-contained, specific, actionable where possible",
      "notes": "optional — caveats, dissents, conditions",
      "tags": ["1st", "Recommended"]
    }
  ]
}
```

The MCP posts the final agreement and patches status to `Complete`. The agreement appears in the **Reflection** tab in the browser.

---

## Step 10 — Done

Claude tells you it's finished and how many agreement points were recorded. Open the Conclave UI to read the full conversation and final agreement.

---

## Resume path

If a previous run was interrupted (hard-kill, network drop, etc.) and the session went stale (no activity for 15+ minutes), the deliberation enters the `Stalled` state. Running `/conclave:conclave-deliberate XXXX-XXXX` again with the same code:

1. The old session is cleared automatically
2. `conclave_connect` returns `mode: "resume"` with the full message history
3. Claude skips consent, prepends the history to Gemini's preamble, and picks up from the next turn

The connect code remains valid; you don't need to generate a new one.

---

## Cancellation

If you ask Claude to stop after step 7, it calls `conclave_cancel` before stopping. This marks the deliberation `Cancelled` in the UI so it doesn't sit in `Live` indefinitely.

If the process is hard-killed, the server-side inactivity lease marks the deliberation `Stalled` after 15 minutes. Resume via the same connect code once the session is stale.
