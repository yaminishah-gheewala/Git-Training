# CodeBasics AI Support Agent — n8n (Week 3)

A multi-step n8n agent that handles front-line learner support for CodeBasics:
it **triggers** on a learner query, **reasons** with an LLM to classify intent,
**acts** by retrieving answers from your FAQ knowledge base and logging every
response, and **outputs** either a learner-facing reply or an **escalation to a
human** for genuine edge cases (billing disputes, "payment deducted but failed",
real bugs).

Import file: [`Yamini_Week3_AI_Support_Agent.json`](./Yamini_Week3_AI_Support_Agent.json)

## What the workflow does

```
Chat trigger → Prepare Input → Download FAQ (Drive .xlsx) → Parse spreadsheet
   → Build FAQ context → Classify & Draft Answer (LLM + structured output)
       → Needs Human?
            ├─ YES → Escalate to Human → Notify Support Team (email)
            └─ NO  → Auto-Answer
       → Log to Yamini_week3_Responses (Google Sheet) → Format Reply
```

| Step | Node | Purpose |
|------|------|---------|
| Trigger | **When chat message received** | Learner types a question (one-click test chat). |
| — | **Prepare Input** | Captures query + metadata (name, email, session, timestamp). |
| Act #1 (retrieve) | **Download FAQ from Drive** → **Parse FAQ Spreadsheet** → **Build FAQ Context** | Reads `Yamini_Week3_FAQ_Knowledge_Base.xlsx` and turns it into LLM context. |
| Reason | **Classify & Draft Answer** (+ **OpenAI Chat Model**, **FAQ Output Parser**) | LLM classifies the query, decides if a human is needed, and drafts the answer. Returns structured JSON (`category`, `intent`, `needs_human`, `confidence`, `answer`). |
| Route | **Needs Human?** | Branches on `needs_human`. |
| Escalate | **Escalate to Human** → **Notify Support Team** | Holding message to learner + email to the support queue. |
| Answer | **Auto-Answer** | Sends the FAQ-grounded answer back. |
| Act #2 (log) | **Log to Yamini_week3_Responses** | Appends a row to your Google Sheet for every query. |
| Output | **Format Reply** | Returns the response text to the learner. |

This satisfies the rubric: a trigger, an LLM reasoning step, **two+ chained
actions** (FAQ retrieval + sheet logging, plus the escalation email), a
learner-facing output, and a **human-escalation branch**.

## Setup (after importing into n8n)

1. **Import** `Yamini_Week3_AI_Support_Agent.json` (Workflows → ⋯ → Import from File).
2. **Connect credentials** on these nodes:
   - **OpenAI Chat Model** → an OpenAI credential (model preset to `gpt-4o-mini`; swap if you like).
   - **Download FAQ from Drive** → a Google Drive credential, then click the
     `File` field and **select `Yamini_Week3_FAQ_Knowledge_Base.xlsx`** from the list.
   - **Log to Yamini_week3_Responses** → a Google Sheets credential, then paste
     your `Yamini_week3_Responses` sheet **URL/ID** into `Document` and pick the
     tab in `Sheet`.
   - **Notify Support Team** (optional) → a Gmail credential, and set `sendTo`
     to your real support inbox. *Delete this node if you don't want the email —
     the sheet row with `Status = Escalated` is already your human queue.*
3. **Add the header row** to `Yamini_week3_Responses` (row 1):

   ```
   Timestamp | Session ID | Learner Name | Learner Email | Query | Category | Intent | Needs Human | Confidence | Status | Response
   ```

4. (Recommended) Confirm your FAQ spreadsheet has columns like **Question** and
   **Answer** (optionally **Category**). The `Build FAQ Context` node also accepts
   `Query/Response/Topic/Type` headers; if your columns differ it falls back to
   the first two columns.

## Test queries to run (for your submission)

Open the **Chat** panel and try these — they exercise both branches:

1. **Course access (auto-answered):**
   *"I can't log in to my account and I can't access the course I bought."*
   → `category: course_access`, `needs_human: false`, FAQ-based reset steps.
2. **Refund in window (auto-answered):**
   *"I bought the Power BI course two days ago and want a refund."*
   → `category: refund`, `needs_human: false`, refund-policy answer.
3. **Payment dispute (escalated):**
   *"My payment failed but the money was deducted from my bank account."*
   → `category: payment_issue`, `needs_human: true`, holding message +
   escalation email + sheet row marked `Escalated to Human`.

Each run appends a row to `Yamini_week3_Responses`, so the sheet itself is your
evidence of inputs and outputs.

## Reflection (template — replace with your own ~half page)

- **What broke:** _e.g. the LLM occasionally returned prose instead of JSON; the
  FAQ download failed until the right file was selected; the IF branch fired on
  the wrong output field._
- **How you fixed it:** _added a Structured Output Parser so the model must
  return `category/intent/needs_human/confidence/answer`; selected the Drive file
  explicitly; branched on `output.needs_human`._
- **What you'd improve next:** _vector-store retrieval (embeddings) instead of
  dumping the whole FAQ into the prompt; a real account/order lookup tool; a
  confidence threshold that auto-escalates low-confidence answers; multi-turn
  memory; ticket creation in a help desk instead of an email._

## Notes

- Account/order data is intentionally mocked — the focus is the agent's logic.
- The chat trigger is used so you can demo and screenshot easily. To go
  "production", swap it for a Webhook + Respond-to-Webhook and feed `query`,
  `learnerName`, `learnerEmail` from your real intake form.
