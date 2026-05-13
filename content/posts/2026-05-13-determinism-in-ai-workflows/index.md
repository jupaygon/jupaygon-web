---
title: "What's already decided isn't up for debate: determinism in AI workflows"
date: '2026-05-13'
slug: 'determinism-in-ai-workflows'
draft: false
description: 'An AI agent is non-deterministic by design. To make it reliable, you take the decisions that are already made out of the LLM and crystallise them into hooks, scripts and tables. What stays inside the model is bounded execution; the human is who decides.'
tags: ['ai', 'development', 'claude-code', 'workflow']
cover:
  image: 'cover.jpg'
  alt: 'Deterministic guardrails around an AI agent workflow'
  relative: true
ShowToc: true
TocOpen: false
---

An AI agent is non-deterministic by design.

Ask it the same thing twice and you get two different answers. Most of the time that doesn't matter: two equally valid ways of phrasing a commit message, two reasonable ways to explain a bug. Sometimes it's even desirable, because the creative part lives precisely in that margin.

Until you hit decisions that admit no margin.

"Never push to master." "Always create a branch + worktree before touching anything." "Talk to GitHub via the MCP, never `curl`." These decisions are already made (by me, by the team, by some past operational scar). I don't want the agent re-litigating them every session.

This is where the design gets interesting. Because the instinct, when you work with a language model, is to write the rule in plain text and stuff it into the system prompt. "Never push to master." Done. It works 95% of the time.

The remaining 5% is a push to master.

## The prompt trap

As long as the rule lives only as text in a policy, it's a suggestion. A very well-argued suggestion, written in caps, repeated three times, with examples. But in the end, a suggestion the model evaluates against everything else in its context.

One day it's dragging too many tokens, another day the ticket says "urgent" three times, another day there was just a small ambiguity in the initial instruction, and the rule blurs. It doesn't fail dramatically; it fails silently: the agent takes the "fast" decision because, inside its reading of the context, it seemed the most reasonable one.

Policies are good for many things. For absolute protections they're not enough.

## Get the decision out of the LLM

What does work is **moving the decision outside the model's runtime**. Crystallising it into code that doesn't execute inside the LLM, but before or after. Three layers, with concrete examples from my own setup:

### 1. Hooks: hard block

A hook is a shell script the harness runs before (or after) each tool call. If it returns `exit 2`, the call is blocked. The model never even sees the tool succeed: the harness hands back an error.

```bash
# .claude/hooks/block-push-master.sh
if [ "$BRANCH" = "master" ] || [ "$BRANCH" = "main" ]; then
    echo "BLOCKED: direct push to master/main not allowed."
    exit 2
fi
```

It doesn't matter what the model thinks "the best solution" is: the tool doesn't run. The "never master" decision has left the LLM and now lives in `bash`, which is deterministic out of sheer dullness. I already covered how hooks work in [Claude Code internals](/en/posts/2026-04-01/claude-code-internals/), so I won't rehash it; what matters here is **where the rule lives**.

This case has a second lock on top: the agent's GitHub user has no direct push permission on `master` for the protected repos. If the local hook ever failed, the remote rejects. Two-layer defence (both layers in two different places and outside the LLM).

### 2. Scripts: fixed recipe

It's not all about forbidding. Much of an agent's work is **executing a sequence that's already decided**.

When a PR merges, you have to: pull master, delete the worktree, delete the local and remote branches, mark the PR as merged in the tracker, close the task, mark referenced Sentry issues as resolved. Seven steps. Always the same.

That's not LLM work. That's `scripts/post-merge-cleanup.sh` work. The agent invokes the script; the script does the seven steps without re-deciding anything. If instead you let the model "do the cleanup", what you'll get is a slightly different version every time, some of them incomplete.

Same with bootstrapping a worktree: seven steps pinned down in a protocol. There's a subagent whose only job is reading the protocol, running the seven steps, and handing control back. It doesn't think about alternatives. It doesn't need to.

### 3. Routing tables: which tool for what

This is the least flashy layer, and the one that prevents the most bugs.

In the `CLAUDE.md` my agent loads at boot there's a six-row table that says things like:

| Action | Use | Don't use |
|---|---|---|
| Read a private PR | `mcp__github__github_pr_read` | `curl`, `gh`, `WebFetch` |
| Create a Jira issue | `mcp__jira__jira_create` | `curl`, CLI |

That looks like documentation. In reality it's **a decision taken out of the model's runtime**. If you let the agent pick a tool case by case, sooner or later it will reach for `curl` because "it's simpler". And `curl` doesn't propagate the auth error, doesn't log the operation, doesn't respect the agent's identity. The table closes that decision in advance.

## Who decides what

It's worth being explicit about the split, because it's easy to slip.

**The human decides.** Architecture, what to do at an ambiguous trade-off, which problem to tackle first, when to throw an approach away and start over. That doesn't get delegated to a language model. Not out of distrust, but by design: judgement is exactly what an agent must not improvise on my behalf.

**The human crystallises those decisions** into hooks, scripts, tables, policies and protocols. Every time a decision is made, it drops from the prompt to the code.

**The agent executes**, inside a perimeter narrowed by all those layers. It interprets the ticket (with bounded margin), writes the code (within style policies and mandatory tests), drafts the commit (from a template), opens the PR (with a structured body). What remains isn't "creative judgement" by the agent; it's execution inside the gaps the human deliberately left open.

It's human-in-the-loop, with one important nuance: the human **isn't only at the end**, reviewing what the agent produced. The human is **at the start**, deciding what cannot move. The loop opens with the human defining the deterministic perimeter, continues with the agent working inside it, and closes with the human reviewing what came out.

## Bonus: determinism is cheap

There's a nice side effect of moving decisions out of the LLM: the cheap model can do the deterministic work.

The seven worktree bootstrap steps don't need a frontier model. They're seven commands. A subagent with a smaller model runs them just fine because there's nothing to reason about; only the recipe to execute. Same with post-merge cleanup, with a smoke test whose steps are written down, with a UI translation.

The expensive model is freed for what actually needs it: design with the human, debugging with unknown root cause, the self-review before a push.

This only works if the deterministic tasks are actually crystallised. If the "recipe" is in fact "more or less this, you'll see", no model, big or small, will give you the same result twice.

## Closing

An AI agent doesn't become reliable by being given more freedom or better prompts. It becomes reliable by **shrinking the space where it has to decide**.

Every hook you add is a question the model no longer asks. Every script it invokes is a sequence it doesn't reinvent. Every routing table is a tool choice that leaves the moment's reasoning and enters the contract.

What stays inside the LLM is bounded execution. What gets decided is decided by me, beforehand. And it's crystallised as soon as the decision is made, so I don't have to take it again.

That's what separates an agent that "works almost always" from one you can sleep at night with.
