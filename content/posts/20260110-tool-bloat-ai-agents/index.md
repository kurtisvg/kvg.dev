---
title: "Stop Drowning Your Agent in Tools"
description: "Why loading 50 tools into your agent's context window tanks accuracy, explodes latency, and drains your budget—and what to do about it."
summary: "MCP servers are proliferating, but many ship with serious Tool Bloat problems. Learn why excessive tool definitions degrade agent performance and how to build lean, effective systems."
categories: ["Agents", "Developer"]
tags: ["agents", "mcp", "tools"]
date: 2026-01-10
draft: false
---

## The Tool Bloat Epidemic

MCP servers are everywhere now. In the last few months, the ecosystem has
exploded — there's an MCP server for everything from filesystem operations to
Slack integrations to cloud infrastructure management. Developers love them
because they're plug-and-play: add a config line, and suddenly your agent has
new capabilities.

Unfortunately, many of these servers are shipping with far too many tools, and
unintentionally ruining agent performance.

Take a few real examples:

- **[GitHub's official MCP
  server](https://github.com/github/github-mcp-server?tab=readme-ov-file#tools)**:
  Nearly 50 tools. That's every possible GitHub operation: repository
  management, issue tracking, pull requests, CI/CD workflows, security scanning.
  All loaded into your context whether you need them or not.
- **[Playwright MCP
  server](https://github.com/microsoft/playwright-mcp#tools)**: 24 tools (with
  optional add-ons bringing it higher). Browser automation, screenshots, console
  monitoring, network capture, JavaScript execution, form filling. Many of which
  you'll never use for a single task.
- **[Chrome DevTools MCP
  server](https://github.com/ChromeDevTools/chrome-devtools-mcp?tab=readme-ov-file#tools)**:
  26-27 tools for performance analysis, debugging, and browser automation. Want
  to check one performance metric? You've loaded tools for emulation, network
  inspection, accessibility testing, and tracing.

While 20 tools for a single server doesn't sound like a lot, that can easily be
15,000 to 20,000 tokens. With 3-4 MCP servers, you'll cross the ~50,000 token
threshold — almost 25% of a typical 200k context window — before your agent does
any actual work.

This isn't just inefficient. It's quietly degrading your agent's accuracy,
exploding its response time, and draining your API budget.

## What is Tool Bloat?

Tool Bloat is the accumulation of excessive cognitive load in an agent's system
prompt. You might notice the following symptoms: 

**Excessive Tool Count**: Your agent probably doesn't need 50 tools to answer a
user's question — it needs 3. But because you need multiple tool servers-one for
your database, one for filesystem operations, one for your cloud provider, one
for Slack, one for GitHub, one for monitoring-you end up with 60-100 tools
loaded into every single inference call, depending on the servers you've
connected.

**Too Many or Too Complex Parameters**: Tools with overly complex parameter
structures invite the model to make mistakes. Consider a tool with nested
dictionaries for configuration, maps of key-value pairs, and free-text fields.
Every additional layer of nesting exponentially increases the decision space the
model has to navigate. Compare these two examples:
```python
# Bad: Complex, error-prone
def execute_database_query(
    # {host, port, database, credentials: {username, password}}
    connection_config: dict,  
    query: str,
     # arbitrary key-value pairs for query parameters
    params: dict, 
     # {timeout, retry_policy: {max_attempts, backoff_ms}}
    options: dict 
)

# Good: Simple, constrained
def execute_database_query(
    query: str,
    timeout_seconds: int = 30
)
```

The first tool has 4 parameters, three of which are complex nested dictionaries
requiring the model to hallucinate valid structure at multiple levels. The
second has 2 parameters, all primitives with clear constraints.

**Verbose Descriptions**: That 500-word docstring explaining every edge case and
historical quirk of your `create_user` tool? The model has to process that on
every inference, and most of it is irrelevant noise. Tool descriptions are for
the agent, not a place to paste your entire Swagger file.

To be clear: MCP didn't start this fire, but it's certainly poured gasoline on
it. The protocol decouples tool authoring from consumption, which is powerful.
Unfortunately, it also makes it trivially easy to import someone else's
monolithic server. A developer needing only `git_status` will typically install
the entire Git MCP server, injecting dozens of irrelevant tool definitions into
their context without a second thought.

It's a convenience trap.

## Why Tool Bloat Kills Agent Performance

Tool Bloat isn't just a theoretical concern; it has measurable, non-linear
impact.

### Accuracy: Context Rot is Real

If you've shipped agents in production, you've probably seen this: your agent
starts picking the wrong tools, hallucinating parameters, or just… giving up.
[Jenova AI](https://www.jenova.ai/en/resources/mcp-tool-scalability-problem)
validated it publicly. Their results showed that once you exceed around **20
active tools**, accuracy starts to degrade noticeably. The exact threshold
varies by model and task complexity, but the pattern is consistent.

Chroma Research named this phenomenon ["Context
Rot"](https://research.trychroma.com/context-rot). The drop can be steeper than
you'd expect.

Here's why: Transformer models distribute attention probability across the
entire context window. When you flood that window with unused tool definitions,
these tokens act as "distractors" that dilute the attention scores. The model's
retrieval heads (the mechanisms responsible for deciding which tool to use) have
to work harder to distinguish signal from noise. As the ratio of irrelevant
tokens increases, reasoning accuracy collapses. It's like trying to pick a
single voice in a room with 50 people shouting at you. 

### Latency: Death by a Thousand Tokens

More tokens in the context window means longer processing time to generate a
response. This is a linear relationship at the model level, but in agentic
systems, it often compounds exponentially.

A single user request rarely triggers just one inference call. Most agent
frameworks execute multiple hidden steps: reasoning loops, tool invocations,
sub-agent delegations, reflection phases. If Tool Bloat adds .5 seconds of
latency to each inference step, a 5-step workflow now takes 2.5 extra seconds.
Your agent goes from snappy to sluggish.

This hits hardest on the models you actually want to use. The largest, most
capable models (Gemini Pro, Claude Opus, etc) also take the longest to process.
You're paying for the best reasoning engine available, and then crippling it
with irrelevant tokens.

### Cost: Burning Money on Noise

LLM API pricing is generally character or token-based. Every tool definition you
load gets billed on every turn of every conversation. If you're carrying 30,000
tokens of unused tool schemas across a 10-turn support conversation, you're
paying for 300,000 tokens instead.

And because Tool Bloat also tanks accuracy and latency, you're not just spending
more — you're getting worse results. It's a lose-lose-lose.

## Solutions: Keeping Your Agent Lean

Fortunately, Tool Bloat is solvable. It requires discipline at the tool design
level and intelligence at the architecture level. Here's how to fight back.

### Tool Design Hygiene

If you're building or maintaining MCP servers, start here. Every tool you ship
should be as simple as possible.

**Parameter Simplicity**: Avoid complex types like maps, nested JSON, or
protocol buffers. Stick to primitives: strings, integers, booleans. Keep
parameter counts under 5. Every additional parameter increases the risk of
hallucination.

Prefer enums over free-text fields. Instead of accepting any string for a
`status` parameter, enumerate the valid options: `['open', 'closed',
'pending']`. This constrains the model's decision space and eliminates an entire
class of errors.

**Concise Descriptions**: Resist the urge to write comprehensive documentation
in your tool docstrings. Focus on what's relevant to *most* use cases, not every
edge case. Every extra sentence is cognitive load. Save the details for external
docs. Your tool schema is not the place for a user manual.

### Granular Toolsets

Stop shipping monolithic MCP servers. Unbundle your capabilities into focused
toolsets.

**Aim for 5-8 tools per toolset.** This is the sweet spot where an agent can
reason effectively about its options without drowning in choices.

Organize tools by **user journey**, not by technical domain. Don't build a
"Database MCP Server" with 40 tools for every conceivable database operation.
Instead, build:

- `database-admin`: `create_instance`, `resize_instance`, `configure_replicas` —
  for infrastructure and DevOps workflows
- `database-query`: `execute_query`, `list_tables`, `describe_schema` — for data
  analysis and application development

When our team launched [Toolbox for
Databases](https://github.com/googleapis/genai-toolbox) early last year, we
introduced a concept called "toolsets." Users can select their tools and specify
them in a `tools.yaml` configuration, giving them fine-grained control over
which tools they access. They can then load these tools selectively by appending
the toolset name to the URL:
```
http://your.toolbox.com/mcp/database-query
http://your.toolbox.com/mcp/database-admin
```

We've since seen other MCP servers adopt similar patterns. GitHub's MCP server
[supports toolset
selection](https://github.com/github/github-mcp-server?tab=readme-ov-file#specifying-toolsets)
via the `--toolsets` flag when running locally, letting you load only specific
capabilities (repos, issues, pull_requests, actions, etc.).

Supabase's MCP server uses a `feature group` parameter as well:
```
https://mcp.supabase.com/mcp?features=database,docs
```

Note: There's no standard for this in MCP yet — but there is a [Primitive
Grouping Working
Group](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions?discussions_q=is%3Aopen+Primitive+Grouping+label%3Anotes+sort%3Adate_created)
actively discussing solutions. 


### Progressive Discovery

For agents with more than a handful of tools, don't load anything upfront. Treat
your context window like carry-on luggage: pack only what you need for this leg
of the trip.

Progressive Discovery (sometimes called "Dynamic Discovery" or "Progressive
Disclosure") is about selectively loading or compressing tool descriptions (or
other context) until it's actually needed to solve the problem. This keeps your
active context window lean while still maintaining access to broad capabilities.

#### Search Tool Pattern

Provide a single `search_tools(query)` function instead of loading all tool
definitions. The agent searches for capabilities as needed ("find tools for
querying postgres"), and the system dynamically loads only the relevant schemas
returned by that search.

I don't think it's a good idea for individual MCP servers to implement search
tools — otherwise users will end up with multiple search tools. Instead, I
expect orchestrators (Gemini CLI, Claude Code, ADK, LangGraph, etc.) to add
search tools that automatically incorporate tools added to the agent. Claude
Code recently shipped a native [tool search
tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
for exactly this purpose. Cursor followed with [dynamic context
discovery](https://cursor.com/blog/dynamic-context-discovery), syncing tool
descriptions to a folder for on-demand retrieval.

#### Skills (Unpackable Context)

Anthropic introduced the concept of [Claude
Skills](https://github.com/anthropics/skills) and standardized it as an
open-source spec. The idea: an agent interacts with a high-level "Skill"
definition. When activated, the agent unpacks the full instruction set and tool
definitions into context, executes the mission, and then repacks (deletes)
everything. Context pollution only during execution, not permanently. Think of
it as checking out a library book instead of buying the whole shelf.

Currently, Skills rely on executing bash or python scripts rather than native
tool definitions. I don't know if I expect this to stick: it raises a number of
issues around dependencies, portability, and challenges for security and
enterprise governance. Emerging tools like
[`mcp-execution`](https://github.com/bug-ops/mcp-execution) and
[`mcp-cli`](https://github.com/philschmid/mcp-cli) are bridging the
Skills-to-MCP gap, enabling execution of MCP tools within the Skills framework.

#### Agent-to-Agent (A2A) Delegation

Instead of loading a specialized toolset like `database-admin-tools` into your
primary agent's context, delegate the task to a sub-agent. Google's [Agent2Agent
(A2A) Protocol](https://a2a-protocol.org/latest/) is an open standard designed
specifically for this—enabling agents to discover each other, negotiate
interaction modalities, and collaborate on tasks without exposing internal state
or tools.

Your primary agent never loads the 30 database tool definitions. It just asks
the specialist "find overdue invoices" and gets back the answer. Context stays
lean, results stay complete.

This pattern keeps the primary agent's context clean and offers better context
management. You can be selective about which context gets passed to the
sub-agent and what instructions are included in its prompt. This could backfire
if you omit critical context, leading the sub-agent to make the wrong call.
However, well-designed sub-agents can attempt to clarify or disambiguate when
they lack necessary information.

#### Mixing Strategies

These strategies aren't mutually exclusive. A sophisticated system might use
search to discover broad tool categories, skills to temporarily unpack a
specific toolset, and delegation to offload complex execution to a sub-agent.
The goal is always the same: keep the active context window lean, relevant, and
free of distractor tokens.

## Summarizing This Context Window

More tools don't make your agent smarter. They make it anxious and confused. 

Working with agents that need broad capabilities requires discipline. Audit what
you're loading. Cut what you don't need.

This isn't optimization for the sake of optimization. It's the difference
between an agent that works and one that doesn't.

If you're building MCP servers: stop shipping monolithic bundles. Unbundle your
tools into focused, journey-based toolsets. Your users shouldn't have to import
40 tools if they only need 2.

If you're building agents: audit your context window. Count your tool
definitions. Cut ruthlessly. Every irrelevant tool you remove is a boost to
accuracy, a drop in latency, and a reduction in cost. Your users will notice the
difference immediately.

As the MCP ecosystem matures, expect better standards for toolset organization
and progressive discovery. Until then, we have the patterns and the discipline
to build lean, effective agents today.

Your context window is expensive real estate. Stop filling it with junk.
