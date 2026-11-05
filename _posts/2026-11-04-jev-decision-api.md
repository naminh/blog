---
layout: post
title:  "Jev and decision APIs"
date:   2026-11-04 09:00:00 +0000
categories: "AI-Posts"
---

## Stop paying frontier prices for if-statements

A lot of LLM calls in production aren't "write me an essay" calls. They're "is this ticket about billing?", "which team owns this?" or "how urgent is it?". You pay for generation, then parse the text and hope it matches your schema.

A newer class of **decision APIs** skips the generation. TypeSafe AI's **Jev** is one example, and it's what I'm looking at here.

### What it is

TypeSafe calls Jev a "System One" model. You send some `state` (text or JSON) plus named, typed questions, and you get typed answers with probabilities back. There's no text generation and nothing to parse. Questions are answered in parallel, in one request:

- **noul**: yes/no, returned as a probability from 0 to 1
- **choice**: pick one option from up to 255
- **score**: rate on an ordered scale, returned as a probability-weighted score

A request looks like this (taken from their docs):

```json
POST https://api.typesafe.ai/v1/systemone

{
  "model": "jev-latest",
  "state": "I was charged twice this month. Please refund the extra charge.",
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which team should handle this?",
      "criteria": {
        "shipping": "Delivery problems",
        "returns": "Refunds and returns",
        "billing": "Charges and invoices"
      }
    }
  }
}
```

and you get back the chosen option, a probability for every option and a confidence value.

### Why it's interesting

The headline claims are about cost and speed. These are the vendor's numbers, not independent benchmarks, so take them with salt:

- Input is priced at about **$0.042 per million tokens**, and output tokens are free.
- They report **70 to 500 ms** end to end, versus seconds for frontier LLMs.
- Their homepage cites roughly 190x faster and 440x cheaper on "System One shaped" workflows. They themselves say that's the high end of what you'd see in the real world.

Even if you divide those by ten, that's a big difference for anything sitting in a hot path.

### Where it fits

Think of it as a smart `if` statement:

- **Triage and routing**: tickets, emails, leads
- **Guardrails**: a cheap pre-check before an expensive call
- **Filtering at scale**: classify millions of documents without a million LLM bills

My favourite pattern is **routing between models**. Ask a decision model two questions about each incoming request: how hard is it (`score`), and does it need tools or long context (`noul`). Send the easy majority to a small, cheap model and escalate only the hard tail to a frontier model. Jev's docs don't describe this exact use, so it's my suggestion. They do describe the building blocks, such as routing on probability thresholds, screening with a yes/no question first, and sending low-confidence cases to a human. You can swap in any small model with structured outputs for the same trick.

### Caveats

- **Confidence isn't accuracy.** The docs are upfront that it only measures how concentrated the probabilities are. Pick thresholds on your own labelled data.
- **English works best.** Other languages are handled less well.
- **Evidence is mostly vendor-supplied.** Their reference answers come from frontier LLMs, and I found no independent evals or calibration curves. Run your own before trusting it.
- **Young vendor.** Check pricing, rate limits and data handling before you build anything critical on it.

### Take-away

Not every AI task needs a language model that talks. If your task is classify, route or score, a decision API is worth an afternoon of testing against your current LLM calls. The savings could be large.

**Sources:** [TypeSafe AI](https://typesafe.ai), [Introducing System One models and Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev), [Jev docs](https://docs.typesafe.ai/)
