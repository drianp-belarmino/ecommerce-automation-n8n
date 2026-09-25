# E-commerce Automation Workflows (n8n)

Two production automations built for a Shopify apparel store: an abandoned cart
recovery sequence and an AI customer-service bot on Facebook Messenger.

Both ran live. The client name and every credential have been removed, and the
workflow JSON is exported as a template. These are the real workflows, not
tutorial rebuilds.

## What is here

| | What it does | Stack |
|---|---|---|
| [Abandoned cart recovery](abandoned-cart-recovery/) | Three-email win-back sequence that re-checks whether the customer already bought before every send | n8n, Shopify Admin API, Mistral, Gmail, Google Sheets |
| [Messenger bot](messenger-bot/) | AI support bot answering from a maintained knowledge base, with buying-intent detection and hot-lead logging | n8n, Facebook Graph API, Gemini, Supabase, Google Docs, Google Sheets |

## The thread running through both

Neither of these is a happy-path demo. The work that mattered was in what
happens when things go wrong:

- **Nothing sends blind.** The cart sequence re-queries Shopify before every
  email, so a customer who already purchased never gets chased.
- **Failures are loud, in layers.** The bot alerts to a sheet, then emails the
  owner, then alerts again if the alert itself fails.
- **External calls retry.** Providers return 503s. A single bad response should
  not end a customer conversation.
- **Spam and duplicates are filtered before they cost anything.** Facebook
  re-sends messages, and one user can flood a workflow. Both are handled
  upstream of any paid API call.

## Running these yourself

Import the JSON into n8n, then replace every value marked
`REPLACE_WITH_YOUR_...` and reconnect the credentials. Credential references in
the exports are n8n's internal ids and names only; no secret is included.
