---
layout: post
title: "auditsift: security-audit every PR with an AI agent, without drowning in false positives"
date: 2026-10-21T02:20:30Z
description: "A Python CLI and MCP server that maps your attack surface, runs offline security rules, then has one model explain each finding and a second, cheaper model confirm it against the real code. Output is SARIF for GitHub code scanning."
canonical_url: https://dev.to/bayuh85/auditsift-security-audit-every-pr-with-an-ai-agent-without-drowning-in-false-positives-24cm
tags: [security, python, opensource, ai]
---
*Originally published on [dev.to](https://dev.to/bayuh85/auditsift-security-audit-every-pr-with-an-ai-agent-without-drowning-in-false-positives-24cm).*

This post walks through **auditsift**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

{% raw %}

**Repo:** 👉 [github.com/bayuh85/auditsift](https://github.com/bayuh85/auditsift) (MIT, Python 3.10+, 64 offline tests)

## The problem

Ask a coding agent to "review this PR for security issues" and you usually get back a long, confident list. Some of it is real. A lot of it is plausible-sounding noise: an "SQL injection" in a query that is already parameterized, a "hardcoded secret" that is a test fixture, severities that change every time you ask. There is no file and line you can trust, and nothing your CI can parse.

Then there is the bill. Running a flagship model over every pull request "just in case" is the kind of habit teams drop after the first invoice.

What I wanted was boring and reliable: cheap offline checks first, a model only for the handful of candidates worth explaining, and a second opinion that has to quote the actual code before a finding counts.

## What auditsift does

auditsift runs an audit in phases. The first two need no API key and never touch the network.

| Phase | What it does | Needs a key? |
| --- | --- | --- |
| `map` | Languages, entry points, routes, dependency manifests, redacted secret-like strings | No |
| `scan` | Offline rules: hardcoded secrets, SQL built by string interpolation, `shell=True` / `os.system` with input, `eval`/`exec`, `pickle` and `yaml.load`, weak hashes, `verify=False`, path traversal, SSRF, permissive CORS | No |
| `triage` | Sends only the candidate snippet to a model for impact and a fix | Yes |
| `verify` | A second, independent model must confirm or reject each candidate, quoting a snippet that really exists in the file | Yes |
| `report` | JSON, SARIF 2.1.0 and a short Markdown summary, plus `--fail-on high` for CI | No |

The verify step is the part I care about most. It doesn't see the triage write-up, so it can't just agree with it. A "confirmed" verdict only counts when the quoted evidence is found in the code. Rejected findings are dropped from SARIF and counted in the summary, so you can see how much noise was removed.

## Real use case: gate a Flask PR in CI

The repo ships a deliberately broken sample app in `examples/flask-notes`. Here is the offline scan:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
auditsift scan examples/flask-notes
```

```text
auditsift  examples/flask-notes
13 findings kept, 0 rejected

high     AS-SQLI     examples/flask-notes/app.py:28  SQL query is built with string interpolation.
          impact: An attacker who controls the interpolated value can change the query.
          fix: Use a parameterized query and pass values as bound parameters.
high     AS-DESER    examples/flask-notes/app.py:41  Untrusted data can reach yaml.load.
          fix: Use yaml.safe_load, or a format such as JSON that does not construct objects.
high     AS-SSRF     examples/flask-notes/app.py:53  Outbound request target includes external input (target).
high     AS-TLS      examples/flask-notes/app.py:53  TLS certificate verification is disabled.
high     AS-SHELL    examples/flask-notes/app.py:60  Shell command is built from external input.
...
cost estimate
  map      local    0 in /  0 out  (0 calls)
  scan     local    0 in /  0 out  (0 calls)
```

Secrets are redacted before they are printed, written to a report or sent to a model.

For a pull request you only care about the lines that changed, so `--diff` filters findings to the `git diff` against a ref:

```bash
auditsift audit . --diff origin/main --fail-on high --out results
```

Exit code `1` means a kept finding is at or above the gate, so CI blocks the merge. `results/` holds `auditsift.json`, `auditsift.sarif` and `auditsift.md`.

The GitHub Actions job is short:

```yaml
name: auditsift sarif
on:
  pull_request:
permissions:
  contents: read
  security-events: write
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install .
      - run: >
          auditsift audit . --diff origin/${{ github.base_ref }}
          --offline --fail-on high --out results
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: results/auditsift.sarif
```

`--offline` keeps the required check free. When you want explanations and a second opinion on the diff, add your key as a repo secret and drop `--offline`. Findings show up in the Security tab like any other code-scanning tool.

## Adding the models without the flagship bill

auditsift keeps the model work small on purpose. The scan narrows a whole repo down to a few snippets, and only those go out. `--max-llm` caps how many candidates are sent at all. Every run prints a token ledger per phase, so you can see exactly what triage and verify cost.

The default endpoint is [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative. That matters here because the setup wants two different models through one key: a decent triage model and a cheaper verify model.

```bash
export ATMOROUTER_API_KEY=ar-your-key-here
export AUDITSIFT_TRIAGE_MODEL=openai/gpt-4o-mini
export AUDITSIFT_VERIFY_MODEL=openai/gpt-4.1-nano
auditsift audit examples/flask-notes --fail-on high --out results
```

AtmoRouter gives you 109 models behind one OpenAI- and Anthropic-compatible key, pay per token, at an average of 83.7% below official prices. Running two passes on every PR stops being a budget conversation. If you'd rather use another provider, set `AUDITSIFT_BASE_URL` and `AUDITSIFT_API_KEY` to any OpenAI-compatible endpoint, or `AUDITSIFT_API_STYLE=anthropic` for `/v1/messages`.

## Use it from Cursor or Claude Code

`auditsift serve` is an MCP server over stdio with `auditsift_map`, `auditsift_scan` and `auditsift_audit`. Map and scan stay offline, and `auditsift_audit` only calls models when the agent passes `use_llm: true`, so an agent doesn't spend tokens by accident.

```json
{
  "mcpServers": {
    "auditsift": { "command": "auditsift", "args": ["serve"] }
  }
}
```

```bash
claude mcp add auditsift -- auditsift serve
```

A good agent loop: scan the diff, read the candidates, and only ask for the paid audit on the ones that look real.

## Try it

- Code: [github.com/bayuh85/auditsift](https://github.com/bayuh85/auditsift)
- Gateway used by default: [atmorouter.dev](https://atmorouter.dev)

It's MIT licensed and the rule set is small and readable, so adding a rule for your own framework is a short PR. If it catches something real in your repo, a star on GitHub helps other people find it.

{% endraw %}
