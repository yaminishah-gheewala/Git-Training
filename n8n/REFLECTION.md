# Reflection — Building the CodeBasics AI Support Agent (n8n)

## What I built
A multi-step n8n agent that triages learner-support queries end to end: a trigger
captures the query, an LLM step classifies intent and drafts an answer grounded in
our FAQ knowledge base (`Yamini_Week3_FAQ_Knowledge_Base.xlsx` on Google Drive),
a decision node routes genuine edge cases to a human, and every interaction is
logged to a Google Sheet (`Yamini_week3_Responses`). The agent auto-answers the
common queries (login/course access, certificates, refund policy, discounts,
course guidance) and escalates the handful that truly need a person (payment
disputes, double charges, real bugs, partnership enquiries).

## What worked
- **The reasoning + routing core was solid from the first run.** Once wired up, the
  LLM classified queries correctly (e.g. "I can't log in" → `course_access`,
  `needs_human: false`) and the IF branch sent payment disputes down the escalation
  path. Using a **structured output parser** (forcing `category / intent /
  needs_human / confidence / answer`) made the branching reliable instead of having
  to parse free-text.
- **FAQ grounding worked** — the agent's answers reflected the knowledge base rather
  than inventing policy, which is exactly the behaviour we want for support.
- **Logging every query to a sheet** gave an instant audit trail and doubled as the
  human queue for escalated items.

## What didn't work (and how we fixed it)
1. **LLM quota walls.** The first run failed at the LLM step with an OpenAI
   "insufficient quota" error — the account had no credits. We switched the model
   node to **Google Gemini (free tier)**. Gemini then returned a `limit: 0` quota
   error, which we traced to the API key landing in a Google Cloud project with no
   free-tier grant. *Fix:* created a fresh key in a new project and used a current
   model (`gemini-2.5-flash`, since `1.5-flash` is being retired). **Lesson:**
   "quota" errors are account/billing config, not workflow bugs — diagnose the
   provider, don't keep re-running.
2. **Chat returned `{ "output": null }` even though the run succeeded.** The
   classification and answer were correct in the node output, but the final reply
   was blank. *Root cause:* the final node read a field that the Google Sheets/Gmail
   nodes had dropped (those nodes replace the item data with their own output).
   *Fix:* rewired the final reply and the sheet's Status/Response columns to read
   from nodes that **always execute** (the Classify and Prepare Input nodes) instead
   of relying on field pass-through. **Lesson:** in n8n, don't assume fields survive
   downstream — reference the source node explicitly.
3. **Learner name/email logged as placeholders.** The chat trigger only captures the
   message text, so those columns logged default values. *Fix identified:* swap the
   chat trigger for a Form trigger that collects name, email, and question, then map
   them in Prepare Input.

## What I'd improve next
- **Retrieval over prompt-stuffing:** load the FAQ into a vector store and retrieve
  only the relevant entries, so the agent scales past a small spreadsheet.
- **A real account/order lookup tool** (mocked sheet today) so the agent can act on
  account-specific questions, not just answer from docs.
- **Confidence-based auto-escalation:** if the model's confidence is low, escalate
  rather than risk a wrong answer.
- **Closed-loop delivery:** email-in → email-out, and create a help-desk ticket on
  escalation instead of a notification email.
- **Guardrails + evaluation:** a small test set of queries run on every change to
  catch misclassification and regressions early.

## Takeaway
The agent logic — classify, ground, route, log, escalate — came together quickly
and was robust. The real work was in the integration plumbing: provider quotas and
n8n's data pass-through between nodes. Both were debuggable by reading the actual
error and inspecting node-level input/output rather than assuming the logic was
broken.
