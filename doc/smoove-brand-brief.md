# smoove — brand brief

A short north star for the rebrand: how smoove talks, what it stands for, and
how it should look. Pair with [`rename-to-smoove-plan.md`](./rename-to-smoove-plan.md)
(the mechanical rename) — this is the *why* and the *feel*.

---

## Essence

**Smooth moves, in code.** smoove is a timeline-driven animation engine for the
canvas — motion that's buttery in preview and light on the server, built on
concepts humans *and* LLMs already understand.

> The name says it: every animation should feel **smoove**.

## What it is (one line)

> A timeline-driven animation engine for Konva — keyframe motion that runs
> anywhere, renders fast, and is built on familiar concepts an LLM can reason
> about to author videos.

## Audience

**Technical creators — developers first.** Two concentric circles:

1. **Core:** JS/TS developers adding real animation to canvas apps without
   adopting a heavy video framework.
2. **Edge (the ambition):** *anyone technical* — because the real unlock is
   handing skills to an LLM and letting it generate the videos. smoove is the
   engine; the human describes the motion, the model writes the timeline.

This works because the API is built on **concepts an LLM already knows** —
keyframes, timelines, flexbox layout, familiar shape primitives — so a model
that has never seen smoove can still reason from the API and produce correct
output. We close the gap with **documented best practices and skills** that
teach the model how to compose them into real videos. Clarity for the LLM is
clarity for the human.

## Personality

**Playful & smooth.** Confident but never stiff. It leans into the pun without
being a gimmick — the wink is in the name, the substance is in the engine. Think
*effortless*: the hard machinery (layout, timing, SSR) is hidden so the result
just glides.

Three words: **effortless · playful · sharp.**

## Positioning

smoove stands on its own — a canvas animation engine, judged on its own merits.
**Do not position it against, or even mention, other frameworks.** Lead with what
makes it good:

- **Smoother preview** — higher-performance playback in the browser.
- **Smaller footprint** — lighter server-side requirements, fewer dependencies.
- **Faster renders** — lean, potentially faster SSR/offline rendering.
- **No React, no WASM** — platform-agnostic; runs in the browser, in Node, and
  headless. Import the whole drawing vocabulary, animate it on a frame clock.
- **LLM-authorable** — built on familiar concepts (keyframes, timelines,
  flexbox, standard shapes) plus documented skills, so a model can reason from
  the API and generate videos.

## Messaging pillars

| Pillar | The promise | Proof |
| --- | --- | --- |
| **Smooth** | Animation that feels good to watch and to write | High-perf preview, frame-accurate timeline |
| **Light** | Less to install, less to run | Small footprint, low server requirements, no React/WASM |
| **Anywhere** | One engine, every runtime | Browser · Node · headless SSR, platform-agnostic |
| **Authorable** | Familiar to humans and models alike | API built on known concepts + documented skills, so an LLM can reason from it and generate videos |

## Voice

**Do**
- Speak plainly and a little playfully — short sentences, active verbs.
- Let the motion metaphor carry: *glide, flow, smooth, move, frame, timeline.*
- Show, don't boast — a tight code sample beats an adjective.
- Write docs an LLM can follow literally: unambiguous, example-led.

**Don't**
- Compare to or name other libraries/frameworks.
- Overclaim with hard numbers we haven't measured (say "lighter / faster,"
  not "3× faster," until benchmarked).
- Get cute at the cost of clarity — the joke is the name, not the API docs.
- Touch the underlying Konva vocabulary in brand copy; smoove is the layer
  *on top*, never a replacement claim for Konva itself.

## Visual direction

For the wordmark, color, and asset look (feeds the logo + existing
`smoove-{mark,icon-*}.svg` assets):

- **Motion-as-identity.** The strongest motif is *afterimage / motion trail* —
  the double **o** in smo**o**ve is a gift: ghosted, offset, or blurred to read
  as movement frozen mid-glide. The existing `icon-afterimages` mark already
  points here; make it the signature.
- **Wordmark:** lowercase, rounded, friendly geometric sans. Lowercase keeps it
  casual and smooth; rounded terminals echo eased motion (no hard stops).
- **Color:** keep the **gradient** mark as the hero — a flowing two-tone gradient
  reads as motion and energy, on brand with "smooth." Pair with a clean
  black/white wordmark for type-heavy contexts (`mark-black` / `mark-white`).
- **Feel:** generous spacing, eased curves, nothing rigid. If a shape could be a
  keyframe in a smooth animation, it belongs.

## Taglines (pick / iterate)

- **Smooth moves, in code.**
- **Motion that glides.**
- **Animate anything. Anywhere. Smoove.**
- **Timelines for the canvas.**
- **Describe the motion — smoove renders it.** *(LLM-forward)*

---

*Guiding rule (carried from the rename plan): rebrand the project tokens only.
The underlying **Konva** library is untouched and uncriticized — smoove is the
animation layer that rides on top of it.*
