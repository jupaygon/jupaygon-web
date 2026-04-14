---
title: 'Context window: when your AI agent starts forgetting'
date: '2026-04-14'
slug: 'context-window'
draft: true
description: 'The context window is the scarcest resource in AI-augmented development. When your policies, protocols, and sessions grow, the agent starts ignoring instructions. Here is how we fixed it.'
tags: ['ai', 'development', 'claude-code', 'productivity']
cover:
  image: 'cover.jpg'
  alt: 'Context window limit in AI-augmented development'
  relative: true
ShowToc: true
TocOpen: false
---

Everything starts great.

You give your AI agent instructions, it follows them to the letter. You define policies, it complies. You pass it work protocols, it executes them. It is fast, precise, and never complains.

Until one day it stops.

No error. No warning. It simply ignores rules it used to follow. It makes stupid mistakes. It repeats questions you already answered. What changed?

The answer is the **context window** — we get into the work and stop paying enough attention to it.

## What the context window is and why it matters

The context window is the amount of text an AI model can "keep in mind" at any given moment. It is its working memory. Everything you inject — system instructions, policies, code files, conversation history — competes for the same space.

The model I have been using lately has a context window of one million tokens. That sounds enormous. But when you work seriously with an AI agent, you discover that a million tokens fills up faster than you think.

Why? Because it is not just about what you write. In a real development session, the agent:

- Reads complete code files to understand context.
- Runs commands and receives their output.
- Reads documentation, tests, configurations.
- Accumulates the entire conversation history.

Each of these actions consumes tokens. And when it approaches the limit, the system starts compressing earlier messages. In other words: **the agent starts forgetting**.

## What we inject before the agent says a single word

In my setup, the agent starts every session with a set of files injected into the system prompt — the fundamental instructions it must always follow:

| File                    | Purpose                                                                              |
|-------------------------|--------------------------------------------------------------------------------------|
| **IDENTITY.md**         | Who the agent is: name, email, GitHub user                                           |
| **CONTACTS.md**         | Its address book: the Owner, other agents, relevant people                           |
| **MAIN_RULES.md**       | Hard rules: never push to master, always use worktrees, execution states             |
| **BASE_POLICY.md**      | Operational policy: verifiable truth, security, autonomy, credentials                |
| **DEVELOPER_POLICY.md** | Role policy: code quality, task tracking, permissions                                |

On top of that, there are protocols loaded on demand: Git development workflow, Jira workflow, CSS rules, visual verification...

All of this exists because without explicit instructions, the agent makes its own decisions — and they are not always good ones. Policies are what turn a generic AI into a reliable collaborator.

The problem is that every line of policy consumes tokens from the context window. And those tokens compete with the code the agent needs to read, the tools it needs to use, and the conversation it needs to remember.

## When the policies stopped working

A few days ago I ran into a serious problem: the agent had stopped following fundamental rules. Rules that were written in its policy. Rules it used to follow.

The cause: the policy files had grown too large. What started as concise documents had gradually inflated with explanations, examples, historical context sections, and duplications across files. The system prompt had become a wall of text that, combined with a long work session, pushed important content outside the model's attention zone.

The agent did not "decide" to ignore the rules. It simply could not see them — they were buried under so much text that the model deprioritized them.

**The most treacherous symptom: there is no error.** The agent keeps responding, keeps working, keeps appearing competent. But its decisions no longer respect the constraints you defined. And you do not notice until you start getting desperate with the errors.

## The great reduction

The solution was drastic but necessary: rewrite all policies from scratch with a clear criterion — **every token must earn its place in the context window**.

The result for the files injected in every session:

| File                | Before           | After          | Reduction |
|---------------------|------------------|----------------|-----------|
| IDENTITY.md         | 18 lines         | 6 lines        | **67%**   |
| CONTACTS.md         | ~27 lines        | 19 lines       | **30%**   |
| MAIN_RULES.md       | —                | 41 lines       | new       |
| BASE_POLICY.md      | 353 lines        | 50 lines       | **86%**   |
| DEVELOPER_POLICY.md | 391 lines        | 47 lines       | **88%**   |
| **Total**           | **~789 lines**   | **163 lines**  | **79%**   |

On top of this, the protocols loaded on demand during work sessions (Jira, Symfony, visual verification, etc.) went from 651 lines to 302.

In total, the injectable context went from ~1,440 lines to ~465. Mind you, I later had to make adjustments, because in some areas I had cut too much.

### Concrete example: IDENTITY.md

Before:

```markdown
## Agent

- Display name: Jarvis
- Role: developer
- Execution environment: Claude Code on macOS
- Timezone: Europe/Madrid
- Operational email: jarvis@example.com
- GitHub user: jarvis-aidev
- Code language: English

## Jira Instances

- zeronet
  - URL: xxxxxxxxxxxxxxxxx
  - Key: ZN
  - Credentials: credentials/jira-zeronet.conf
```

After:

```markdown
# Identity

- Display name: Jarvis
- Role: developer
- Email: jarvis@example.com
- GitHub user: jarvis-aidev
- Code language: English
```

The execution environment, the timezone, the Jira instances — none of that needed to be in the permanent prompt. The agent already knows it runs on macOS. The timezone does not affect its decisions. And Jira credentials are resolved through a different mechanism.

### Concrete example: BASE_POLICY.md

Before — the policy change control section:

```markdown
## HARD RULE — Policy file change control

- No automated agent may edit policy files directly on `master`.
- Exception: direct updates to `master` are allowed only when explicitly instructed by the Owner.

Default rule for any policy change (unless the exception above is explicitly invoked):

- Any change to a policy must be made only:
  1) by the Owner manually, OR
  2) via a dedicated branch + Pull Request explicitly requested by the Owner.
- Agents must never push policy changes to `master` (no direct commits, no force-push) unless the Owner explicitly invokes the exception above.
- If the Owner requests a policy change via PR, the agent must:
  - propose the diff,
  - create a PR,
  - and wait for the Owner to review/merge.
- If in doubt, do not modify. Ask.
```

After — the same rule in MAIN_RULES.md:

```markdown
## Hard Rules

1. ALL code changes → branch + worktree. No exceptions.
2. Never push master/main.
3. Never merge PRs.
```

Three lines. The same protection. Zero ambiguity.

## The telegraphic style: less prose, more instruction

The key to the reduction was not removing rules, but changing the format. We moved from **explanatory prose** to **telegraphic style**: short sentences, no unnecessary articles, no long justifications.

A policy is not a document for humans. It is an instruction for a machine. It does not need to convince; it needs to be clear and brief.

This is counterintuitive. When you write policies for an AI agent, the natural instinct is to be thorough — explain the why, give examples, cover every edge case. But the more text you add, the more you dilute important instructions in noise.

There is a variant of this that is becoming trendy, called the "caveman" style — yes, seriously — which apparently also works very well, with a tremendous token reduction. I have not had the time or the inclination to try it yet, but I am sure I will.

**Less text, more compliance.**

## Long sessions: the other enemy

Optimized policies solve half the problem. The other half is long sessions.

In an intense work session, the agent can read dozens of files, run commands, receive long outputs, and accumulate hundreds of message exchanges. All of that stacks up in the context window. And when it approaches the limit, the system automatically compresses the oldest messages.

That compression is necessary, but it comes at a cost: the agent loses nuance. Decisions you made together at the beginning of the session, agreements on how to approach a problem, task-specific context — all of that fades.

The symptoms:

- **Repeats questions** you already answered.
- **Re-reads files** it already read.
- **Contradicts decisions** made earlier in the same session.
- **Loses its tone** — becomes more generic, less precise.

The solution is not technical — it is operational: **learn to end sessions at the right time.** A fresh session with clean context performs better than a long session where the agent drags compressed tokens from an hour ago. Ask it to write a summary with the essentials needed to continue the unfinished work in a future session, and to leave it somewhere it can retrieve it.

## Lessons learned

After weeks of optimizing my agent's context window, these are the lessons I take with me:

**1. The context window is a budget, not unlimited space.**
Every token you spend on policies is one less token for code, tools, and conversation. Manage that budget with the same discipline that maaaany years ago was used to manage RAM when programming.

**2. If the agent does not follow a rule, the problem might not be the agent.**
Before blaming the model, check how much text you are injecting. Sometimes the solution is not repeating the instruction — it is removing everything else.

**3. Write policies like code, not like documentation.**
Concise, unambiguous, no unnecessary prose. If a rule needs a paragraph of explanation, it is probably not a good rule.

**4. Sessions have a shelf life.**
Do not try to do everything in one session. Cut when you notice degradation. A new session with clear instructions is more productive than an old session with compressed context.

**5. Measure before you optimize.**
Ask the agent itself to measure the token cost of all injected files and list the biggest consumers. It can do this on its own in seconds.

## The context window as a design constraint

Working with an AI agent is not just about writing good prompts. It is about designing a system where the right information reaches the model at the right time, without saturating the available space.

The context window is not a technical detail you can ignore. It is the fundamental constraint that determines whether your agent is a reliable collaborator or a machine that generates generic responses.

The sooner you optimize it, the sooner you will stop wondering why your agent "stopped working." It did not stop working. It lost relevant information that you are not giving it (without realizing).
