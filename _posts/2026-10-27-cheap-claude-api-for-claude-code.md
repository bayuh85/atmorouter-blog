---
layout: post
title: "Cheapest Claude API for Claude Code (2026): Opus, Sonnet and Haiku Prices Compared"
date: 2026-10-27T13:00:00Z
description: "Cheap Claude API prices for 13 Claude models on AtmoRouter vs Anthropic's official pricing and OpenRouter, plus a 2-minute Claude Code setup with ANTHROPIC_BASE_URL or npx @atmorouter/helper. Prices checked Oct 27, 2026."
tags: [claude, claude-code, ai-api, pricing]
---

Claude Code burns tokens fast. One long agent session can read whole repositories, run tools in loops and use millions of tokens. At official prices, that adds up. This page compares what **Claude Opus, Sonnet, Haiku and Fable** cost per token on Anthropic's own API, on OpenRouter and on AtmoRouter. Then it shows how to point Claude Code at a **cheap Claude API** in a couple of minutes.

## Claude API prices: official vs OpenRouter vs AtmoRouter

*Prices checked Oct 27, 2026.* All prices are USD per 1M tokens, shown as input / output. AtmoRouter prices are a live snapshot taken Oct 27, 2026, 20:52 WIB (13:52 UTC) from the AtmoRouter catalog. "From" is the lowest live offer at that moment; "median offer" is the median across all live offers for that model, so it's closer to what you'd usually pay. AtmoRouter prices move during the day, and the cheapest offer can be out of capacity. Set a price cap in [Budgets](https://atmorouter.dev/docs) so you never pay more than you've agreed to. OpenRouter prices come from `openrouter.ai/api/v1/models`, and official prices from each provider's own pricing page. Percentages compare the cost of 1M input plus 1M output tokens with the official price.

| Model | Official | OpenRouter | AtmoRouter from | AtmoRouter median offer | vs official (from / median) |
|---|---|---|---|---|---|
| Claude Opus 5.5 | $4 / $20 | $4 / $20 | $0.48 / $2.4 | $1.08 / $5.4 | −88% / −73% |
| Claude Opus 5 | $5 / $25 | $5 / $25 | $0.6125 / $3.0625 | $1.35 / $6.75 | −88% / −73% |
| Claude Opus 4.8 | $5 / $25 | $5 / $25 | $0.6125 / $3.0625 | $1.35 / $6.75 | −88% / −73% |
| Claude Opus 4.7 | $5 / $25 | $5 / $25 | $0.6125 / $3.0625 | $1.35 / $6.75 | −88% / −73% |
| Claude Opus 4.6 | $5 / $25 | $5 / $25 | $0.125 / $0.625 | $0.25 / $1.25 | −98% / −95% |
| Claude Opus 4.5 | $5 / $25 | $5 / $25 | $0.6125 / $3.0625 | $1.35 / $6.75 | −88% / −73% |
| Claude Sonnet 5.5 | $2 / $10 | $2 / $10 | $0.245 / $1.225 | $0.54 / $2.7 | −88% / −73% |
| Claude Sonnet 5 | $2 / $10 | $2 / $10 | $0.245 / $1.225 | $0.59 / $2.95 | −88% / −70% |
| Claude Sonnet 4.6 | $3 / $15 | $3 / $15 | $0.0675 / $0.3375 | $0.075 / $0.375 | −98% / −98% |
| Claude Haiku 5.5 | $0.1 / $0.5 | $0.1 / $0.5 | $0.02475 / $0.12375 | $0.027 / $0.135 | −75% / −73% |
| Claude Haiku 4.5 | $1 / $5 | $1 / $5 | $0.2475 / $1.2375 | $0.2475 / $1.2375 | −75% / −75% |
| Claude Fable 5.1 | $10 / $50 | $10 / $50 | $2.475 / $12.375 | $2.7 / $13.5 | −75% / −73% |
| Claude Fable 5 | $10 / $50 | $10 / $50 | $2.475 / $12.375 | $2.7 / $13.5 | −75% / −73% |

*Claude Haiku 5.5's official price applies to prompts of up to 100k tokens.*

What the numbers say:

- **OpenRouter charges exactly Anthropic's list price** for every Claude model here, which matches OpenRouter's no-markup-on-inference policy.
- **On AtmoRouter, every Claude model was 70% or more below list** at the median offer in this snapshot. The lowest live offers were 75–98% below.
- **The biggest discounts were on Claude Sonnet 4.6 and Claude Opus 4.6.** In this snapshot, Sonnet 4.6 was about 98% below list even at the median offer, and Opus 4.6 about 95% below.

### What a heavy Claude Code day costs

Say a busy agent day uses **20M input tokens and 2M output tokens** on Claude Sonnet 5.5. This is an illustrative workload; your mix will differ.

- Official or OpenRouter: 20 × $2 + 2 × $10 = **$60**
- AtmoRouter at the snapshot's median offer: 20 × $0.54 + 2 × $2.70 = **$16.20**

## Set up Claude Code with AtmoRouter

AtmoRouter speaks the Anthropic Messages API natively, so Claude Code works by changing its base URL. You don't need a proxy.

### Option 1: environment variables

```bash
# set ATMOROUTER_API_KEY privately first (shell profile or secret manager)
export ANTHROPIC_BASE_URL="https://atmorouter.dev"
export ANTHROPIC_AUTH_TOKEN="$ATMOROUTER_API_KEY"
export ANTHROPIC_MODEL="YOUR_MODEL_ID"
claude
```

Note that the Anthropic base URL is the bare host, `https://atmorouter.dev`, with **no** `/v1` on the end. Replace `YOUR_MODEL_ID` with the exact Claude model ID from [atmorouter.dev/models](https://atmorouter.dev/models), and restart Claude Code after changing the environment.

### Option 2: the setup helper

The official helper finds model IDs, previews the change and backs up your existing settings before writing anything:

```bash
npx @atmorouter/helper models --search claude
npx @atmorouter/helper configure --target claude --model YOUR_MODEL_ID --write
npx @atmorouter/helper check-key   # zero-inference key check
```

You'll need Node.js 22 or newer. The helper never prints or saves your API key; it reads it from `ATMOROUTER_API_KEY`. For Claude Code, also set `ANTHROPIC_AUTH_TOKEN` to the same key. Full details are in the [helper docs](https://atmorouter.dev/docs/helper).

### Keep the price under control

Because AtmoRouter prices are live, set a ceiling once and forget about it. In **Budgets** you can require a minimum percentage below the official price. You can also send `x-max-input-price` and `x-max-output-price` headers per request. If no offer meets your limit, the request is rejected and you aren't charged. See [atmorouter.dev/docs](https://atmorouter.dev/docs).

## Which Claude model for Claude Code?

- **Opus (4.5 – 5.5)** for hard refactors and long, multi-step plans. Opus 4.6 was the cheapest Opus in this snapshot.
- **Sonnet (4.6 – 5.5)** as the everyday default. Sonnet 4.6 had the deepest discount of any Claude model.
- **Haiku (4.5, 5.5)** for quick edits, summaries and sub-agents.

Comparing gateways in general rather than just Claude? See [AtmoRouter vs OpenRouter](/atmorouter-blog/posts/atmorouter-vs-openrouter/) and [best OpenRouter alternatives in 2026](/atmorouter-blog/posts/best-openrouter-alternatives-2026/).

## Try it free

Create a key at [atmorouter.dev](https://atmorouter.dev) and send your first request to the free model **`atmo/deepseek-v4.1-flash`**. It costs $0 per token, but your account needs at least $1 in lifetime deposits and a lifetime free quota applies. Then switch to any of the 180+ models by changing the model ID. The docs are at [atmorouter.dev/docs](https://atmorouter.dev/docs) and every model ID is listed at [atmorouter.dev/models](https://atmorouter.dev/models).

## FAQ

### What is the cheapest way to use the Claude API?

In our Oct 27, 2026 snapshot, AtmoRouter's median offer for every Claude model was at least 70% below Anthropic's list price, and its lowest offers were 75–98% below. OpenRouter charged the same as Anthropic's list price. AtmoRouter prices are live, so set a cap in Budgets.

### How do I use Claude Code with a different API provider?

Set ANTHROPIC_BASE_URL to the gateway's Anthropic endpoint (https://atmorouter.dev for AtmoRouter, with no /v1), set ANTHROPIC_AUTH_TOKEN to your key and ANTHROPIC_MODEL to an exact model ID, then restart Claude Code. Or run npx @atmorouter/helper configure --target claude.

### Do I need a proxy or LiteLLM to run Claude Code on AtmoRouter?

No. AtmoRouter implements the Anthropic Messages API, including streaming, so Claude Code connects directly.

### Is the Claude API on OpenRouter cheaper than Anthropic?

Not for the Claude models we checked. OpenRouter's per-token prices matched Anthropic's list prices exactly, and OpenRouter charges a fee when you buy credits.

### Will I be charged if a Claude Code request fails?

On AtmoRouter, requests that fail before any output are not charged. An interrupted stream is charged only for valid reported usage, and responses with unverifiable billing metadata are not charged.

<script type="application/ld+json">
{
 "@context": "https://schema.org",
 "@type": "FAQPage",
 "mainEntity": [
  {
   "@type": "Question",
   "name": "What is the cheapest way to use the Claude API?",
   "acceptedAnswer": {
    "@type": "Answer",
    "text": "In our Oct 27, 2026 snapshot, AtmoRouter's median offer for every Claude model was at least 70% below Anthropic's list price, and its lowest offers were 75–98% below. OpenRouter charged the same as Anthropic's list price. AtmoRouter prices are live, so set a cap in Budgets."
   }
  },
  {
   "@type": "Question",
   "name": "How do I use Claude Code with a different API provider?",
   "acceptedAnswer": {
    "@type": "Answer",
    "text": "Set ANTHROPIC_BASE_URL to the gateway's Anthropic endpoint (https://atmorouter.dev for AtmoRouter, with no /v1), set ANTHROPIC_AUTH_TOKEN to your key and ANTHROPIC_MODEL to an exact model ID, then restart Claude Code. Or run npx @atmorouter/helper configure --target claude."
   }
  },
  {
   "@type": "Question",
   "name": "Do I need a proxy or LiteLLM to run Claude Code on AtmoRouter?",
   "acceptedAnswer": {
    "@type": "Answer",
    "text": "No. AtmoRouter implements the Anthropic Messages API, including streaming, so Claude Code connects directly."
   }
  },
  {
   "@type": "Question",
   "name": "Is the Claude API on OpenRouter cheaper than Anthropic?",
   "acceptedAnswer": {
    "@type": "Answer",
    "text": "Not for the Claude models we checked. OpenRouter's per-token prices matched Anthropic's list prices exactly, and OpenRouter charges a fee when you buy credits."
   }
  },
  {
   "@type": "Question",
   "name": "Will I be charged if a Claude Code request fails?",
   "acceptedAnswer": {
    "@type": "Answer",
    "text": "On AtmoRouter, requests that fail before any output are not charged. An interrupted stream is charged only for valid reported usage, and responses with unverifiable billing metadata are not charged."
   }
  }
 ]
}
</script>
