---
description: Run a structured deliberation between Claude and Gemini via Conclave
---

# /conclave-deliberate

Drive a full AI-to-AI deliberation on the Conclave platform. You compose every Claude message, call Gemini via the MCP, and post both sides of the conversation live to the Conclave UI. At the end you produce a structured final agreement and mark the deliberation complete.

## Usage

```
/conclave-deliberate XXXX-XXXX
```

The code comes from the **Connect to Claude** tab in the Conclave UI. It is valid for 30 minutes.

---

## Step-by-step process

Follow these steps in order every time. Do not skip or reorder them.

### 1. Connect

Call `conclave_connect` with the code from the arguments.

- If the response `status` is `Complete` or `Cancelled`: tell the user this deliberation has already ended and stop. They will need to create a new one.
- If `mode` is `resume`: acknowledge it to the user — "Resuming deliberation from turn {turns_completed}." Keep the full `messages` array from the response; you will need it to re-seed Gemini's context in step 7.
- If `mode` is `fresh`: proceed normally.

Store `brief` from the response. You will use `brief.toneOfVoice`, `brief.geminiPersona`, `brief.maxTurns`, and `brief.agreementGuidance` throughout.

### 2. Check agy

Call `conclave_check_agy`. If it fails, tell the user:

> "Antigravity (agy) is not responding. Please check that agy is installed and you are authenticated with your Google account, then try again."

Stop. Do not proceed.

### 3. Get the conversation ID

Call `conclave_get_conversation_id`. This starts the Gemini conversation thread that will be used for the entire deliberation.

### 4. Ask for user consent

Show the user:

> **Topic:** {topic}
> **Title:** {title or "(no title yet)"}
>
> Ready to start the deliberation with Gemini?

Wait for confirmation. If the user declines, stop. They can re-invoke the command later with the same code.

### 5. Set the title (if null)

If `title` is null or empty, generate a concise title of 10–15 words that captures the essence of the topic. Call `conclave_set_title` with it.

### 6. Prime Gemini

Build the Gemini persona prompt. Use `brief.geminiPersona` as the base (the topic is already interpolated into it). If this is a **resume**, prepend the prior conversation history using this exact structure:

```
[PRIOR CONVERSATION — for context only]
Turn 1 — Claude: {message}
Turn 1 — Gemini: {message}
Turn 2 — Claude: {message}
Turn 2 — Gemini: {message}
... (all prior turns)
[END OF HISTORY]

{brief.geminiPersona}

You are resuming this conversation from turn {turns_completed}. Write your next response directly without acknowledging, summarising, or re-answering any prior messages. Begin now:
```

Call `conclave_prime_gemini` with this prompt. Gemini's response to this is discarded — it is never posted to Conclave.

### 7. Go Live

Call `conclave_status` with `"Live"`. The deliberation is now officially in progress.

### 8. Turn loop

Repeat the following until you decide to stop (see "Stopping" below).

**a. Compose your message.**

Write your contribution for this turn. Follow `brief.toneOfVoice` throughout. Good deliberation messages:
- Engage directly with what Gemini said in the previous turn (after turn 1)
- Make one or two clear, substantive points — do not try to cover everything
- Challenge where you genuinely disagree; acknowledge where you genuinely agree
- Raise new angles when the current thread is exhausted
- Avoid sycophancy ("Great point!") and hollow agreement

Your first message (turn 1) should open the deliberation by framing your initial position on the topic.

**b. Call `conclave_send`.**

Pass your composed message as `claude_message`. The tool posts it to Conclave, sends it to Gemini, posts Gemini's response, and returns:

```
{ turn, max_turns, gemini_response, claude_message_id, gemini_message_id }
```

Read `gemini_response` carefully. This is Gemini's actual reply.

**c. Record inline agreements.**

Immediately after each `conclave_send`, assess whether this turn produced genuine unanimous agreement on one or more specific points. Use `brief.agreementGuidance` as your standard.

Genuine agreement means both sides have explicitly and unambiguously converged on a specific claim — not polite acknowledgement ("I see your point"), not partial overlap, not agreement in principle with caveats that negate it.

For each genuinely agreed point, call `conclave_agree`:
- `after_message_id`: use `gemini_message_id` from the `conclave_send` response
- `title`: a short label for the point (e.g. "Agreement: shared deployment model")
- `content`: a clear, standalone statement of exactly what was agreed

You may call `conclave_agree` multiple times in the same turn if multiple distinct points were agreed.

**d. Decide whether to continue.**

Stop the loop and move to step 9 if any of the following are true:

- You judge that genuine consensus has been reached and the deliberation has nothing productive left to explore
- `turn` equals `max_turns` (if set) — the cap has been reached
- The last 2–3 turns have covered the same ground with no new points being raised (stalemate)
- The user asks you to stop mid-deliberation (call `conclave_cancel` and stop)
- `conclave_send` throws an error containing `ERR_DELIBERATION_CLOSED` — the UI has cancelled the deliberation; stop gracefully without calling `conclave_cancel`

**Approaching the cap:** when `turn` is 2 turns before `max_turns`, begin steering the conversation toward conclusions. When `turn` is 1 turn before `max_turns`, make your final points clearly and explicitly seek agreement on the outstanding items.

### 9. Build and submit the final agreement

Synthesise everything agreed across the whole deliberation into a structured final agreement. Call `conclave_finish` with:

- `title`: a concise title for the final agreement
- `summary`: one paragraph capturing the overall nature of what was agreed, the key tensions that were resolved, and any significant remaining disagreements
- `points`: an array of the agreed points, each with:
  - `title`: short label
  - `detail`: full explanation of the point — self-contained, specific, actionable where possible
  - `notes`: optional — caveats, dissents, or conditions attached to the point
  - `tags`: optional — use `"1st"`, `"2nd"`, `"3rd"` for the top three most significant points; `"Recommended"` for the recommended course of action; `"Not recommended"` for explicitly rejected alternatives

Draw your points from the inline agreements already recorded plus any additional points both sides clearly converged on but that were not captured as inline agreements.

### 10. Tell the user

After `conclave_finish` succeeds:

> "The deliberation is complete. {n} agreement points were recorded. You can view the full conversation and final agreement in the Conclave UI."

---

## Cancellation

If the user asks to stop at any point after step 7 (Go Live), call `conclave_cancel` before stopping. This marks the deliberation as Cancelled in the UI so it does not sit in a Live state indefinitely.

If the process is interrupted without a chance to call `conclave_cancel`, the server-side inactivity lease will mark the deliberation as Stalled after 15 minutes. It can be resumed by running this command again with the same connect code once the session has gone stale.

---

## Resume

When `conclave_connect` returns `mode: "resume"`, the deliberation was started but not completed. The full message history is in the response. Follow the normal steps but:

- Skip the user consent question — just tell the user you are resuming and from which turn
- In step 6, include the full prior conversation history in the Gemini persona prompt using the `[PRIOR CONVERSATION]` structure above
- In step 8, your first message continues the conversation rather than opening it

---

## Error handling

| Error | Action |
|---|---|
| `conclave_connect` → 409 ERR_ALREADY_CLAIMED | Tell the user another Claude session is already driving this deliberation. Stop. |
| `conclave_connect` → 404 | Tell the user the connect code was not found. Stop. |
| `conclave_connect` → 422 | Tell the user the connect code has expired. Ask them to generate a new one from the Conclave UI. Stop. |
| `conclave_check_agy` fails | Tell the user agy is not working. Stop. |
| `conclave_send` → ERR_DELIBERATION_CLOSED | The UI cancelled mid-run. Stop gracefully without calling `conclave_cancel`. |
| Any other API error | Tell the user what happened and stop. Do not retry automatically. |
