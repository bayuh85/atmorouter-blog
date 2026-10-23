---
layout: post
title: "graphloom: give your coding agent a map of the repo instead of letting it grep 40 files"
date: 2026-10-23T02:30:00Z
description: "An open-source Python CLI and MCP server that turns a repository (code, SQL, docs, config) into a queryable knowledge graph with file:line evidence on every edge. The graph is built offline; only questions go to a cheap model through AtmoRouter."
canonical_url: https://dev.to/bayuh85/graphloom-give-your-coding-agent-a-map-of-the-repo-instead-of-letting-it-grep-40-files-ai5
tags: [ai, python, opensource, productivity]
---
*Originally published on [dev.to](https://dev.to/bayuh85/graphloom-give-your-coding-agent-a-map-of-the-repo-instead-of-letting-it-grep-40-files-ai5).*

This post walks through **graphloom**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

{% raw %}

**Repo:** 👉 [github.com/bayuh85/graphloom](https://github.com/bayuh85/graphloom) (MIT, Python 3.10+, standard library only, 32 offline tests)

## The problem

Ask a coding agent "what touches the invoices table?" and watch what it does. It greps for `invoices`, opens the first match, reads 300 lines, greps again, opens the schema, opens a doc, opens another service. Ten tool calls later it has burned a lot of context and still might miss the function that builds the SQL string two modules away.

The information it needs is not hidden. Imports, calls, `CREATE TABLE` statements and the backtick names in your docs are all sitting in the repo. They just are not connected anywhere the agent can look them up.

So I wanted a map: build it once, cheaply and offline, then let the agent ask short questions against it and get back a few lines with `file:line` evidence instead of whole files.

## What graphloom does

`graphloom build` walks a repo and writes three files into `.graphloom/`:

| File | What it is |
| --- | --- |
| `graph.json` | Versioned graph. Every edge has a provenance tag and a `file:line` excerpt. |
| `REPORT.md` | Hub nodes by degree, isolated modules, cross-layer links, suggested questions. |
| `graph.html` | One self-contained interactive viewer. No CDN, works offline. |

What it understands, all without a model:

- **Python** through the stdlib `ast` module: modules, classes, functions, imports, calls
- **JS/TS** with a small scanner: imports, exports, calls, `fetch("/route")` URLs
- **SQL**: `CREATE TABLE` nodes and foreign keys, plus edges from code whose SQL strings name a table
- **Markdown**: heading nodes linked to the symbols you wrote in backticks
- **TOML/YAML/JSON**: config keys, linked to the code that reads them

Then you query the graph instead of the tree: `query`, `path`, `explain` (offline) and `ask` (one LLM call over a small subgraph). `graphloom serve` exposes the same tools over MCP for Cursor and Claude Code.

## Quickstart on the bundled example

The repo ships a small wholesale checkout app (`examples/harbor-shop`): Python services, a SQL schema, a browser cart and two docs.

```bash
git clone https://github.com/bayuh85/graphloom && cd graphloom
python3 -m venv .venv && . .venv/bin/activate
pip install -e .
cd examples/harbor-shop
graphloom build .
```

```text
nodes: 67
edges: 108
  parsed: 75
  linked: 33
  llm: 0
wrote graph: .graphloom/graph.json
wrote report: .graphloom/REPORT.md
wrote html: .graphloom/graph.html
```

Now the question from the intro. "From the HTTP route, how do we end up sending a receipt email?"

```bash
graphloom path post_invoice send_receipt
```

```text
path: post_invoice -> send_receipt
resolved: fn:backend/app.py:post_invoice -> fn:backend/notify/emailer.py:send_receipt
hops: 4
directed: yes

1. fn:backend/app.py:post_invoice
   invokes [parsed] -> fn:backend/billing/invoices.py:create_invoice
   backend/app.py:10  return create_invoice(payload, settings)

2. fn:backend/billing/invoices.py:create_invoice
   invokes [parsed] -> fn:backend/billing/payments.py:record_payment
   backend/billing/invoices.py:39  record_payment(str(payload["invoice_id"]), total, "card", str(payload["email"]))

3. fn:backend/billing/payments.py:record_payment
   invokes [parsed] -> fn:backend/notify/emailer.py:queue_receipt
   backend/billing/payments.py:12  queue_receipt(email, invoice_id)

4. fn:backend/notify/emailer.py:queue_receipt
   invokes [parsed] -> fn:backend/notify/emailer.py:send_receipt
   backend/notify/emailer.py:5  send_receipt(email, invoice_id)
```

(Trimmed slightly for width.) Four hops, each with the exact line that makes the call. An agent can cite that, or open just those four lines.

And "what touches the invoices table?":

```bash
graphloom query invoices --budget 1600
```

```text
## tbl:invoices
kind: table   layer: data   name: invoices
at: sql/schema.sql:17
columns: id, customer_id, sku, total_cents, status
edges:
  -> ties [parsed] tbl:customers (customers)  sql/schema.sql:17
  -> ties [parsed] tbl:products (products)  sql/schema.sql:17
  <- mentions [linked] doc:docs/billing.md:h:creating-an-invoice  docs/billing.md:9
  <- ties [parsed] tbl:payments (payments)  sql/schema.sql:25
  <- touches [linked] fn:backend/billing/invoices.py:create_invoice  backend/billing/invoices.py:35  "INSERT INTO invoices (customer_id, sku, total_cents, status) "
```

The `--budget` flag matters for agents: the output stops at roughly that many tokens and says `truncated: yes` when it does.

## Provenance: what was read vs. what was guessed

Every edge carries one of three tags:

| Tag | Meaning |
| --- | --- |
| `parsed` | Read straight from syntax: an import, a call to a known function, a foreign key |
| `linked` | Resolved afterwards: a backtick in a doc, a table name inside a SQL string, a `fetch("/invoices")` matched to a route |
| `llm` | Added by the optional semantic pass, and kept only if it anchors to a node that already exists |

This is the part I care about. When an agent says "`create_invoice` writes to `invoices`", you can see that came from a `linked` edge on line 35, not from the model's imagination.

## A real repo: httpx

The toy example is nice, so I also pointed it at [encode/httpx](https://github.com/encode/httpx) (commit `b5addb6`, about 280 KB of Python plus the docs).

```text
$ time graphloom build .
nodes: 1767
edges: 3388
  parsed: 2244
  linked: 1144
  llm: 0

real    0m0.494s
```

Half a second, no network. The hubs table in `REPORT.md` reads like a cheat sheet of the codebase: `Response` (degree 160), `Client` (152), `MockTransport` (101), `Request` (99), `AsyncClient` (84).

How does `Client.get` reach `send`?

```bash
graphloom path fn:httpx/_client.py:Client.get fn:httpx/_client.py:Client.send
```

```text
hops: 2
directed: yes

1. fn:httpx/_client.py:Client.get
   invokes [parsed] -> fn:httpx/_client.py:Client.request
   httpx/_client.py:1053  return self.request(

2. fn:httpx/_client.py:Client.request
   invokes [parsed] -> fn:httpx/_client.py:Client.send
   httpx/_client.py:825  return self.send(request, auth=auth, follow_redirects=follow_redirects)
```

That answer is 609 bytes. `httpx/_client.py` alone is 65,713 bytes, which is what an agent typically reads to find the same thing.

To be fair about the limits: graphloom does not do type inference. Asking for a path from `Client.get` to `HTTPTransport.handle_request` comes back `directed: no`, routed through the "Custom transports" doc heading, because the real call is `self._transport.handle_request(...)` and a static scan can't know what `self._transport` is. The output tells you that (`directed: no`, `linked` edges), so the agent knows to go read the code at that point instead of trusting the path.

## Where the model comes in (and why it stays cheap)

The graph never needs a model. Two optional features do:

- `graphloom build --semantic --max-calls 4` sends **doc chunks only** (never source code) to a model, which proposes concepts and anchors. Results are cached by content hash, so a second build of unchanged docs costs nothing.
- `graphloom ask "question"` retrieves a small subgraph and asks the model to answer citing node ids and `file:line`.

For `ask "how does Client.get end up sending the request?"` on httpx, the retrieved context sent to the model is about 10 KB, roughly 2.5k tokens. That's one small call per question, not a whole file per tool call.

Because the calls are small and frequent, the price per token matters more than the model's size. graphloom defaults to [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway with an OpenAI-compatible API (and an Anthropic-compatible one), and an OpenRouter alternative where one key covers 109 models at an average of 83.7% below official prices. The default model is `zai/glm-5.3-flash`.

```bash
export ATMOROUTER_API_KEY=ar-your-key-here
graphloom build . --semantic --max-calls 4
graphloom ask "what touches the invoices table?"
```

It's not locked in. Any OpenAI-compatible endpoint works:

| Variable | Default |
| --- | --- |
| `GRAPHLOOM_BASE_URL` | `https://atmorouter.dev/v1` |
| `GRAPHLOOM_API_KEY` | falls back to `ATMOROUTER_API_KEY` |
| `GRAPHLOOM_MODEL` | `zai/glm-5.3-flash` |
| `GRAPHLOOM_API_STYLE` | `openai` (or `anthropic` to POST `{base}/messages`) |

## Using it from Cursor or Claude Code

Build once, then register the MCP server. The server speaks newline-delimited JSON-RPC over stdio and exposes `graph_query`, `graph_path`, `graph_explain` and `graph_ask`.

```json
{
  "mcpServers": {
    "graphloom": {
      "command": "graphloom",
      "args": ["serve", "--graph", ".graphloom/graph.json"]
    }
  }
}
```

Put that in `.cursor/mcp.json` for Cursor or `.mcp.json` for Claude Code. Then tell the agent to try `graph_path` / `graph_query` before grepping. Three of the four tools are offline, so you only pay when it calls `graph_ask`.

## Try it

- Code: [github.com/bayuh85/graphloom](https://github.com/bayuh85/graphloom)
- Default gateway: [atmorouter.dev](https://atmorouter.dev)

It's MIT licensed, and the extractors are small, readable files (`extract_python.py`, `extract_sql.py`, ...), so adding Go or Rust support is a contained PR. If it saves your agent some file reads, a star on GitHub helps other people find it.

{% endraw %}
