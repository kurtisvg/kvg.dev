---
title: "MCP vs CLIs: The Good, the Bad, and the Ugly"
description: "CLIs have major advantages for agents today, but MCP offers security, governance, and a path to evolve."
summary: "CLIs are efficient, composable, and familiar to agents. MCP is younger and rougher, but its weaknesses can be fixed. It also offers security and governance that CLIs structurally cannot."
categories: ["Agents", "Developer"]
tags: ["agents", "mcp", "cli", "security", "governance"]
date: 2026-07-30
draft: false
---

On Tuesday, MCP finalized the latest release of the specification
(`2026-07-28`). It's [arguably the biggest change to the
spec](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)
since the initial release: it evolves MCP into a stateless protocol, removes the
`initialize` handshake, reworks how servers and clients exchange requests, and
adds several improvements that make MCP traffic more visible to traditional
networking infrastructure. It's also the culmination of a year and a half of my
work. I've been one of the biggest champions for making MCP stateless, alongside
the rest of the MCP Transport Working Group.

*Note: Everything below is my personal opinion. It doesn't represent the views
of MCP, the Transport Working Group, or my current employer (Google).*

Alongside the spec work, I've spent a lot of the past few months talking to MCP
users and MCP server producers and advising them on how to build these surfaces.
Almost always, the same question comes up: **which is better for agents, MCP or
CLIs?** Inside Google, producers want to know which to prioritize shipping.
Outside, developers want to know which to use themselves, or what they should be
recommending to their organizations.

I'll give you my answer up front: today, CLIs have lots of advantages over MCP
tools. But CLIs have disadvantages as well, and those often get left out of the
discussion. Choosing between them comes down to tradeoffs. Producers have to
decide which interfaces to prioritize and support; consumers have to decide
which tools fit their workflows.

The decision is more complicated than it's often portrayed. The key difference
is that MCP's problems have a path to being fixed, while CLIs' problems may not.

## Understanding the "CLI vs MCP" debate

The "MCP vs CLI" debate built through 2025 and picked up momentum as the year
went on. In August, Simon Willison [summarized MCP's token
cost](https://simonwillison.net/2025/Aug/22/too-many-mcps/), noting that the
usable context window in something like Amp or Cursor sits around 176,000
tokens, and that adding just the GitHub MCP defines 93 tools and swallows
another 55,000 of them.

Willison then pointed to the alternative:

> If your coding agent can run terminal commands and you give it access to
> GitHub's `gh` tool it gains all of that functionality for a token cost close
> to zero - because every frontier LLM knows how to use that tool already.
>
> I've had good experiences building small custom CLI tools specifically for
> Claude Code and Codex CLI to use. You can even tell them to run `--help` to
> learn how the tool [works], which works particularly well if your help text
> includes usage examples.

Two months later, Peter Steinberger (of OpenClaw fame) [made the same argument
more bluntly](https://steipete.me/posts/just-talk-to-it):

> I can just refer to a cli by name. I don't need any explanation in my agents
> file. The agent will try $randomcrap on the first call, the cli will present
> the help menu, context now has full info how this works and from now on we
> good. I don't have to pay a price for any tools, unlike MCPs which are a
> constant cost and garbage in my context.

Their arguments have a lot of merit, and I don't intend to spend this post
pretending otherwise. MCP made it easy to write, distribute, and share tools. A
side effect of making tools easy to distribute is that there are a lot of tools.
I wrote about the ["Too Many
Tools"](https://kvg.dev/posts/20260110-tool-bloat-ai-agents/) problem myself
earlier this year.

However, it's worth pointing out that both of these developers are
describing the same situation: an individual working on their own machine,
driving a coding agent through their own work. That's the case where CLIs win
hardest, and it's also the case where almost everything MCP offers around
security, governance, multi-user auth, and audit trails is less valuable.
OpenClaw, the CLI-first agent built by one of the people I just quoted, has
since [added MCP support](https://docs.openclaw.ai/cli/mcp).

This probably isn't the first "MCP vs CLI" post to cross your feed. My aim is to
present a fuller picture of the advantages and disadvantages that come into play
when deciding between the two. To do this, I've borrowed the "Good, Bad, and
Ugly" format:

- **The Good:** where CLIs genuinely win, and have clear advantages over using
  MCPs
- **The Bad:** where CLIs have weaknesses, but solutions are available (with
  some cost)
- **The Ugly:** where the cost can't be paid down, because it's structural to
  the model.

## The Good

### CLIs tend to be 'agent friendly'

CLIs provide an interface for humans in a terminal, and the best practices that
have grown up around them reflect that. They tend to focus on usability,
composability, and efficient use of text and screen space. Take a look at some
of the recommendations in the [Command Line Interface
Guidelines](https://clig.dev/). A lot of them also turn out to be great advice
for working with agents.

"Saying (just) enough" is a good example. The emphasis is on clarity:

> A command is saying too little when it hangs for several minutes and the user
> starts to wonder if it's broken. A command is saying too much when it dumps
> pages and pages of debugging output, drowning what's truly important in an
> ocean of loose detritus. The end result is the same: a lack of clarity,
> leaving the user confused and irritated.

That advice was written for humans, but it matters twice as much for agents. The
noise that buries the signal is also noise you're paying for by the token.

Sections like "Ease of discovery" lay down the principles for interfaces that
can be progressively discovered by any agent:

> Discoverable CLIs have comprehensive help texts, provide lots of examples,
> suggest what command to run next, suggest what to do when there is an error

The guidance that makes a CLI good for humans turns out to make it good for
agents. That's not universal (plenty of CLIs are hostile to both) but where a
CLI follows this kind of guidance, it gets agent-friendliness for free. Compare
that to MCP, where one of the most common mistakes new developers make is
porting an existing API wholesale into a server. With CLIs, common practice
pushes developers (intentionally or not) toward a more agent-friendly interface
than a traditional CRUD API.

### The CLI may already exist

If you already ship a CLI, you're mostly done. MCP is a new surface, and a new
surface is new work: every feature now has to be released in the API, in the
CLI, and in the MCP server, and each of those needs its own testing and
maintenance. As I mentioned earlier, the best MCP servers don't just wrap the
API; they rethink it with an agent-first mindset. That's more work still, and
it's work you may have already done when you designed the CLI.

### Models already know your CLI

If your CLI is popular enough, it's probably in the training data. When you ask
Claude to create a new Cloud SQL instance, it reaches straight for `gcloud sql
instances create`. It doesn't need to start at `gcloud --help` and work its way
down, because it already knows the shape of the command. That saves both tokens
and time.

This is also the one advantage on this list that can't be bought. You can write
an excellent MCP server tomorrow; you can't retroactively put it in a model's
training data.

### Programmatic composability

"Programmatic composability" is a fancy way of saying agents can interact with
and compose CLIs through a shell, which gives agents two big advantages.

First, the agent gets to choose where the output goes. By default it lands in
the context window, but for large outputs the agent can redirect to a file
instead. It can then either ignore it if it doesn't care about the result, or
come back to it later if it does.

Second, as an extension of the first, the agent can direct output into other
commands. It can compose several of them in a single turn. Say a user asks the
agent to find the Cloud SQL instances in a region without automated backups and
fix them. The agent can write this:

```bash
gcloud sql instances list \
  --filter="region:us-central1" \
  --format="value(name)" |
while read -r instance; do
  enabled=$(gcloud sql instances describe "$instance" \
    --format="value(settings.backupConfiguration.enabled)")
  if [[ "$enabled" != "True" ]]; then
    gcloud sql instances patch "$instance" --backup-start-time=03:00 --quiet
  fi
done
```

Look at what never enters the context window. `gcloud sql instances describe`
returns a large object per instance: machine type, IP configuration, maintenance
windows, labels, replica configuration. In a conventional MCP tool-calling flow,
the equivalent is a `list_instances` call whose full response lands in context,
then a `describe` call per instance whose full response also lands in context,
then a `patch` call for each one that needs fixing. The model does the
filtering, and you pay for every token it filtered through. Here the shell does
it, and you pay for none of them.

In clients that approve a whole shell command at once, there's one other
advantage: that's one approval. The user says yes once, and the whole job runs.
(Hold that thought; I'll come back to what that single approval is actually
worth.)

### Standardization by convention

CLI conventions govern the *interface* and almost nothing else. Flags, `stdout`
and `stderr`, exit codes, `--help`. That's the contract, and it's enforced by
convention rather than by any specification. Everything underneath it is
entirely up to you.

Your auth provider doesn't support Client ID Metadata Documents (CIMD)? No
problem: use whatever authentication protocol you want. Need to upload files?
MCP has no general-purpose file upload mechanism today. Want gRPC to cut latency
on large transfers? Awesome; your CLI can speak REST, gRPC, or your bespoke
carrier-pigeon transport, and the agent never has to know.

MCP specifies the transport instead of the interface. That buys you a lot, but
it also means you're living inside the protocol's constraints (whether or not
they suit your problem).

## The Bad

### Models know existing CLIs, but what about yours?

Above I argued that models already know popular CLIs and how they work. But if
yours isn't popular, your agent is going to have to learn it. The agent can run
`--help` and navigate down the subcommand tree as needed, but that isn't as
efficient as knowing from training the exact command to open a GitHub PR.

This also applies when your CLI does something close to an existing one. Models
mix up `npm`, `yarn`, and `pnpm` constantly, despite all three being heavily
represented in training data (`npm install` versus `yarn add`, or `--save-dev`
versus `--dev` versus `-D`). If a model confuses tools it has seen a million
times each, it can easily confuse your niche CLI with a popular one it
resembles. It'll probably figure out the right commands eventually, but you'll
waste time and tokens getting there.

### Bash is not a good language

Nobody actually understands Bash. You might not think this matters, since your
agent probably understands it better than you do. But what happens when your
agent proposes some multi-command monstrosity that chains `awk` and `grep` and
`curl` together in ways you can't follow? It *might* be safe. If you don't
understand it, how would you know?

Here's a much smaller example:

```bash
set -euo pipefail
failed=0

find . -name '*.json' -print0 |
while IFS= read -r -d '' file; do
  jq empty "$file" || failed=1
done

exit "$failed"
```

It looks careful. It uses `set -euo pipefail`, handles filenames safely, and
tracks whether `jq` finds invalid JSON. But [Bash runs each command in a
pipeline in a
subshell](https://www.gnu.org/software/bash/manual/html_node/Pipelines) by
default. The `failed=1` assignment disappears when the loop ends, so the script
exits successfully even when a file is invalid. `pipefail` doesn't save it
because the `jq` failure is handled by `||`.

Remember the composability example from earlier, and the single approval I
mentioned above? This is what that approval is actually worth. You (or your
users) end up with two options: decline the command because you can't verify
it's safe, or approve it anyway and hope. Neither is good, and in practice
people pick the second one every time.

Some MCP clients do offer [programmatic tool
calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
in Python instead. It offers the same context savings that make the shell
attractive because only the final output enters the model's context, not the
intermediate tool results. It also uses a language people can actually read.
Unlike shell commands, those MCP calls still have schema-defined inputs and,
when provided by the server, schema-defined outputs. The client can validate
each boundary even when several calls are composed in code. The catch is that it
isn't enabled by default and many clients don't support it yet. But it's a path.

### Portability and dependencies

The most commonly used languages today are Python and Node.js. If you want to
write a CLI in either, you'll have to make sure your user has a compatible
runtime and dependencies installed, or a correctly configured virtual
environment.

If you pick a more reasonable language for a CLI, such as Go or Rust, you'll
still need to compile it for every architecture and operating system you need to
run on. I can't speak for Rust, but for Go this isn't too bad... until you find
out that a dependency uses cgo and you've got to rebuild your compilation
pipeline from scratch.

None of this is impossible. It's just work, and the work returns every time a
new feature has to ship.

### Not every agent has a shell

Claude Code can run CLIs. The Claude web app cannot run local CLIs.

This might change. We're already seeing ChatGPT and Gemini ship "computer use"
options that give hosted assistants some limited ability to run commands in a
sandbox. But these hosted environments don't generally let users install
arbitrary CLIs today. To get there, they'd have to either constrain themselves
to a strict environment with specific Python versions and OS and architecture
combinations, or ask you to hand over a pile of build variants just in case the
job gets scheduled somewhere unexpected.

An MCP server is deployed once and reachable by any client that speaks the
protocol. A CLI needs to be installed in each execution environment, with a
compatible build or runtime available.

### CLIs are 'agent friendly', but not 'agent secure'

I argued earlier that CLIs tend to be pretty good for agents. They also have
some hard limits.

It's common advice not to give your agents direct access to passwords or other
secrets, and it's solid advice for several reasons: the host running your model
might do who knows what with that information, and the model might leak it into
the wrong tool call.

Good luck following that advice while using `psql`. You could set the password
as an environment variable, but then a single `echo $PGPASSWORD` puts it in the
context anyway. Worse, every other CLI your agent runs inherits that same
environment. That means a tool with nothing to do with your database could
compromise the secret without you ever knowing.

And `psql` is one of the *better* cases, because it won't take a password as a
flag at all. `gcloud sql users create` will. A secret passed as a flag can
appear in process listings and, depending on how the command is invoked, shell
history. That's exactly what the Command Line Interface Guidelines tell you not
to do. The guidance is right, and it doesn't help you here: the agent needs the
secret to run the command, so your options are not to use the tool or to accept
that the secret is now somewhere you can't control.

Per-command secret injection that the agent can't read could help. That brings
me to my next point.

### CLIs probably aren't changing any time soon

MCP is a specification. It has a governing body, a process for proposing changes
through Specification Enhancement Proposals, and as of this release, a formal
deprecation policy with a minimum twelve-month window. CLIs have none of that.

I want to be careful here, because the obvious objection is a good one: CLI
conventions have changed plenty over the last forty years. `--json` output,
`NO_COLOR`, single-binary distribution, help text that leads with examples: none
of these are original to Unix, and all of them are widespread now.

But every one of those changes happened by diffusion. Somebody did it, it
worked, other people copied it. There's no version number on "CLI conventions,"
no compatibility guarantee, and no way for your CLI to know whether the agent
calling it has picked up the convention you're relying on. You find out at
runtime, if at all.

So the claim isn't that CLIs can't evolve. It's that CLI evolution can't be
*coordinated*. The same freedom that lets a CLI speak anything from gRPC to
carrier pigeon now works against us: there's no central body to coordinate a
change to the convention.

MCP's process is slower by construction, and that's a real cost. What you get
for it is a process for coordinating changes, a stated compatibility window, and
a version that both sides can identify.

Two examples of this in MCP today: first, Tasks shipped with this release as an
official extension, enabling long-running tool calls that an agent can poll
rather than block on. Second, Events are still experimental in the Triggers and
Events Working Group. They would let a server wake your agent when something
happens externally.

Both look difficult for CLIs to match. A CLI can wait on a long operation today,
but that requires a long-running process somewhere, and "somewhere" is doing a
lot of work in that sentence.

## The Ugly

### Security

A CLI is executable code running in your environment. You've got to execute it
*somewhere*.

Developers have spent the last decade trying to get shells out of production
containers. [`distroless`
images](https://github.com/GoogleContainerTools/distroless) exist specifically
so that an attacker who achieves code execution has fewer tools available. That
work was hard and deliberate. Giving agents arbitrary binaries to run undoes a
good deal of it.

That's not to say safe execution is impossible. There's real work happening in
the agent sandbox space, and some of it is genuinely interesting. A sandbox
contains the risk, but it doesn't change the fact that the thing inside can run
arbitrary code.

Many of MCP's shortcomings are design decisions: the file upload problem, the
transport constraints, the tool schema tax. Someone can file a proposal for any
of them, and for several of them someone already has.

"A CLI runs code in your environment" isn't a design decision. It's the
deployment model. You can sandbox it, containerize it, and restrict what it
reaches, but each of those is a mitigation rather than a fix.

### Governance

MCP's biggest advantage over CLIs is that MCP creates a place where governance
can live. Stick an MCP gateway between all of your agents and all of your MCP
servers (and there are a *lot* to choose from) and you can centralize
observability, logging, and access control. With the right identity and
telemetry, you can see who called which tool, how many times, and how long it
took.

There is no protocol-level equivalent shared by all CLIs. A CLI makes whatever
calls it makes, to whatever endpoints it wants, over whatever protocol it chose.
If you want visibility into that, you're often reconstructing it after the fact
from process trees and network logs, separately for every platform your
developers use. OS audit logs may identify the user's account, but they often
can't tell whether the developer ran the CLI or an agent acted on the
developer's behalf.

This is also why I'd expect enterprises to gravitate toward MCP regardless of
what individual developers prefer. Organizations already control what software
runs on managed machines; agents don't create that impulse, they extend it to a
new category of tool.

## Conclusion

CLIs have lots of advantages, and today they probably outperform MCP tools for
most agents on both token efficiency and overall effectiveness. That makes
sense. CLI conventions have had forty years to settle, and models have been
trained on decades of people using them. MCP is less than two years old.

MCP continues to change and evolve. Tool Search is a step (if imperfect) toward
better efficiency. The Tasks extension shipped this week. Work on serving Skills
over MCP is underway. None of that was true a year ago.

The things that hold CLIs back aren't moving as quickly. Bash is still Bash. A
CLI still runs code in the agent's environment, and every sandbox built around
one is containment rather than a fix. Conventions still change by diffusion,
which works but provides no way to coordinate changes. These aren't problems
that people have failed to solve. They're structural limits of the local
executable model.

So: if you can afford to ship both, do it. And if you can't, make sure you take
a long, hard look before ruling out MCP.
