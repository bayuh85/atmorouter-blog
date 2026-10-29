---
layout: post
title: "rankprobe: an SEO and AI-crawler audit that fails your PR before a stray noindex ships"
date: 2026-10-29T02:33:47Z
description: "A stdlib-only Python CLI that runs 36 deterministic SEO and AI-search checks (canonical, robots.txt and AI crawlers, JSON-LD, llms.txt) with evidence, gates CI with SARIF, and asks a cheap model for a fix plan from the findings only."
canonical_url: https://dev.to/bayuh85/rankprobe-an-seo-and-ai-crawler-audit-that-fails-your-pr-before-a-stray-noindex-ships-1h1g
tags: [seo, python, opensource, webdev]
---
*Originally published on [dev.to](https://dev.to/bayuh85/rankprobe-an-seo-and-ai-crawler-audit-that-fails-your-pr-before-a-stray-noindex-ships-1h1g).*

This post walks through **rankprobe**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

{% raw %}
I cross-post my dev.to articles to a small Jekyll site on GitHub Pages. Last week I pointed a new audit tool at that site and it told me 15 out of 15 pages had a meta description Google will cut off, 15 out of 15 titles were too long, no page had an `og:image`, and the origin served no `robots.txt`, no sitemap and no `llms.txt`. None of that was visible from the browser. Every page looked fine.

That tool is **rankprobe**, and this post shows what it does on a real site and how to wire it into CI.

**Repo:** [github.com/bayuh85/rankprobe](https://github.com/bayuh85/rankprobe) (MIT, Python 3.10+, standard library only, 80 offline tests)

## The problem

SEO breakage is silent. A `noindex` left over from staging, a canonical pointing at the wrong host, a robots.txt that blocks GPTBot and ClaudeBot because someone copied a "block AI" snippet, JSON-LD with a trailing comma. The page renders, the PR gets approved, and you find out weeks later from Search Console, or from the fact that no AI search engine ever cites you.

The usual options are a paid SEO suite (slow, per-seat, not in your CI) or handing the HTML to a chat model and asking "what's wrong with my SEO?", which spends tokens rediscovering things a parser settles in milliseconds, and sometimes invents issues.

I wanted the boring part to be deterministic and the model part to be optional and small.

## What rankprobe does

- `rankprobe audit <url-or-file>` fetches the page (or reads a local HTML file) and runs 36 checks: status and redirect chain, meta robots and `X-Robots-Tag`, canonical, hreflang, title and description length, one `h1`, heading order, missing `alt`, `lang`, viewport, page weight, Open Graph and Twitter tags, JSON-LD syntax and required fields, plus `robots.txt`, sitemap and `llms.txt` at the origin. Every finding carries the evidence it found.
- `rankprobe robots <url>` prints which AI crawlers (GPTBot, OAI-SearchBot, ChatGPT-User, ClaudeBot, Claude-User, PerplexityBot, Google-Extended, CCBot, Applebot-Extended) robots.txt allows or blocks. It uses its own matcher: longest match wins, `Allow` wins ties, `*` and `$` work.
- `rankprobe plan <url>` runs the audit and sends **only the findings JSON** plus a short text excerpt, never the full HTML, to a model. You get back a prioritized fix plan with snippets you can paste.
- `--max-pages N` crawls same-origin links and flags duplicate titles and descriptions. `--fail-under N` exits 1 for CI. `--sarif` writes SARIF 2.1.0 for GitHub code scanning. `--json` is there for scripts.

```bash
pip install git+https://github.com/bayuh85/rankprobe
rankprobe audit https://your-site.dev
```

## Real use case: auditing a cross-posted blog

Here is the crawl I ran on the blog, 15 pages:

```bash
rankprobe audit https://bayuh85.github.io/atmorouter-blog/ --max-pages 15
```

Grouped by check id, the findings were:

| Severity | Check | Pages |
| --- | --- | --- |
| warn | `meta-description` (over 160 chars) | 15 |
| warn | `title-length` (over 65 chars) | 15 |
| warn | `og-image` missing | 15 |
| warn | `canonical-target` (canonical on another origin) | 9 |
| warn | `duplicate-description` | 1 |
| warn | `robots-txt`, `sitemap`, `llms-txt` missing at origin | 1 each |
| info | `jsonld-types`: BlogPosting, Person, WebPage, FAQPage | 15 |

Each finding shows the actual value, so you don't have to go and look:

```text
On-page
  warn   meta-description        Meta description is 197 characters; expected 50-160.
         url: https://bayuh85.github.io/atmorouter-blog/posts/applyfit/
         evidence: {"description": "A Python CLI and MCP server that keeps your profile and every job posting as plain files, scores fit offline from 0 to 100, ...", "length": 197}
```

Two of these deserve a comment, because a good audit tool should tell you things you then decide to ignore:

- **`canonical-target`** fires on every cross-posted article, because the canonical points at dev.to. That's intentional here. The dev.to copy is the original. For a normal site, though, a canonical on another host is one of the fastest ways to get a page dropped from the index, so I want it loud.
- **`robots-txt` / `sitemap` / `llms-txt`** are checked at the origin (`bayuh85.github.io/robots.txt`), because that's the only place crawlers look. A project site under a subpath can't fix that without a user-site repo. Good to know before you blame the generator.

The real fixes are all in one Jekyll layout: shorten the `<title>` template, cap the description, add a default `og:image`. One PR, 15 pages fixed.

For comparison, the same command on `example.com` returns `score: 49/100` with no canonical, no `h1`, no description and no JSON-LD. Running `rankprobe robots https://www.nytimes.com` shows all nine AI crawlers blocked, which is a deliberate choice on their side. You probably want to know whether your site makes the same choice by accident.

## The fix plan, for about 1,400 tokens

Then I asked for a plan on one post:

```bash
rankprobe plan https://bayuh85.github.io/atmorouter-blog/posts/vetskill/
```

```text
sent 5774 chars (~1444 tokens, estimated as chars/4) to atmo/deepseek-v4.1-flash at https://atmorouter.dev/v1

# Fix plan — https://bayuh85.github.io/atmorouter-blog/posts/vetskill/

## 1. robots-txt — create /robots.txt (also fixes sitemap discovery and locks in AI crawler access)
...
## 3. canonical-target — point the canonical at the URL you want indexed
... If the dev.to post should stay canonical (you're treating GitHub Pages as a mirror), leave it ...

## 4. title-length — shorten the title to 15–65 chars
Current title is 170 characters ...
<title>vetskill: scan a Claude Code skill before install</title>
```

The prompt tells the model to work only from the findings it was given, and the first line tells you exactly how much was sent. 5.7 KB of JSON instead of the 26 KB HTML page. The plan took about 10 seconds.

## In CI

The repo ships [`examples/github-actions-audit.yml`](https://github.com/bayuh85/rankprobe/blob/main/examples/github-actions-audit.yml). The important line is:

```yaml
- run: rankprobe audit "$TARGET" --fail-under 80 --sarif rankprobe.sarif
```

Point `TARGET` at your preview deploy URL, or at a built HTML file in the repo. A PR that adds a `noindex` or breaks the JSON-LD now fails the check, and the SARIF upload puts the findings in the code scanning tab. The audit only fetches your page plus robots.txt, the sitemap and llms.txt from its origin, and never calls a model.

## The model provider

`plan` is one short chat call, so it should cost close to nothing. rankprobe defaults to [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative: 181 models behind one OpenAI-compatible and Anthropic-compatible key, pay per token, averaging 79.8% below official prices when I checked today. The default model, `atmo/deepseek-v4.1-flash`, is free there, and that's the model in the run above.

It isn't tied to it. Any OpenAI-compatible endpoint works:

```bash
RANKPROBE_BASE_URL=http://127.0.0.1:11434/v1 RANKPROBE_MODEL=llama3.2 OPENAI_API_KEY=local \
  rankprobe plan https://your-site.dev
```

| Variable | Default |
| --- | --- |
| `ATMOROUTER_API_KEY` / `OPENAI_API_KEY` | first one set wins |
| `RANKPROBE_BASE_URL` | `https://atmorouter.dev/v1` |
| `RANKPROBE_MODEL` | `atmo/deepseek-v4.1-flash` |

## Limits, honestly

rankprobe reads the HTML the server sends. It doesn't run JavaScript, so a client-rendered SPA will look empty to it, and to a lot of crawlers too. It doesn't measure Core Web Vitals, check backlinks or know anything about rankings. The multi-page score adds penalties across pages, so a 15-page crawl bottoms out fast. Read the per-check counts, not the number. It's the pre-merge lint for SEO, not the whole SEO job.

## Try it

- Code: [github.com/bayuh85/rankprobe](https://github.com/bayuh85/rankprobe)
- Default gateway for `plan`: [atmorouter.dev](https://atmorouter.dev)

Checks live in `checks.py`, each with an id, a severity and a message, so adding one is a small PR. If it catches a stray `noindex` before your launch, a star helps other people find it.
{% endraw %}
