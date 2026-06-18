---
title: 'Loop engineering: designing the system that prompts the agent'
date: '2026-06-17'
slug: 'loop-engineering'
draft: false
description: "Addy Osmani and Boris Cherny gave it a name: you stop prompting the agent and start designing the loop that prompts it. Here are their six pieces, which I've spent months trying out in my own setup, plus some critical small print."
tags: ['ai', 'development', 'claude-code', 'workflow', 'agents']
cover:
  image: 'cover.jpg'
  alt: 'A self-running loop discovering, executing and verifying work around a human checkpoint'
  relative: true
ShowToc: true
TocOpen: false
---

In June 2026 Addy Osmani and Boris Cherny put a name on something that isn't new (by the current pace of "new", at least): **loop engineering**. Osmani is an engineering leader at Google Chrome and a prolific writer on web development; Cherny is one of the creators of Claude Code. They laid it out in [a thread on X](https://x.com/addyosmani/status/2064127981161959567) and a couple of essays.

The one-liner that captures it is Cherny's: "I don't prompt Claude anymore. I have loops running that prompt Claude." The shift is small to say and large to live with. You stop being the person who holds the tool through every turn, and you start designing the system that finds the work, hands it out, checks it, writes down what's done, and decides the next thing.

They did that hard part: seeing the shape clearly enough to name it. I've spent months trying those six pieces in my own setup. So this post is two things at once: a map of the pieces they lay out, each one wired to my real setup, and an honest note about the bill.

## One floor above the harness

I [wrote before](/en/posts/2026-05-13/determinism-in-ai-workflows/) about pulling decisions out of the language model and crystallising them into hooks, scripts and routing tables. That's the harness: the deterministic perimeter a single agent runs inside.

Loop engineering sits one floor above that. The harness answers "how does one agent behave safely". The loop answers "who starts the agent, how many run at once, what happens when one finishes, and how the next piece of work gets chosen". The harness is the safety rails. The loop is the thing driving on the road.

Osmani breaks the loop into six pieces. Let's take them one by one.

## 1. Automations: the heartbeat

A loop needs something that fires without you. Scheduled jobs and watchers that discover work and surface it instead of waiting for you to type.

In my case that's two layers. Hooks react to events (a PR opened kicks off a watcher that polls until CI is green or a review lands). Scheduled runs go looking for work on a cadence: pick the next unchecked row from the roadmap, open the tracked task, start the branch. The point is that the first prompt of a session isn't always mine. Often the system wrote it.

This is the piece that feels like magic the first week and like a liability the third, for reasons I'll get to.

## 2. Worktrees: parallel without collisions

The moment more than one agent is alive, they fight over files. Git worktrees solve this cleanly: each agent gets its own checkout on its own branch, sharing history, never stepping on each other's working tree.

It has to be a hard rule in the setup: nothing gets touched on `master`, every task starts with a branch and a worktree, and the bootstrap that prepares it is [a fixed recipe](/en/posts/2026-05-13/determinism-in-ai-workflows/) run by a cheap subagent. Parallelism stops being scary once isolation is the default instead of an afterthought.

## 3. Skills: intent that survives the reset

A skill is a folder with a `SKILL.md` that captures a convention, a build step, a piece of domain knowledge the agent would otherwise re-derive (badly) every cycle. Osmani calls the thing they prevent "intent debt": the cost of the model rediscovering, again and again, what you already decided.

I keep a versioned set of them. One pins down the API surface of an admin framework that broke half its methods between major versions, so the agent stops guessing. Another encodes a technical-SEO checklist. They live in the agent's repo, reviewed like code, because [that's what they are](/en/posts/2026-03-29/ai-skills-easyadmin5/): executable intent, not documentation that rots in a wiki.

## 4. Connectors: the agent acts, it doesn't suggest

Without connectors a loop is a very articulate intern who can only talk. With them it reads a PR, comments on it, transitions a ticket, resolves an alert, opens an issue. MCP servers are the wiring: GitHub, the issue tracker, the error monitor, the project board.

This is the difference between "here's what I would do" and "done, here's the link". It's also where a [routing table](/en/posts/2026-05-13/determinism-in-ai-workflows/) adds the most value: the loop must reach a private PR through the GitHub connector, never through `curl`, or it loses identity, logging and auth handling in one shortcut.

## 5. Sub-agents: separation of concerns

One agent explores, another implements, another verifies. They run with different instructions and, often, different model tiers. The cheap model runs the deterministic recipes; the expensive one is saved for design and for the review before a push.

The sharpest reason to split them is Osmani's, and it's worth quoting straight: "the model that wrote the code is way too nice grading its own homework." A fresh agent with no stake in the previous answer is a far better critic than the one defending it. So the auditor isn't the author. The verifier reads the diff cold.

## 6. External state: memory on disk

The model forgets everything between runs. So the memory can't live in the context window: it has to live on disk. Markdown files, a task board, a tracker. The loop reads its state at the start of a run and writes it back at the end.

This is the quiet backbone. The roadmap that says what's done and what's next, the tracked task that survives a crash, the auto-memory that remembers a quirk I explained three weeks ago. I [care a lot about what goes into the context window](/en/posts/2026-04-14/context-window/); the corollary is caring just as much about what lives safely outside it.

## The bill

Here's the part that doesn't fit on a launch thread.

A loop is an amplifier, and amplifiers don't care about the sign of the signal. Point it at good practice and it compounds good practice. Point it at a shaky assumption and it compounds the mistake, in parallel, while you sleep. The same machinery that ships ten correct PRs overnight will ship ten wrong ones with the same confidence and the same green checkmarks.

The subtler cost is what Osmani calls comprehension debt. A loop will happily let you move fast on work you understand deeply, and it will just as happily let you move fast on work you've stopped understanding at all. From the outside the two look identical: tickets close, the board goes green. The difference only shows up the day something breaks and you realise you can't reason about a system your loops built without you.

So the human doesn't leave the loop. The human moves to the two ends of it. At the start, deciding what cannot move: the architecture, the perimeter, which problem is even worth a loop. At the end, the verification: not "did CI pass" but "do I understand what changed and am I willing to put my name on it". That second checkpoint is the actual bottleneck of the whole thing, and the one piece you must not automate away, because the moment you do, the loop is no longer making you faster. It's making you a spectator of your own codebase.

## Closing

Loop engineering is real, and it's a genuine step up from prompting turn by turn. I've had it running for months and I wouldn't go back.

But the leverage cuts both ways. A well-designed loop multiplies a careful programmer. It also multiplies a careless one, faster. The design work, the part that's actually engineering, isn't wiring up the six pieces. It's deciding what the loop is allowed to do on its own, and keeping the one checkpoint that's still yours: shipping code you confirmed works, and understand.
