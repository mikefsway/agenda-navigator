# Agenda Navigator

Paste what you've worked on and a sentence about what you want from the week,
get a personal timetable out of a conference programme: one pick per slot with
the evidence for it, alternatives underneath, and genuine clashes shown as a
fork rather than silently resolved.

**Live example:** [RGS-IBG 2026](https://mikefsway.github.io/rgs-agenda/) —
593 sessions, 2,217 papers, four days.
**Build one for your conference:** [paste your programme
URL](https://mikefsway.github.io/rgs-agenda/build.html) and it writes the
instructions.

This repository is the **kit**: the engine, the pipeline and the porting guide,
with no conference data in it. `docs/data/` is empty and the page says so until
you fill it. The RGS-IBG deployment above is the same code with a programme
loaded, and is where to look to see it working.

## Everything runs in the browser

The programme ships as precomputed embeddings; the user's text is embedded
locally with [transformers.js](https://huggingface.co/docs/transformers.js)
(bge-small, ~30 MB, cached after first visit) and never leaves the device. No
account, no tracking, no server, no build step. Two off-origin requests remain
and neither carries the user's text: the library from jsDelivr and the model
from Hugging Face, both on first visit only. Fonts are served from the repo
rather than from Google, and a `Content-Security-Policy` meta tag caps what a
compromised CDN could do with a page that is holding someone's whole
publication list.

That privacy model is the reason the tool is worth pasting a real publication
list into, and it is the first thing a port can break without noticing. See
`PORTING.md` §7.

## Getting one running

```
git clone https://github.com/mikefsway/agenda-navigator.git my-conference
cd my-conference
```

Then read [`PORTING.md`](PORTING.md). It is the instruction set, written for a
coding agent and readable by a person; `.claude/skills/port-navigator/` makes
it a skill in the clone, so `/port-navigator` in Claude Code picks it up.

The short version is four steps, of which only the first is new code:

1. **Write the adapter** — replace `pipeline/fetch.py` and
   `pipeline/normalize.py`. Everything downstream depends on
   `docs/data/sessions.json` and nothing else, so that file is the whole
   integration surface. The Ex Ordo versions in this repo are a worked example.
2. **Run `pipeline/embed.py` unchanged** to build the matrix.
3. **Swap about a dozen constants** in `docs/` — §4 lists them one by one.
4. **Re-check the filters** against the new programme and assert the result in
   `test/data.test.mjs`.

Start with §0, which is about whether the conference is worth doing this to.
Two of its gates matter more than they look: a programme of bare titles gives a
route that reads convincingly and is close to random, and a conference with two
parallel sessions per slot doesn't need a tool to choose between them.

## What's here

```
pipeline/            build-time Python, run by hand, output committed
  fetch.py           the Ex Ordo adapter — replace this
  normalize.py       raw API JSON -> docs/data/sessions.json — replace this
  embed.py           sessions -> facet embeddings (bge-small, float16)
docs/                the whole site (GitHub Pages serves this directory)
  index.html/app.js/style.css
  build.html/build.js    "build one for your conference" — writes a porting prompt
  scholar.js             deterministic cleanup for pasted publication lists
  sw.js                  service worker: shell + data cached for offline use
  fonts/                 IBM Plex, self-hosted — no Google Fonts call
  data/                  empty: the conference goes here, see its README
test/                no deps, no runner — plain node scripts
  parse.test.mjs       the Scholar parser against a real 68-article profile
  data.test.mjs        the shipped data files against each other and the app
  monitor.mjs          a deployed site and what it depends on
tools/copyedit.mjs   edit the site's copy over an SSH tunnel without opening the HTML
PORTING.md           the instruction set
CLAUDE.md            why the code is shaped the way it is — read before scoring changes
```

## How the matching works

- **Facet model.** Each session is embedded as several rows: title+description
  chunks, plus one row per paper title. Session score is
  `0.75 * best_facet + 0.25 * mean(top 3)`, so one strongly matching paper can
  surface its session while depth still beats a lone bullseye. The matched
  facets are shown as evidence — *"Matches paper X — from your '…'"*.
- **Two boxes, two pools.** Past work and current intent are scored as separate
  pools, each converted to ranks before blending, and combined
  non-compensatorily. Which sounds like fussiness and isn't: every one of those
  three words is a bug that shipped here first. A single pool lets 40 paper
  titles outvote one sentence of intent; blending raw cosines gives the bigger
  pool a free head start on every facet; and an arithmetic mean lets a session
  the works box barely reaches ride a strong aims rank into the agenda.
- **Clash rule.** If the gap between the top two in a slot is in the closest
  fifth of the slots you're actually deciding, both are shown and neither is
  chosen.
- **A brief for the user's own LLM.** The route, their profile and the top 14 of
  every slot, copied as markdown, so they can hand it to Claude or ChatGPT and
  get the judgement the embeddings can't do — what they've had enough of, who
  they're avoiding, what's already in their diary. It's a second opinion on the
  route rather than a second route. This is the one thing in the tool that sends
  the user's text off their device, by their choice, and the caveat sits next to
  the button. If you keep the feature, keep the caveat: `PORTING.md` §7.

## The part worth reading before you change anything

`CLAUDE.md` is an account of the failures this code already had. Nearly all of
them had **no symptom** — a plausible agenda quoting the wrong papers, a real
workshop silently absent from every route, one generic session winning most
slots because the embedding backend was returning noise. The tests exist
because prose in a working-notes file does not fire.

Three rules catch most of it: no absolute thresholds over cosine similarities
(every threshold in the scoring path is a percentile), nothing but immutable
identity in `normalize.py`'s sort key (`facets.json` addresses sessions by row
index, so a display field in the sort key silently rescores the whole
programme), and `embedderSelfCheck` must throw rather than pass when it has
nothing to check.

## Licence and credit

MIT — see [`LICENSE`](LICENSE). Do what you like with it; the copyright notice
stays in the source.

Beyond that, one request rather than a condition: **keep a visible credit line
in the footer of anything you deploy.**

```html
<p>Built with <a href="https://github.com/mikefsway/agenda-navigator">Agenda
Navigator</a> by Mike Fell. Profile format: <a href="https://fraglet.org">fraglet</a>.</p>
```

Stripping it is within your rights. Improvements sent back as a pull request are
worth more than the line.

And please keep the "not affiliated" and "check the official programme" notes:
this builds on someone else's programme, and saying so plainly is what makes
that fine.
