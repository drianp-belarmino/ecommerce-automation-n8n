# AI Messenger Bot

An AI customer-service bot for a Shopify store's Facebook page. It answers from
a maintained knowledge base, replies in English, Tagalog or Taglish depending on
how the customer writes, flags buying intent, and logs every conversation.

Three workflows:

| File | Role |
|---|---|
| `messenger-bot.workflow.json` | The bot itself, 23 nodes |
| `knowledge-base-ingestion.workflow.json` | Loads the product/FAQ doc into the vector store |
| `hot-lead-followup.workflow.json` | Follows up with customers flagged as ready to buy |

## Flow

```
Facebook webhook
  ├── GET  ──> verification handshake ──> respond with challenge
  └── POST ──> verify page id ──> respond {"status":"ok"} immediately
                  │
             is it text?  ── no ──> "unsupported format" reply
                  │ yes
             drop duplicates (message id, 5 min window)
                  │
             rate limit (10 messages per user per 5 min)
                  │
             load knowledge base
                  │
             generate reply (Gemini + 10-message memory)
                  ├── error ──> fallback message
                  │              └── error ──> log to sheet ──> email owner
                  │
             detect buying intent (EN + TL keywords)
                  ├── yes ──> log hot lead
                  └── no
                  │
             send reply (3 retries, 1s apart)
                  └── error ──> log to sheet ──> email owner
```

## Decisions worth explaining

**Acknowledge Facebook before doing any work.**
Facebook retries any webhook it does not hear back from within 20 seconds, and
a retry means the customer gets answered twice. The workflow returns
`{"status":"ok"}` immediately, then continues processing. Everything expensive
happens after the acknowledgement.

**Deduplicate on message id anyway.**
Fast acknowledgement reduces duplicates, it does not eliminate them. Message ids
are held in workflow static data for five minutes and repeats are dropped.

**Rate limit per user, upstream of the model.**
Ten messages per user per five minutes. One person hammering the page should
not be able to run up an API bill or starve other customers. The limit sits
before the LLM call, not after, so blocked messages cost nothing.

**Ground the model in a Google Doc, not the prompt.**
Prices and stock change. Keeping them in a doc the owner edits means the
business can update the bot without anyone touching n8n. The model is instructed
to answer only from that content and never invent a price or a stock status.

**Answer in the customer's language.**
Customers write in English, Tagalog, and Taglish, often mid-sentence. The bot
matches whatever they used rather than normalising to English.

**Three layers of failure alerting.**
If the model fails, the customer still gets a fallback message. If the fallback
fails, it is logged to a sheet. If that fails too, the owner gets an email with
the customer id, their message, and the exact error. The design assumption is
that a silent failure is worse than a noisy one, because the owner finds out
from an angry customer instead of from the system.

**Detect buying intent with bilingual keywords.**
`magkano`, `presyo`, `paano mag order`, `gusto ko` alongside the English
equivalents. An English-only keyword list would miss most of the actual buying
signals on a Philippine page.

**Log every conversation, not just the leads.**
The full log is what makes the knowledge base improvable. Reading the questions
the bot answered badly is the only way to find out what the doc is missing.

## Known gaps

- Static data for dedupe and rate limiting lives in the n8n instance, so it
  resets on restart
- Keyword-based intent detection misses phrasings not on the list; a classifier
  would generalise better
- No handoff to a human, the bot either answers or apologises

## Setup

Import all three workflows. Replace `REPLACE_WITH_YOUR_ACCESS_TOKEN` in the
Graph API nodes, `REPLACE_WITH_YOUR_AUTHORIZATION` in the follow-up workflow,
and `REPLACE_WITH_YOUR_GOOGLE_DOC_ID` in the ingestion workflow. Reconnect the
Gemini, Supabase, Google Docs, Google Sheets, and Gmail credentials.
