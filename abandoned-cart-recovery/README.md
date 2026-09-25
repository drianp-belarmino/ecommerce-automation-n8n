# Abandoned Cart Recovery

A three-email win-back sequence for a Shopify store. Each email is written by an
LLM against the customer's actual cart, and the workflow re-checks Shopify
before every send so nobody who already bought gets chased.

## Flow

```
Shopify abandoned checkout
  └── Has an email address?  ── no ──> stop
       │ yes
       ├── wait 1 hour
       │     └── query Shopify: did they purchase?
       │           ├── purchased ──> stop
       │           └── still abandoned ──> generate email 1 ──> send ──> log
       ├── wait 23 hours
       │     └── same purchase re-check ──> email 2
       └── final wait
             └── same purchase re-check ──> email 3
```

## Decisions worth explaining

**Re-query Shopify before every send, not just at the start.**
A cart abandoned at 9am is often a purchase by 3pm, through a different device
or a phone call. Checking once at the top of the sequence and then firing three
emails on a timer means emailing paying customers about items they already own.
That erodes trust faster than a missed recovery costs. Every send is gated on a
fresh `financial_status` check.

**Generate each email instead of templating it.**
The LLM writes against the specific cart contents and how long it has been
sitting, so email three reads differently from email one. A static template
sequence gets recognised and filtered.

**Parse the model output before sending it.**
A dedicated parse step sits between generation and Gmail. A model that returns
a preamble, a markdown fence, or a refusal should not become the body of a
customer email.

**Spread the sequence across roughly a day.**
One hour, then 23 hours, then a final gap. Fast enough to catch live intent,
slow enough not to read as harassment.

**Log every send to Sheets.**
Not for vanity metrics. When the owner asks why a customer received a
particular email, the answer needs to exist.

## Known gaps

- No unsubscribe handling in the workflow itself; it relies on the sending
  account's policy
- Wait nodes hold state in n8n, so a long outage mid-sequence drops in-flight
  carts
- No A/B testing on the generated copy

## Setup

Import the JSON and replace `REPLACE_WITH_SHOPIFY_ACCESS_TOKEN` in the
HTTP nodes, then reconnect the Shopify, Mistral, Gmail, and Google Sheets
credentials.
