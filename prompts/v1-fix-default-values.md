# Prompt: Remove default inputs, TPM formula, gated calculations

Conversation transcript (user message):

---

```text
Fix v1/index.html:

1. Remove all default values from input fields.
   All fields should start empty.
   Show placeholder text instead:
   - Monthly Input Tokens: placeholder "e.g. 100"
   - Monthly Output Tokens: placeholder "e.g. 20"
   - PTU Count: placeholder "e.g. 100"

2. When fields are empty, show "--" instead of 
   calculated values. Do not calculate or display 
   any costs until the user has entered values 
   in all required fields.

3. TPM formula:
   input_TPM = monthly_input_millions × 1,000,000 
               / (30 × 24 × 60)
   output_TPM = monthly_output_millions × 1,000,000 
                / (30 × 24 × 60)

4. Make sure ALL values update instantly via oninput 
   when ANY field changes.
```

**Follow-up refinement:** Subsequent iteration split “required for **all** costs” vs “show serverless + TPM as soon as **both** token fields are valid,” and added PAYG-only chart when PTU was still empty (see `v1-fix-serverless-cost.md`).
