# Prompt: Serverless cost stuck at “--” and live updates

Conversation transcript (user message):

---

```text
Fix v1/index.html:
The "Current monthly Serverless cost" is showing "--" 
even when the user has entered numbers in the input fields.
Make sure the cost calculation triggers on every 
input change and displays the result immediately.
Also fix all other calculations (TPM, cost cards, 
chart) to update on every input change.
```

**Root cause (for maintainers):** For PTU-supported models, `render()` returned early when PTU count was empty and forced the serverless line to `--` even when monthly input/output were valid.

**Resolution direction:** After both token fields parse, show **Current monthly Serverless cost** and **TPM** immediately; if PTU is still missing, show **PAYG horizon total** on the first card and a **PAYG-only** cumulative chart; full PTU comparison + break-even markers once PTU is entered. Harden parsing with `valueAsNumber` for `type="number"` and bind `input` / `change` / `keyup` on numeric fields.
