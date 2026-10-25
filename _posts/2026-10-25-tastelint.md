---
layout: post
title: "tastelint: a linter that fails the build when your coding agent ships the purple-gradient landing page again"
date: 2026-10-25T02:00:00Z
description: "An open-source, offline taste linter for AI-generated UI. 17 deterministic rules for HTML, CSS, JSX/TSX, Vue, Svelte and Tailwind, a 0-100 taste score, a CI gate, and an optional review step through a cheap AI API gateway."
canonical_url: https://dev.to/bayuh85/tastelint-a-linter-that-fails-the-build-when-your-coding-agent-ships-the-purple-gradient-landing-3p5c
tags: [ai, webdev, opensource, css]
---
*Originally published on [dev.to](https://dev.to/bayuh85/tastelint-a-linter-that-fails-the-build-when-your-coding-agent-ships-the-purple-gradient-landing-3p5c).*

This post walks through **tastelint**, an open-source tool that runs by default on [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative.

{% raw %}

**Repo:** 👉 [github.com/bayuh85/tastelint](https://github.com/bayuh85/tastelint) (MIT, Python 3.10+, standard-library scanner, 37 offline tests)

## The problem

Ask any coding agent for a landing page and you can draw the result before it finishes: a purple-to-blue gradient, Inter, a centered hero, "✨ Supercharge your workflow", a `Get Started` button, and three identical glass cards with an emoji on top and lorem ipsum underneath.

None of that is a bug. Type checks pass, tests pass, the page renders. It just looks like every other generated page, and nobody catches it until a human opens the preview and sighs.

ESLint and Stylelint check correctness. Nothing in the pipeline checks *taste*. So I wrote a linter for the habits themselves: name the pattern, point at `file:line`, give the page a score, and fail CI when the score is too low.

## What tastelint does

- `tastelint scan` walks HTML, CSS, SCSS, JSX/TSX, Vue, Svelte, MDX and Tailwind class strings with a deterministic rule engine. No browser, no network, no model.
- `tastelint review` (optional) sends a compact summary of the findings to an OpenAI-compatible model and asks for concrete edits.
- `tastelint skill install` drops a design guide where Claude Code, Cursor and Codex look for instructions, and tells the agent to run the scanner before it says "done".

There are 17 rules, each with a severity and a one-line fix:

| Rule | Severity | Catches |
| --- | --- | --- |
| `gradient-wash` | high | Cool purple-to-blue gradient |
| `hype-phrase` | high | A sentence that could ship on any other product |
| `nested-card` | high | Card inside a card |
| `gray-on-gray` | high | Neutral text on a nearby neutral |
| `stock-sans` | medium | Inter / Roboto / Poppins / system stack as the actual face |
| `glass-panel` | medium | Translucent blur panels |
| `feature-trio` | medium | Three matching feature cards |
| `violet-default` | medium | Indigo or violet as the unchosen accent |
| `gradient-text` | medium | Gradient clipped into a headline |
| `radius-inflation`, `shadow-bloat`, `center-gravity`, `emoji-heading`, `filler-copy` | medium | The rest of the costume |
| `blur-orb`, `emoji-icon`, `empty-cta` | low | Decorative orbs, emoji icons, "Learn More" |

The score: high findings cost 12, medium 5, low 2, each rule is capped at 12, and the result is 100 minus the sum (floor 0). `--fail-under 90` means one high finding fails the gate.

## A real use case: gating an agent-generated landing page

Say you asked an agent for the homepage of a small shipping tool. This is the kind of component that comes back (a typical example, trimmed to one file):

```tsx
export default function Landing() {
  return (
    <main className="min-h-screen bg-gradient-to-br from-indigo-600 via-purple-600 to-blue-500 font-sans">
      <section className="flex flex-col items-center justify-center text-center py-32">
        <h1 className="text-6xl font-extrabold">✨ Supercharge your shipping workflow</h1>
        <p className="mt-6 text-gray-400">Leverage cutting-edge tools to streamline every parcel.</p>
        <button className="mt-10 rounded-full bg-violet-600 px-8 py-4 shadow-2xl">Get Started</button>
      </section>
      <section className="grid grid-cols-3 gap-8 px-16">
        <div className="rounded-3xl bg-white/10 backdrop-blur-xl p-8 shadow-xl"><h3>📦 Fast</h3><p>Lorem ipsum dolor sit amet.</p></div>
        <div className="rounded-3xl bg-white/10 backdrop-blur-xl p-8 shadow-xl"><h3>🔒 Secure</h3><p>Lorem ipsum dolor sit amet.</p></div>
        <div className="rounded-3xl bg-white/10 backdrop-blur-xl p-8 shadow-xl"><h3>🌍 Global</h3><p>Lorem ipsum dolor sit amet.</p></div>
      </section>
    </main>
  );
}
```

Run the scanner with a gate:

```text
$ tastelint scan src --fail-under 90
tastelint 0.1.0
src  ·  1 file  ·  24 findings (3 high, 20 medium, 1 low)  ·  taste score 9/100

src/Landing.tsx:3  high    gradient-wash     Cool purple-to-blue gradient.
src/Landing.tsx:4  medium  center-gravity    Layout collapsed onto the center.
src/Landing.tsx:5  high    hype-phrase       Sentence that could ship on any other product.
src/Landing.tsx:5  medium  emoji-heading     Emoji sitting in a heading.
src/Landing.tsx:6  high    hype-phrase       Sentence that could ship on any other product.
src/Landing.tsx:7  low     empty-cta         Call to action with no object.
src/Landing.tsx:7  medium  shadow-bloat      Heavy drop shadow.
src/Landing.tsx:7  medium  violet-default    Indigo or violet used as the default accent.
src/Landing.tsx:9  medium  feature-trio      Three matching feature cards.
src/Landing.tsx:10  medium  filler-copy       Placeholder copy still in the file.
src/Landing.tsx:10  medium  glass-panel       Glass panel.
src/Landing.tsx:10  medium  radius-inflation  Corner radius larger than the object.
... (11 more lines)
src/Landing.tsx:12  medium  emoji-heading     Emoji sitting in a heading.

gate: 90  ·  fail
exit 1
```

Score 9/100, exit code 1. Every line names the rule, and `--verbose` adds the fix and the offending source line, for example:

```text
  hint: Say what the control does: 'Reserve a firing slot', not 'Get Started' or 'Learn More'.
  > <a class="cta" href="#">Get Started</a>
```

That is the part an agent can act on. The fix isn't "make it prettier", it's "this button has no object", "this card is inside a card", "this sentence works for a bank and a bakery".

Here is the same page after following the hints: one chosen serif, one pigment, copy that only fits this product, left-aligned, no card grid:

```tsx
<main className="min-h-screen bg-stone-50 text-stone-900 font-['Fraunces',serif]">
  <section className="max-w-3xl px-8 py-24">
    <h1 className="text-5xl font-semibold">Label, weigh and book a courier for 40 parcels before lunch</h1>
    <p className="mt-6 text-stone-700">Paste your order export. Parcelbay prints the labels, picks the cheapest courier per postcode, and emails buyers their tracking links.</p>
    <button className="mt-10 rounded-md bg-emerald-800 px-6 py-3 text-white">Import today's orders</button>
  </section>
  ...
</main>
```

```text
$ tastelint scan src --fail-under 90
tastelint 0.1.0
src  ·  1 file  ·  0 findings (0 high, 0 medium, 0 low)  ·  taste score 100/100

gate: 90  ·  pass
exit 0
```

The linter doesn't make a page good. It makes the default page fail, which forces somebody (you or the agent) to make a decision instead.

## The bundled examples

The repo ships a deliberately generic marketing page in five formats (HTML, CSS, React, Vue, Svelte) and a small ceramics-studio page as the control:

```text
$ tastelint scan examples --quiet
tastelint 0.1.0
examples  ·  7 files  ·  85 findings (25 high, 52 medium, 8 low)  ·  taste score 0/100

$ tastelint scan examples/quiet-studio --fail-under 90 --quiet
tastelint 0.1.0
examples/quiet-studio  ·  2 files  ·  0 findings (0 high, 0 medium, 0 low)  ·  taste score 100/100

gate: 90  ·  pass
```

JSON and SARIF are the same report in another shape, so you can upload SARIF to GitHub code scanning and see findings inline on the PR:

```bash
tastelint scan src --format sarif -o tastelint.sarif
```

Need an exception? `// tastelint-disable-next-line hype-phrase`, or `tastelint-disable-line all`.

## Teaching the agent before it writes the page

Catching it in CI is good. Not generating it is better:

```text
$ tastelint skill install --target claude
installed .claude/skills/tastelint/SKILL.md
$ tastelint skill install --target codex
installed AGENTS.md
```

(`--target cursor` writes `.cursor/rules/tastelint.mdc`.) The installed text is an original guide, the "Plain Surface Charter": name the type, spend one pigment, write a sentence that fails elsewhere, center a line not a page, and run `tastelint scan <path> --fail-under 90` before calling the change done. Re-running the Codex install replaces its section instead of duplicating it.

## Where the model comes in (and why it stays cheap)

The scanner never needs a model. `tastelint review` is the optional second opinion: it scans first, stops if the score is already 100, and otherwise posts the score, rule ids and short snippets (capped at 20 findings) and asks for edits that name a file and a replacement. `--dry-run` shows you exactly what would be sent without opening a socket:

```text
$ tastelint review examples/sloppy-landing --dry-run
dry run  ·  https://atmorouter.dev/v1/chat/completions  ·  model atmo/deepseek-v4.1-flash  ·  taste score 0/100
network was not called

Critique this interface using the Plain Surface Charter. Return concrete edits only.
... (JSON summary of the first 20 findings)
```

A design critique like this is a small prompt you run on every UI change, so price per token matters more than model size. That's why the default host is [AtmoRouter](https://atmorouter.dev), a cheap AI API gateway and OpenRouter alternative: one OpenAI/Anthropic-compatible key for 169 models, pay per token, at an average of 80.0% below official prices. The default model is `atmo/deepseek-v4.1-flash`.

```bash
export ATMOROUTER_API_KEY=ar-your-key-here
tastelint review src
```

It isn't locked in. Point it at any OpenAI-compatible server, including a local one:

```bash
tastelint review src --base-url http://127.0.0.1:11434/v1 --model llama3.2 --api-key local
```

| Variable | Default |
| --- | --- |
| `ATMOROUTER_BASE_URL` | `https://atmorouter.dev/v1` |
| `TASTELINT_MODEL` | `atmo/deepseek-v4.1-flash` |
| `TASTELINT_BASE_URL` / `TASTELINT_API_KEY` | override the AtmoRouter values |

## CI in four lines

```yaml
- run: python -m pip install -e .
- name: Generated pages have to clear the gate
  run: tastelint scan src --fail-under 90
```

Exit codes: `0` pass, `1` under the gate, `2` bad path or arguments, `3` review request failed, `4` review has no key.

## Limits, honestly

tastelint reads source, it doesn't render. It doesn't compute layout or real contrast against a gradient, and a "card" is a class named `card` or a node that combines a large radius with a shadow, border or fill. It will miss things and occasionally flag something you meant. That's why every finding is one rule you can disable per line, and why the score is a gate, not a verdict.

## Try it

- Code: [github.com/bayuh85/tastelint](https://github.com/bayuh85/tastelint)
- Default gateway for `review`: [atmorouter.dev](https://atmorouter.dev)

Rules are small functions in `rules.py`, so adding one (bento grids, anyone?) is a contained PR. If it stops one more purple hero from shipping, a star on GitHub helps other people find it.

{% endraw %}
