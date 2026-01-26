---
title: "MCP and Skills: Why Not Both?"
description: "The community keeps framing MCP and Skills as competitors. They're not. They solve completely different problems."
summary: "MCP addresses a N×M integration problem. Skills tackle tool bloat. The surface-level overlap in primitives doesn't make them competitors—it makes them complementary. Here's why the 'versus' framing is wrong and how they work together."
categories: ["Agents", "Developer"]
tags: ["agents", "mcp", "skills", "tools"]
date: 2026-01-25
draft: false
---

{{< alert icon="info" >}} 
**Bias warning:** I recently joined the MCP governance as a Core Maintainers. I
don't think that's colored my opinions here, but you should know so you can
decide for yourself. 
{{< /alert >}}

I've been seeing a lot of discussion lately about Skills. It's not surprising -
they've taken off in popularity over the last couple of months, much like MCP
did around this time last year. One thing that does surprise me is the
discussion seems to focus on an undercurrent of competition between the two.
They paint MCP and Skills as competitive ideas. Here are a few examples: 

- ["The Year in
  LLMs"](https://simonwillison.net/2025/Dec/31/the-year-in-llms/#the-only-year-of-mcp),
  Simon describes MCP's influence as waning, and the launch of Skills as somehow
  Anthropic's acknowledgement of that
- ["MCP is a Fad](https://tombedor.dev/mcp-is-a-fad/) makes similar comparisons,
  expressing that Skills should replace MCPs
- ["Skills vs Dynamic MCP
  Loadouts"](https://lucumr.pocoo.org/2025/12/13/skills-vs-mcp/) contrasts
  Skills with similar dynamic MCP patterns, again with a conclusion that Skills
  seem preferable to MCP

I can't help but feel like this framing is wrong. I don't think MCP and Skills
are competitive -- they are focused on solving completely different problems.
Comparing them is asking if TCP is better than JSON. The question doesn't make
sense because they operate at different layers of the stack.

The interesting question isn't which one wins. It's how they work together.


## Two Different Problems

The biggest, fundamental thing to understand about MCP and Skills is the problem
they were intended to solve.

### The N×M Integration Problem

You have N agents. You bounce between subscriptions for Claude, Gemini, your
IDE's copilot, that internal tool your platform team built. You also have M data
sources. GitHub, Slack, Postgres, Jira, your company's proprietary APIs.

Your agents need data. Context is king. So now, every pair needs a connector.
Agent A uses LangChain and talks to GitHub? Build an integration. Agent B uses
ADK and talks to GitHub? Build another one, because B has different conventions.
Now multiply that across every framework, every language, and every data source.
You're looking at N×M bespoke integrations. This is unsustainable.

The problem isn't that building one integration is hard. It's that building the
hundredth integration is just as hard as building the first. Over time, you
require the same functionality in multiple places, multiplying your total work
by the lesser of N or M.

### The Context Saturation Problem

You have an agent with access to 80 tools. Each tool has a schema - name,
description, parameters, types. You load all of those schemas into the context
window so the agent knows what's available.

Now your context window is stuffed with 30,000 tokens of tool definitions before
the user even says hello. The model's attention is diluted across hundreds of
options. Accuracy drops. Latency increases. Your API bill gets uncomfortable.

I wrote about this in detail in [my previous post on tool
bloat](https://kvg.dev/posts/20260110-tool-bloat-ai-agents/). The short version:
eagerly loading every tool definition is a losing strategy. The more tools you
add, the worse your agent performs.

**Why This Distinction Matters**

These problems are related - they both involve tools - but they're fundamentally
different. The N×M problem is about connectivity and standardization. The
context problem is about information architecture and context window economics.

MCP solves the first. Skills solve the second. Once you see that, the "versus"
framing stops making sense.

## The Solutions

### MCP: Solving Connectivity

MCP's core insight is simple: decouple execution from orchestration.

Instead of every agent implementing its own GitHub connector, you have a GitHub
MCP server. The server knows how to talk to GitHub. The agent knows how to talk
to MCP servers. Now any MCP-compatible agent can use any MCP-compatible server.
N×M becomes N+M.

The architecture is client-server. The agent (client) sends requests over a
standardized protocol. The server executes the actual work - API calls, database
queries, file operations - and returns results.

This buys you a few things beyond just reducing integration work:

- **Remote execution**: Tools can run locally or on a remote server. The agent
  doesn't care.
- **Centralized governance**: Because all tool calls flow through a protocol
  layer, you get a natural place to enforce authentication, logging, and access
  policies.
- **Credential isolation**: The agent never sees your GitHub token. It just asks
  the server to do things on its behalf.

MCP defines three major primitives: 
- Tools (executable functions)
- Resources (files the server can expose)
- Prompts (pre-defined templates for messages)

The protocol itself is straightforward. Nothing exotic. The value isn't in the
format - it's in the standardization. One interface, many implementations.

### Skills: Solving Context Saturation

Skills take a completely different approach. There's no protocol, no
client-server architecture, no RPC.

A Skill is a folder. Inside that folder
  - `SKILL.md` that explains what the skill does and how to use it. 
  - `scripts/` directory with executable code
  - `references/` directory with supporting documentation
  - `assets/` directory with templates or other static files

That's it. It's just files on disk.

The magic is that these files aren't included in the context by default. Instead
of loading every tool schema into context upfront, the agent gets a lightweight
index: skill names and one-line descriptions. When the agent decides it needs a
particular capability, it reads that skill's `SKILL.md` to learn how to proceed.
It might then execute scripts from `scripts/` or pull additional context from
`references/`.

This is progressive disclosure. The agent loads information on-demand rather
than all at once. Initial context load drops from tens of thousands of tokens to
a few hundred. The agent's attention stays focused on the task at hand.

## Where They Overlap

So if MCP and Skills solve different problems, why does everyone keep comparing
them?

Because their primitives look suspiciously similar.

### Prompts vs SKILL.md

MCP Prompts are pre-defined message templates. They inject instructions into the
conversation, often triggered by the user via slash commands. Type
`/migrate-schema` and the agent gets a playbook for database migrations.

A Skill's `SKILL.md` does the same job. It's a manual that teaches the agent how
to approach a particular task. When the agent reads it, those instructions shape
its behavior.

One is pushed by the user. The other is pulled by the agent. I think this is one
advantage the SKILL.md brings -- the agent can decide when to pull the context
in. This is increasingly more useful as agents become more autonomous.

### Tools vs Scripts

MCP Tools are functions hidden behind an RPC boundary. The agent sees a name and
a parameter schema. It calls the tool, the server executes the logic, results
come back.

Skills expose the same capability through `scripts/`. Local executable files
that the agent invokes via the shell. The agent submits arguments, the script
runs, output returns.

In both cases: the agent provides typed arguments to an executable entity that
performs work. This time, I think MCP actually has a clear advantage --
arguements are always well defined, typed, and it's easier to represent optional
parameters.

### Resources vs References

MCP Resources let servers expose contextual data - file contents, database
schemas, application state. Clients can fetch and attach this data to the
conversation.

Skills achieve this with `references/` and `assets/`. Static files the agent can
read when it needs background information. PDFs, CSVs, markdown docs, images.

One problem with resources is that they are generally underdefined -- while
Claude Code and Gemini CLI have landed on using `@resource-name` to pull that
resource into the context, the MCP spec doesn't say much beyond they should be
"application driven". I think this is part of the reason resource adoption is
underwhelming today -- if clients and servers can't agree on how they are used,
it removes the advantage of their format being standardized. 

### The Illusion of Competition

The primitive mapping is real. Prompts and SKILL.md both guide behavior. Tools
and scripts both execute work. Resources and references both provide context.

But this surface similarity obscures a fundamental difference in architecture.
MCP is a protocol for connecting agents to remote capabilities. Skills are a
packaging format for organizing local knowledge. They happen to offer similar
primitives because agents need similar things - instructions, execution, data.
That doesn't make them competitors any more than HTTP and ZIP are competitors
because both can deliver files.

## The Trade-offs

Beyond the primitives, there are real practical differences in when you'd reach
for one versus the other.

### Iteration Speed

Skills let you move fast (but break fast).

A Skill is just files. Edit the `SKILL.md`, save, done. The agent picks up the
change immediately. Need to tweak a script? Edit it. No build step, no server
restart, no deployment.

MCP servers require more ceremony. If you're using someone else's server and
something doesn't work the way you need, you're filing an issue and hoping they
fix it. Or forking the server and sorting out updates later. Neither is fun.

If you're writing your own server, you've got to deal with JSON-RPC over more
familiar HTTP frameworks. The official SDKs bridge a lot of this gap, but if you
outgrow them you're on your own. And every change means restarting the server -
friction that adds up fast during the rapid edit-test-edit cycle when you're
figuring out what you actually need.

Skills sidestep all of this. Agents can author and modify Skills themselves.
Because Skills are just files, an agent can read a script, debug it, and fix it.
Ask the agent to add a new capability and it can write the script directly. But
there are big tradeoffs for this speed.

### Portability

MCP wins here, and it's not close.

Skills are inherently tied to the host environment. A bash script assumes bash
exists. A Python script assumes Python is installed with the right packages. A
script using `grep` behaves differently on macOS (BSD) versus Linux (GNU). Write
a Skill on your Mac, share it with a colleague on Windows, watch it break.

MCP abstracts this away. The agent talks to a server over a protocol. What
language the server is written in, what dependencies it needs, where it runs -
the agent doesn't care. Servers can ship as Docker images or compiled binaries
with everything bundled. If you can make HTTP requests, you can use the tool.

### Security

MCP wins here too, though the picture is more nuanced than it first appears.

Skills require a permissive runtime. The agent needs shell access to discover
skills (`ls`) and read them (`cat`). It needs execution permissions to run
scripts. There is a reason why "distroless" containers exist - where you strip
out shells specifically to limit what attackers - or hallucinating agents - can
do.

MCP provides better isolation. A tool server runs in its own process,
potentially on a different machine. The agent doesn't need to touch the underlying
credentials or systems directly. You get a natural chokepoint for authentication
and audit logging.

That said, MCP has had its share of security incidents. The convenience of "just
install this npm package and add it to your config" introduces supply chain
risk. And isolation only helps so much - an agent that can call `delete_repo()`
doesn't need your GitHub token to cause damage.

Both approaches require careful thought about what you're trusting and why. But
I firmly believe that MCP can grow and improve here, while a dependency on a
shell will always be a hard sell in some environments.

## The Future is Hybrid

If MCP and Skills solve different problems, the obvious question is: why not use both?

Some people already are.

### Bridging the Gap Today

Two projects show what this looks like in practice:

[**philschmid/mcp-cli**](https://github.com/philschmid/mcp-cli) treats an MCP
server like a command-line utility. It wraps the JSON-RPC connection in a CLI
shell, so an agent can discover tools with `mcp-cli list` and execute them with
`mcp-cli call github/create_issue`. Standard shell commands, MCP execution
under the hood. This gives existing MCP servers the progressive disclosure
benefits of Skills without rewriting anything.

[**bug-ops/mcp-execution**](https://github.com/bug-ops/mcp-execution) goes the
other direction. It compiles an MCP server into a static Skill package,
generating individual script files from the server's schema. Build-time
generation that converts a server into local code. The result is a Skill that
makes MCP calls, reducing initial token load while preserving the MCP server's
functionality.

Neither is perfect. But they prove the concepts can work together.

### Two Paths Forward

I see two paths the standards could converge, making it more obvious how these
two standards should play nice. 

**Skills that contain MCP servers.** The Skill package format expands to support
MCP as a sub-component. Add an `mcp/` folder or `mcp.json` that defines which
servers the Skill depends on. When the agent activates the Skill, it reads
`SKILL.md` for instructions and spins up the specified MCP server for execution.
You get progressive disclosure for context management and MCP's protocol for
secure, portable execution.

**MCP that hosts Skills.** The Model Context Protocol expands to treat Skills as
a first-class primitive alongside Tools, Resources, and Prompts. Server
instructions become `SKILL.md`. Resources map to the references folder. The
protocol gains native support for the progressive disclosure pattern.

There's already some work being done on this front.
[SEP-2076](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2076)
proposes adding Agent Skills as a first-class MCP primitive.
[SEP-2084](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2084)
introduces named collections for grouping tools - not Skills exactly, but useful
for implementing progressive disclosure within MCP.

## TL;DR

The "MCP vs Skills" discussion is focused on the wrong thing.

MCP solves connectivity. Skills solve context saturation. Different problems,
different solutions. The surface-level overlap in primitives doesn't make them
competitors any more than a hammer and a screwdriver are competitors because
they're both on your tool belt.

My bet is we end up with some combination of both. Skills for local iteration
and context efficiency. MCP for production deployment and cross-environment
portability. Friends forever.

The future isn't MCP or Skills. It's MCP and Skills.
