---
layout: post
title: "StudyLoop: turn your markdown notes into flashcards that can't lie to you"
date: 2026-10-31T05:00:00Z
description: "An open-source terminal tutor: an LLM drafts flashcards from your notes, unsupported cards are rejected, review with SM-2, ask offline with BM25 line citations. Runs on AtmoRouter."
canonical_url: https://dev.to/bayuh85/studyloop-turn-your-markdown-notes-into-flashcards-that-cant-lie-to-you-280j
tags: [python, opensource, productivity, ai]
---
*Originally published on [dev.to](https://dev.to/bayuh85/studyloop-turn-your-markdown-notes-into-flashcards-that-cant-lie-to-you-280j).*

This post walks through **StudyLoop**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

I keep my notes in markdown. Hundreds of files: RFC summaries, postmortems, the bits of HTTP caching I have to re-learn every time a CDN misbehaves. Writing them is easy. Remembering them is not. Two months later I'm searching my own notes and still getting `no-cache` and `no-store` mixed up.

Flashcards fix the remembering part, but making them by hand takes so long that I never kept it up. Asking an LLM to make them is fast, but it's risky. Sometimes the model invents an answer that sounds right and isn't in the notes, and then you memorize something false.

So I built **StudyLoop**, a small terminal tutor that treats your notes as the only source of truth. It is open source (MIT), pure Python stdlib, and has no runtime dependencies: [github.com/bayuh85/studyloop](https://github.com/bayuh85/studyloop).

## The real use case: learning HTTP caching from my own notes

The repo ships `examples/http-caching/notes.md`, a short set of notes on freshness, `Cache-Control`, validators, conditional requests and `Vary`. Here is the loop I actually run:

```bash
export ATMOROUTER_API_KEY=...        # only card generation needs a key
python3 -m studyloop add examples/http-caching/notes.md --deck caching
python3 -m studyloop study --deck caching --limit 10
python3 -m studyloop ask "what does no-cache mean" --deck caching
```

On a live run this morning, `add` cut the file into 14 passages and the model proposed 32 cards. StudyLoop **kept 30 and rejected 2**:

```text
passages 14  new 14 · moved 0 · unchanged 0 · removed 0
cards 30 accepted · 2 rejected
  rejected  quote not in passage  484e52b3fc7bd77d  What does the `immutable` Cache-Control extension actually do...
  rejected  quote not in passage  484e52b3fc7bd77d  How does `stale-while-revalidate` work...
```

Those two rejections show the whole idea working. The model wrote reasonable questions about `immutable` and `stale-while-revalidate`, but the passage it was given didn't support its answers. So those cards never got into my deck.

## How a card is kept or rejected

Each passage is sent on its own, and the model has to return JSON: `front`, `back`, and a `quote`. A card is stored only if:

- the `quote` is a contiguous excerpt of that passage (whitespace may differ, words and order may not),
- at least half of the content words in `back` appear in the passage, so a real quote attached to an invented answer still fails,
- any citation the model names matches the passage's own `[file:start-end]`.

Every card carries its line range, so when you review it you can jump straight to the source.

## Passages that survive edits

Each passage id is a SHA-256 over the source path plus the exact passage text. If you insert a line near the top of the file, the line ranges shift but the ids stay the same, so your review history stays put. If you edit a passage, it becomes new work. If you delete one, its cards go too. Running `add` again only sends passages the model hasn't seen yet, so re-syncing a notes folder costs almost nothing.

## Review with SM-2, ask with BM25

`study` is classic SuperMemo SM-2: ease starts at 2.5 and never goes below 1.3, intervals run 1 day, 6 days, then grow by ease, and an *again* grade puts the card back in the current session.

`ask` makes **no network call**. It ranks passages with Okapi BM25 (`k1 = 1.5`, `b = 0.75`; hyphenated directives like `max-age` count as one token) and answers in your notes' own words:

```text
`no-cache` does not mean "do not store this". A cache may store a no-cache
response, but it must revalidate with the origin before every reuse.
[examples/http-caching/notes.md:37-43]
```

The answer is never made up, because it's your own sentences with a line citation.

## Try it without a key

There's an offline demo server that stands in for the model, and the 46-test suite runs against it:

```bash
python3 -m studyloop.demo_server --port 8766
python3 -m studyloop add examples/http-caching/notes.md --deck caching \
  --base-url http://127.0.0.1:8766/v1 --api-key demo
```

## Provider

Card generation uses the standard OpenAI chat-completions request, so `--base-url` works with any compatible server. By default it points at [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative: 155 models behind one OpenAI-compatible key, pay per token, at up to 89% below official prices. The default model is `cc/claude-sonnet-4-6`. Turning a whole notes folder into cards costs about as much as a coffee, and asking questions is free because it runs locally.

Repo: [github.com/bayuh85/studyloop](https://github.com/bayuh85/studyloop). Issues and PRs welcome, especially chunking for other heading styles.
