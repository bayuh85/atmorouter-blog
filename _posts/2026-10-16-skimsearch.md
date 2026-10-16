---
layout: post
title: "skimsearch: stop pasting grep dumps into your coding agent"
date: 2026-10-16T02:39:49Z
description: "A Python CLI and MCP server that indexes your repo into SQLite FTS5 and hands your coding agent a token-budgeted deck of ranked snippets instead of a wall of grep output."
canonical_url: https://dev.to/bayuh85/skimsearch-stop-pasting-grep-dumps-into-your-coding-agent-hlj
tags: [ai, python, opensource, devtools]
---
*Originally published on [dev.to](https://dev.to/bayuh85/skimsearch-stop-pasting-grep-dumps-into-your-coding-agent-hlj).*

This post walks through **skimsearch**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

{% raw %}

**Repo:** 👉 [github.com/bayuh85/skimsearch](https://github.com/bayuh85/skimsearch) (MIT, Python 3.10+)

## The problem

Ask a coding agent "where is the late fee calculated?" and watch what happens. It runs `grep -rn "late fee"`, gets back dozens of hits from code, docs, vendored dependencies, and tests, and pastes the whole dump into its context. Then it opens three wrong files before finding the right function.

Every one of those lines is tokens you pay for, re-read on every turn, crowding out the few lines that actually answer the question.

## What skimsearch does

skimsearch indexes the tree once, then answers with a short deck of ranked snippets that fits a token budget.

1. **Index.** `skimsearch index` folds code and docs into *skim cards*: one function, class, heading, or text window each, with its path and line range. Cards live in a compact SQLite file at `.skimsearch/index.sqlite`, ranked offline with SQLite FTS5 (BM25).
2. **Search.** `skimsearch search` returns the best cards, trimmed so the whole deck stays inside a budget (700 tokens by default). Lexical mode never touches the network.
3. **Ask.** `--mode semantic` lets a cheap model turn a prose question into a few lexical probes, then searches locally. `--mode ask` also returns a short answer that cites only the cards it used.

It respects `.gitignore`, supports a `.skimignore`, and skips dependency folders, lockfiles, and secret files like `.env` and `.pem`.

## Real use case: finding the late fee

The repo ships a small sample project, `examples/acme-billing`, with code, docs, a vendored folder, and a fake `.env`. Install and index it:

```bash
git clone https://github.com/bayuh85/skimsearch.git
cd skimsearch
python -m venv .venv
source .venv/bin/activate
pip install -e .

skimsearch index examples/acme-billing
skimsearch search "where is the late fee calculated" --path examples/acme-billing
```

The output:

```text
skimsearch  mode=lexical  cards=4  tokens=322/700

1. src/billing/invoices.py:17-28  function apply_late_fee  (python, score 1.71)
    def apply_late_fee(invoice: dict, as_of: date) -> dict:
        """The late fee is calculated from the subtotal after the grace window."""
        if invoice.get("settled"):
            return invoice
        overdue = (as_of - invoice["due"]).days
        if overdue <= GRACE_DAYS:
            return invoice
        fee = round(invoice["subtotal"] * LATE_FEE_RATE, 2)
        ...

2. docs/billing.md:3-5  heading Late fees  (markdown, score 1.50)
    ## Late fees

    A fee of 1.5 percent is added after a three-day grace period.
```

322 tokens, with the right function first and the matching docs right below it. The vendored code and the fake `.env` never show up. That's what the agent reads instead of a raw grep dump.

If you'd rather get a direct answer:

```bash
export ATMOROUTER_API_KEY=ar-your-key-here
skimsearch search "where is the late fee calculated" --path examples/acme-billing --mode ask
```

## Plugging it into Cursor or Claude Code (MCP)

`skimsearch serve` runs a stdio MCP server (and builds the index if it's missing). For Claude Code, add this to `.mcp.json`:

```json
{
  "mcpServers": {
    "skimsearch": {
      "command": "skimsearch",
      "args": ["serve", "/absolute/path/to/repo"],
      "env": { "ATMOROUTER_API_KEY": "ar-your-key-here" }
    }
  }
}
```

Cursor uses the same block in `.cursor/mcp.json`. The agent gets two tools: `skim_search` (query, mode, budget, limit, format) and `skim_status`. Use `lexical` in tight loops and `ask` for prose questions.

## Handy commands

```bash
skimsearch index . --force          # rebuild
skimsearch index . --embed          # also store /embeddings vectors
skimsearch search "session ttl" -C . --budget 500
skimsearch search "session ttl" --mode ask --format json
skimsearch status .
```

The test suite (44 tests) runs fully offline: `pip install -e ".[dev]" && pytest -q`.

## About AtmoRouter (the default model endpoint)

Semantic and ask modes call an OpenAI-compatible chat API, and the default base URL is `https://atmorouter.dev/v1`. [AtmoRouter](https://atmorouter.dev) is a cheap AI API gateway and an OpenRouter alternative: 109 models behind one key, compatible with both OpenAI and Anthropic clients, pay per token, with prices averaging 83.7% below official rates. It also accepts QRIS and USDC. That fits this tool's job: query rewriting should cost far less than the coding model it's saving tokens for. The default model is `atmo/deepseek-v4.1-flash`.

Prefer the Anthropic Messages API? Set `SKIMSEARCH_API_STYLE=anthropic`. Want another provider? Point `SKIMSEARCH_BASE_URL` at any OpenAI-compatible endpoint. And lexical search needs no key at all.

## Try it

If your agent keeps drowning in grep output, give it a deck instead:

👉 **[github.com/bayuh85/skimsearch](https://github.com/bayuh85/skimsearch)**

Issues with a query that ranks badly (plus a small sample repo) are the most useful feedback you can send.
{% endraw %}
