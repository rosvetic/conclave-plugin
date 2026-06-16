---
description: Run a Conclave deliberation when the user provides a connect code
---

# Conclave Deliberation

When the user asks to run, start, continue, or resume a Conclave deliberation and provides a connect code (formatted as XXXX-XXXX), invoke the `/conclave:conclave-deliberate` command with that code.

## Trigger patterns

- "Run the deliberation for XXXX-XXXX"
- "Start the deliberation XXXX-XXXX"
- "Resume the deliberation for XXXX-XXXX"
- "Conclave XXXX-XXXX"
- "Use connect code XXXX-XXXX"
- Any message containing a code in the format XXXX-XXXX alongside intent to deliberate

## What to do

Extract the connect code from the user's message and run `/conclave:conclave-deliberate <CODE>`.

If no code is provided, ask the user for it:

> "Please share the connect code from the Conclave UI (it looks like XXXX-XXXX, found on the Connect to Claude tab)."
