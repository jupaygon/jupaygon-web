---
title: 'AI-Augmented Development: my real experience working with an AI agent'
date: '2026-03-26'
draft: false
description: 'I have been working with an AI agent as a development collaborator for months. It is not autocomplete. It is another developer on the team. This is what I have learned.'
tags: ['ai', 'development', 'claude-code', 'productivity']
cover:
  image: 'cover.jpg'
  alt: 'Developer working with an AI agent'
  relative: true
ShowToc: true
TocOpen: false
---

There is a lot of noise about AI and programming. Copilots, autocomplete, code generators... Most tools stop at suggesting the next line. But you can do something different, something more: work with an AI agent that operates as **another developer on the team**.

My digital teammate is called Jarvis. It has its own GitHub account, its own Jira access, and a workflow identical to any human developer: branch, implementation, commit, pull request, code review. I have been running this system in production for months, and this is what I have learned.

## What AI-Augmented Development is (and what it is not)

It is not asking an AI to write you a function. It is not smart autocomplete. It is not copying and pasting from ChatGPT.

AI-Augmented Development is integrating an AI agent into your real workflow, with the same tools, the same constraints, and the same process as a human developer:

- **Isolated branch** — each task on its own branch, never on master.
- **Pull request** — all work goes through PR. I review, comment, request changes.
- **Limited permissions** — the agent cannot merge or push to master. Branch protection rules prevent it.
- **Traceability** — every commit is tied to its author. The git log shows who did what.

The difference from a copilot is that Jarvis understands the full project context, not just the file you have open. It brings speed and shorter learning curves, always as a complement that makes you a better programmer, not as a replacement.

An important nuance: Jarvis is not simply opening a Claude Code or Codex session and asking it to do things. It has taken months of iteration. Policies that define what it can and cannot do, procedures for each type of task, connected tools (GitHub, Jira, MCP servers), security rules, code conventions. All of that is structured prompt engineering, and it is what turns a chat session into something viable for real work.

The upside: ask your own agent to help you refine it. When something goes wrong, ask it: *how can we fix this? How can we prevent you from making this mistake again? Review your policies and tell me where and how you propose the change.* Or for a new procedure: *draft a procedure for next time, and how we will enforce that you follow it to the letter on similar tasks.* The agent improves its own configuration. It is a self-reinforcing continuous improvement cycle.

## Productivity: what really changes

The productivity boost is real, but it does not come from where people think.

It is not that the code gets written faster. It is that **I spend more time on what matters**: designing the architecture, thinking through edge cases, reviewing with judgment. The mechanical tasks --- boilerplate, CRUDs, migrations, repetitive unit tests --- Jarvis handles those. And it does them well, because it has the project context.

Tasks that used to take hours now take minutes. Those that took days now take hours. And I am not talking about sloppy code that needs to be redone. The code passes review, passes tests, and follows project standards.

A concrete example: the <a href="https://github.com/jupaygon/symfony-dashboard-skeleton" target="_blank">symfony-dashboard-skeleton</a> --- a full dashboard with EasyAdmin 5, multi-tenant, 4 themes, role hierarchy, i18n --- we built it in hours. Not weeks. Hours. With tests, documentation, and a complete README.

## Documentation: the change I least expected

If there is one thing that has improved dramatically and I did not expect, it is documentation.

Most developers hate documenting. Others enjoy it but never have the time. I was in the second group. Now, documentation is generated as a natural part of the workflow. Jarvis documents what it implements: READMEs, PR comments, Jira ticket descriptions. And I review and adjust the tone.

The result is that my projects have better documentation than ever. Not as an afterthought you do at the end when you remember, but as part of the process. Every PR has its description, every ticket its summary, every repo its updated README.

For example, I have started creating **Project Books**: living HTML documents with charts, colors, and links that reflect the full life of a project --- from the initial brainstorm to production and beyond. They include the roadmap, architectural decisions, links to Jira tickets, current status of each piece. It is like a project dashboard, but narrative. And keeping them up to date is no longer extra effort --- it is part of the flow.

## Stack: the range opens up

This is perhaps the most unexpected change.

You can spend your entire career in a specific language and frameworks, in your comfort zone. But programming is programming --- good practices, SOLID, separation of concerns, layered architecture... are universal and independent of the language.

With this workflow, I bring the experience in those principles and Jarvis masters the syntactic details of each language. Need a Python script? A Go component? Infrastructure configuration? I can work confidently with stacks that would have previously required weeks of learning curve.

It is not that I have become a Python expert overnight. It is that I can apply over two decades of engineering judgment in any language, because syntax is no longer a bottleneck.

## Security: this is not a toy

When I tell other developers that I have an AI agent with its own GitHub account, some raise an eyebrow. Sounds risky. But it is exactly the opposite: it is the safest approach.

- **Separate credentials** — Jarvis does not have my tokens or passwords. It has its own, with limited permissions.
- **Branch protection** — It cannot push to master or merge PRs. Branch protection rules prevent it.
- **Scoped permissions** — In Jira it can create and move tickets, but cannot delete or modify configurations.
- **Full traceability** — Every action is logged under its identity. If something goes wrong, I know exactly what it did and when.

The alternative --- giving it your personal token with all your permissions --- is far more dangerous. And it is what most people do without thinking.

## What I have learned

After months with this system, these are my takeaways:

1. **AI does not replace the developer, it amplifies them.** Engineering judgment remains human. AI executes, the human designs and reviews.
2. **The workflow matters more than the model.** It does not matter whether you use Claude, GPT, or Gemini. What makes the difference is how you integrate AI into your process.
3. **Documentation solves itself** when it is part of the flow, not an afterthought.
4. **Stack is no longer a barrier.** Engineering principles are universal. Syntax is a detail.
5. **Security is not optional.** Separate identity, limited permissions, branch protection. No shortcuts.

If you are a developer and still think of AI as "the autocomplete that sometimes gets it right", I encourage you to explore this other way of working. It is not perfect, but it has radically changed how I build software.

---

*This blog --- including this post --- was built following exactly the workflow I describe. You can see it in the <a href="https://github.com/jupaygon/juanjopaya-web" target="_blank">repository</a>.*
