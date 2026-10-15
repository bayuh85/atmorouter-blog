---
layout: post
title: "grunt-proxy: a local proxy that trims your coding agent's prompts (and shows what it saved)"
date: 2026-10-15T17:25:31Z
description: "A small OpenAI- and Anthropic-compatible proxy for Claude Code, Cursor, and Cline that compresses prompts, asks for terse replies, and prints token and dollar savings per request."
canonical_url: https://dev.to/bayuh85/grunt-proxy-a-local-proxy-that-trims-your-coding-agents-prompts-and-shows-what-it-saved-40e2
tags: [ai, productivity, opensource, typescript]
---
*Originally published on [dev.to](https://dev.to/bayuh85/grunt-proxy-a-local-proxy-that-trims-your-coding-agents-prompts-and-shows-what-it-saved-40e2).*

This post walks through **grunt-proxy**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

{% raw %}

**Repo:** 👉 [github.com/bayuh85/grunt-proxy](https://github.com/bayuh85/grunt-proxy) (MIT, Node.js 20+, TypeScript)

## The problem

If you use Claude Code, Cursor, or Cline every day, you are sending a lot of text you didn't write: system prompts, tool descriptions, tool results, and your own "could you please…" phrasing. Every token of that is billed. And the replies often come back with a greeting, a recap, and a closing paragraph you'll never read.

None of that is a big deal per request. Across a day of agent loops, it adds up, and most of us have no idea by how much.

I wanted something that sits between the agent and the API, trims the obvious padding, asks the model to get to the point, and **tells me what it actually saved** — without breaking code or tool calls.

## What grunt-proxy does

grunt-proxy is a small local HTTP proxy for coding agents. For each request it:

1. **Shortens the prompt** you were about to send (filler phrases, articles, common short forms — configurable).
2. **Asks the model for a terse reply** via a short, stable instruction.
3. **Prints how many tokens and how many dollars that saved.**

What it deliberately does *not* do:

- Replies stay in normal English. The proxy doesn't role-play and doesn't rewrite the model's response.
- Code, JSON, and tool calls stay intact.

It speaks both the **OpenAI Chat Completions** API and the **Anthropic Messages** API (including the Anthropic SSE stream), so the same proxy works for Cursor, Cline, and Claude Code.

## Quickstart

Requires Node.js 20 or newer.

```bash
git clone https://github.com/bayuh85/grunt-proxy.git
cd grunt-proxy
cp .env.example .env
```

Put your upstream key in `GRUNT_API_KEY` in `.env`. The default upstream is AtmoRouter (keys look like `ar-...`), but you can point it anywhere — more on that below.

```bash
npm install
npm run dev
```

grunt-proxy now listens on `http://127.0.0.1:8642`. Try a request through it:

```bash
curl http://127.0.0.1:8642/v1/chat/completions \
  -H "content-type: application/json" \
  -H "authorization: Bearer $GRUNT_API_KEY" \
  -d '{
    "model": "atmo/deepseek-v4.1-flash",
    "messages": [
      {"role": "user", "content": "Could you please explain the configuration options in the documentation? Thank you!"}
    ]
  }'
```

(Model ids change — check your provider's current catalog before relying on an example id.)

Each request prints one line on stderr:

```text
grunt  atmo/deepseek-v4.1-flash  in 1,204→640 (−46.8%)  out 180  saved $0.001692
```

For the running total, hit `GET /savings.txt` or press Ctrl+C when the server is in the foreground:

```text
grunt-proxy savings
requests              12
failed                0
input tokens          48,200 → 21,040  (−56.3%)
input tokens saved    27,160
output tokens         6,410
est. input $ saved    $0.081480
projected output $    $0.000000
est. $ saved          $0.081480
upstream billed       –
```

To be clear, the README labels those numbers as **an illustration of the layout, not a benchmark**. Your savings depend on your prompts, your level setting, and your prices.

For a compiled build: `npm run build && npm start`.

## Setting it up with your agent

### Claude Code

Claude Code (and the Anthropic SDK) appends `/v1/messages` itself, so use the proxy origin **without** `/v1`:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:8642"
export ANTHROPIC_API_KEY="ar-your-key"
claude
```

grunt-proxy receives `POST /v1/messages`, compresses it, and forwards it upstream, streaming included. It sets `anthropic-version: 2023-06-01` if the client didn't send one.

### Cursor

1. Start grunt-proxy (`npm run dev`).
2. Open **Cursor Settings → Models**.
3. Set the OpenAI API key to your upstream key. If `GRUNT_API_KEY` is already set in `.env`, any non-empty placeholder works — the proxy replaces the client key.
4. Override the OpenAI base URL with `http://127.0.0.1:8642/v1`.
5. Add a custom model whose id matches a model on your upstream, e.g. `atmo/deepseek-v4.1-flash`.

### Cline

1. Start grunt-proxy.
2. Set the API provider to **OpenAI Compatible**.
3. Base URL: `http://127.0.0.1:8642/v1`
4. API key: your upstream key (or a placeholder when `GRUNT_API_KEY` is set).
5. Model ID: a catalog id from your provider.

Quick reference:

| Client | Base URL the client uses |
| --- | --- |
| Cursor, Cline, OpenAI SDKs | `http://127.0.0.1:8642/v1` |
| Claude Code, Anthropic SDK | `http://127.0.0.1:8642` |

## How it works

### What gets compressed (and what doesn't)

grunt-proxy only edits text the model will *read*: system and user text, assistant text, tool descriptions, and tool-result text. It leaves these alone:

- fenced code, inline code, and indented code lines
- URLs, file paths, and filenames
- JSON values, including tool-call arguments and tool parameter schemas
- camelCase and PascalCase identifiers
- image blocks and Anthropic `tool_use` / `thinking` blocks

### Levels

| Level | What changes |
| --- | --- |
| 0 | Nothing. |
| 1 | Courtesy and padded phrases ("could you please", "in order to", "thank you"). |
| 2 | Level 1, plus `a`/`an`/`the` and a few other fillers, plus short forms like `config`, `fn`, `param`, `impl`. |
| 3 | Level 2, plus `of`/`that`, some contractions, and shorter forms like `cfg`, `btwn`, `w/o`. |

Level 2 is the default. If an agent starts misreading instructions, drop to level 1. Level 3 is lossier; only use it once you've checked the output.

You can override per request with headers: `x-grunt: off` forwards the body unchanged, and `x-grunt-level: 0–3` / `x-grunt-terse: auto|always|off` override the config for one call.

### The terse instruction

With `GRUNT_TERSE=auto` (the default), the proxy adds a one-paragraph instruction only when compression already pays for it, or when the prompt is roughly 800+ estimated tokens. It tells the model to answer in normal English, skip the greeting and recap, and keep code and tool calls valid.

A nice detail: the instruction is the **same bytes for a given level**, so a provider's prompt cache can still reuse the prefix.

### Honest savings math

This is the part I appreciate most. The README is upfront about what's measured and what's estimated:

- The token delta is a **local before/after estimate**, not a clone of any provider's tokenizer. It's consistent, which is what a savings figure needs. When the upstream returns usage, the actual counts are stored too.
- Dollars are `tokens saved × price / 1,000,000`, using approximate public list prices for known model families, or `$1` / `$3` per million for unknown ids.
- **Set `GRUNT_PRICE_IN` / `GRUNT_PRICE_OUT` to what you actually pay**, otherwise the estimate won't match your invoice (and may overstate savings if your provider is cheaper than list price).
- Output terseness is *requested, not measured*. `projected output $` stays `$0` unless you explicitly set `GRUNT_ASSUME_OUTPUT_CUT`, which is a labeled guess.
- Failed upstream calls count as failures and add `$0`.
- The response headers `x-grunt-original-tokens`, `x-grunt-compressed-tokens`, and `x-grunt-saved-tokens` expose the estimate per call. Saved tokens can even be **negative** when the terse instruction costs more than compression saved — and the proxy says so instead of hiding it.

If you set `GRUNT_LOG_FILE`, you get a JSONL savings ledger (prompts are not written), and `grunt-proxy --report .grunt-savings.jsonl` reprints a report from it without starting the server.

### Safe defaults

- Binds to `127.0.0.1` by default, so the proxy doesn't become a public credit sink.
- No proxy-side request timeout by default, so long Claude Code or Cursor streams aren't cut off. Disconnecting the client aborts the upstream request.
- Oversized bodies (default limit 32 MiB) are rejected with HTTP 413.

## Credit where it's due

grunt-proxy is **inspired by [caveman](https://github.com/JuliusBrussee/caveman)** by Julius Brussee — the idea that you can strip a lot of words from what goes to and from a model without losing meaning. grunt-proxy takes that idea and packages it as a transparent proxy with a savings report, while keeping replies in normal English.

## About AtmoRouter: a cheap AI API gateway and OpenRouter alternative

Full disclosure: grunt-proxy's default upstream is AtmoRouter, but any OpenAI- or Anthropic-compatible upstream works. [AtmoRouter](https://atmorouter.dev) is a [cheap AI API gateway](https://atmorouter.dev) that speaks both the OpenAI Chat Completions API and the Anthropic Messages API, so one key reaches 100+ models from Cursor, Cline, Claude Code, or any compatible client. It is pay-per-token with no subscription, accepts QRIS and USDC, and prices sit well below official rates: AtmoRouter advertises an average of 83.7% below official prices, which makes it an affordable LLM API and a practical OpenRouter alternative for agent workloads.

## Try it

If you run coding agents all day and want to see where your tokens go, give it a spin:

👉 **[github.com/bayuh85/grunt-proxy](https://github.com/bayuh85/grunt-proxy)**

Issues and PRs welcome — especially reports of prompts where a compression level changed the model's behavior, since that's exactly what the level knob is for.

{% endraw %}
