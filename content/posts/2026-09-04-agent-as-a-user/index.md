---
title: 'Your agent is a user of your infrastructure'
date: '2026-09-04'
slug: 'agent-as-a-user'
draft: false
description: "The moment an agent acts instead of suggesting, it stops being a tool and becomes a principal in your threat model. Seven things that cost me to learn, starting with the day the read-only account turned out to be root."
tags: ['ai', 'security', 'agents', 'development', 'claude-code']
ShowToc: true
TocOpen: false
---

For a while the interesting question about AI agents was how well they write code. The interesting question now is what they are allowed to touch.

An agent that suggests is a very articulate text box. An agent that acts has an account, a key, a session on a machine and a line in `auth.log`. That is not a tool any more. That is a principal: something your access control has to hold an opinion about.

Mine didn't, for months. This is what I found when I finally went looking, and what each thing cost to fix.

## The account that was never there

Open your server's auth log. If your agent logs in as the generic user the cloud image ships with, using the same admin key you use, then two things are true at once: it can do everything you can do, and afterwards there is no way to tell what it did from what you did.

That was my setup. The agent's "read-only" access was a sentence in a policy file, which the model reads at the start of a session. The machine itself granted root. The only thing keeping the agent read-only was that it had read the paragraph asking it to be.

Identity comes before permissions. One account per agent, per host, with its own key. Not because you distrust the agent, but because "who did this" should be answerable by the log rather than by memory.

## Least privilege is harder than it sounds

So I wrote a sudoers file. Diagnostic verbs only: read the journal, check a unit's status, read the kernel ring buffer. No restarts, no stops, no data. Very reasonable.

It granted uid 0. By two unrelated routes.

The first: in sudoers, a command listed with no arguments permits *all* arguments. `journalctl` and `systemctl status` both went in bare, and both page their output through `less` by default. From `less`, `!sh` opens a shell. Under sudo, that is a root shell, obtained through a permission I had written down as "let it read the logs".

The second: `openssl x509` reads like an inspection command. It also accepts `-out`, which writes a file: any file, including the sudoers that had just granted the permission.

The lesson isn't "I wrote a bad sudoers". It's that read-only is a property of an *invocation*, not of a binary. Any command that can page, shell out or write a file is a write primitive wearing a read verb's name. The fix was to pin the pager on every specification and leave nothing without declared arguments.

## Enumerating the dangerous doesn't work

Second control: the agent may open SSH only to hosts on an allowlist.

That allowlist turned every host on it into a pivot to anywhere else.

The failure had three faces and they are all the same mistake. SSH option keys are case-insensitive, so a guard matching `ProxyCommand` sails past `proxycommand`. Quoting broke the match. And, worst of the three, neither `Hostname` nor `-F` contains the word "proxy". Yet `Hostname` redirects the connection to a different machine, and `-F` points ssh at a config file that can define absolutely anything.

I was enumerating the dangerous options. There is no finite list of those. The only version that holds is inverted: an option passes only if its key is on the list of options that *cannot move the connection*, and `-F` is refused outright.

The same guard taught me a second thing on its own. It searched for `-F` anywhere in the command line, so an `awk -F:` inside the remote command tripped it. A flag belongs to a program, not to a string: the guard now reads only the tokens that ssh itself owns, walking from the leading `ssh` to the host and stepping over the value of any option that takes one, because `key` in `-i key` is not the host either. The attached form needs no special case: `-ikey` is a single token starting with a dash, so it never gets mistaken for a host. Everything past the host belongs to another program's command line. The exception is a nested `ssh`, which opens a connection of its own and therefore gets its options read too.

## The test that never ran the dangerous shape

Both of those controls had passing tests. The tests asserted on substrings of the rendered configuration. They were green the entire time the sudoers granted root.

A test that matches text does not test a control. What tests a control is the shape it is supposed to refuse. The sudoers test now parses the actual command specifications, and I checked that it fails when the dangerous form is put back. The SSH dry-run test puts a fake `ssh` on the path that fails loudly if anything invokes it.

If you have never watched your security test go red, you don't have a security test. You have a comment written in a testing framework.

## An instruction is not a control

This is the one I would keep if I could only keep one.

I run a second model that watches the main agent's turns and returns a verdict. When it saw something serious it escalated, and the escalation was *text injected into the transcript*: stop, and take this to the human.

Read that again. The verdict about the agent was delivered to the agent, which then decided whether to comply. The defendant received the prosecutor's case and ruled on it.

In one session the judge escalated six times. The agent dismissed all six as false positives and carried on. It was right all six times. That doesn't save the mechanism, because one of the six was a `git push --force`, and whether that happens is not the agent's call to make.

Now an escalation writes a lock file, and a hook denies *every* tool while that file exists. With no tools available the only remaining action is writing text, so the only exit is addressing the human. The agent cannot clear it either: the file write and the shell command it would need are denied by the same hook. It lifts on a human action and on nothing else.

The corollary matters just as much. Only what code can corroborate is allowed to escalate: three identical retries, counted; a command on the irreversible list, matched. Everything else warns without freezing anything. What stops a session is always a verified fact, never a model's opinion. Otherwise you have moved the non-determinism from the agent to its judge, which is [not what deterministic guardrails are for](/en/posts/2026-05-13/determinism-in-ai-workflows/).

## A control that covers one path is not a control

Six of my hooks hung off the file-editing tools. None off the shell.

So a heredoc into `python3`, a `cat >`, a `sed -i`, a `tee`: each of those wrote the file while walking past all six at once. Not one rule bypassed: every rule, simultaneously, through a door nobody had put a guard on. The hooks did not fail to judge that code. They never saw it.

When you count your controls, count the *paths* into the thing you are protecting, not the rules you wrote. An agent has many more ways to write a file than a human ever bothers to use.

## Credentials come back on their own

Short one, and my favourite. I found my own private keys on a machine the agent uses and deleted them. A couple of weeks later they were back.

The provisioning recipe copied them, by design, and it runs often. Deleting a secret from a machine buys you exactly the time until the next run of whatever put it there. The fix is never on the machine; it is in the thing that builds the machine.

## Closing

None of this is exotic security work. It is the ordinary kind (identity, least privilege, allowlists over denylists, tests that can fail, a control on every path), applied to a colleague who happens to be a language model, works at three in the morning, and reads your policy file as advice.

The uncomfortable part is the timing. All of the above was already true the day I handed the agent its first credential. I just hadn't looked. And the more of the work you [hand to a loop that runs without you](/en/posts/2026-06-17/loop-engineering/), the longer the gap between the mistake and the moment anyone notices it.

So start with the log. Whether your infrastructure can tell your agent apart from you is a yes-or-no question, and the answer takes thirty seconds to find.
