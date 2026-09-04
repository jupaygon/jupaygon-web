---
title: 'Agents as users of the infrastructure'
date: '2026-09-04'
slug: 'agent-as-a-user'
draft: false
description: "Notes from auditing what my agents were actually allowed to do on my machines: the sudoers that granted root, the SSH allowlist that let them hop anywhere, and three other things I had assumed were fine."
tags: ['ai', 'security', 'agents', 'development', 'claude-code']
cover:
  image: 'cover.jpg'
  alt: 'An agent and a person entering a server room through separate doors, each with its own badge and its own line in the access log'
  relative: true
ShowToc: true
TocOpen: false
---

I have agents that write code, open PRs and SSH into machines. In August I sat down to check what permissions they actually had while doing it, rather than what my policy files said they had.

What I found were the same access-control problems as always, the ones that show up the moment somebody who is not you starts logging into your servers. Why was that happening? Because I was treating the agents as tools and not as users.

## One account per agent

The agent logged into production as `ubuntu`, using the same admin key I use.

It could do everything I can do, and the auth log could not tell its sessions from mine. The "read-only" part of its access was a sentence in a policy file that the model reads at the start of a session; the machine itself granted root.

The fix is one account per agent per host, with its own key. It costs a provisioning script and it makes "who did this" a question for the log rather than for my memory, which, as I tend to say, is not my strongest feature.

## The sudoers gave root

Next step was a sudoers file with diagnostic verbs only: read the journal, check a unit's status. No restarts, no stops, no data.

It granted uid 0 by two routes.

In sudoers, a command listed with no arguments permits all arguments. `journalctl` and `systemctl status` both went in bare, and both page through `less` by default. From `less`, `!sh` opens a shell, which under sudo is a root shell.

The other route was `openssl x509`, which I had filed under inspection commands. It accepts `-out`, so it writes a file: any file, including the sudoers that had just granted the permission.

Read-only turned out to be a property of the invocation, not of the binary. Anything that can page, shell out or write a file is a write primitive with a read verb's name. The specifications now pin the pager and none of them is left without declared arguments.

## The SSH allowlist let it hop to any host

The next control was an allowlist of hosts the agent may SSH into. It made every host on the list a jump point to anywhere else.

Three ways past it, all the same mistake. SSH option keywords are case-insensitive, so a check for `ProxyCommand` misses `proxycommand`. Quoting broke the match. And neither `Hostname` nor `-F` contains the string "proxy", while `Hostname` sends the connection to another machine and `-F` points ssh at a config file that can define anything.

I was enumerating dangerous options, and that list has no end. It is now inverted: an option passes only if its keyword is on the list of ones that cannot move the connection, and `-F` is refused outright.

The same guard had a second bug worth mentioning: it searched for `-F` in the whole command line, so `ssh host 'awk -F: ...'` was blocked. It now reads only the tokens ssh itself owns, from the leading `ssh` up to the host, stepping over the value of options that take one, since `key` in `-i key` is not the host. The attached form needs no special case, because `-ikey` is a single token starting with a dash. Past the host it is another program's command line, except for a nested `ssh`, which opens its own connection and gets its options read too.

## The tests were green the whole time

Both controls above had tests. They asserted on substrings of the rendered configuration, so they passed while the sudoers granted root.

Matching text does not test a control. The sudoers test now parses the actual command specifications, and I checked that it fails when I put the dangerous form back. The SSH dry-run test puts a fake `ssh` on the path that fails if anything calls it.

Mutating the code to see both tests go red was a few very well spent minutes.

## The hawk's verdict was text in the transcript

I run a second model that reviews the main agent's turns, what [Alejandro Vidal](https://x.com/dobleio) calls a hawk. When it flagged something serious it escalated, and the escalation was text injected into the transcript: stop and check with the human.

So the verdict about the agent was delivered to the agent, which then decided whether to act on it (and that is with Alejandro explaining it rather well 😅). In one session the hawk escalated six times, the agent dismissed all six as false positives and carried on, and it was right all six times. One of the six was a `git push --force`, which is not the agent's decision to make regardless of who turns out to be right.

Now an escalation writes a lock file and a hook denies every tool while that file exists. With no tools left, the only possible action is writing text, so the only exit is addressing the human. The agent cannot clear the lock either, because the write and the shell command it would need are denied by the same hook.

The other half of that change: only things code can corroborate are allowed to escalate. Three identical retries, counted. A command on the irreversible list, matched. Everything else warns without freezing anything, because a model's opinion stopping a session just moves the [non-determinism from the agent to its hawk](/en/posts/2026-05-13/determinism-in-ai-workflows/).

## The hooks hung off Write/Edit, not off Bash

Six content hooks were wired to the file-editing tools and none to the shell.

A heredoc into `python3`, a `cat >`, a `sed -i` or a `tee` therefore wrote the file past all six at once. It was not one rule being skipped: it was every rule, through a path nobody had guarded. The hooks did not judge that code badly, they never received it.

Worth counting the paths into whatever you are protecting rather than the rules you wrote. An agent uses more of them than a person does.

## What I would check first

If you have agents acting on your infrastructure, the cheapest first check is the auth log: see whether you can tell their sessions from yours. Everything else in this post came out of that one.

The rest is ordinary work. Identity before permissions, allowlists instead of enumerating what is dangerous, a test you have seen fail, a control on every path into the thing. The only unusual part is that the user on the other side works at three in the morning, and reads your policy file as advice. The more of the work happens inside [a loop that runs without you](/en/posts/2026-06-17/loop-engineering/), the longer it takes you to notice.
