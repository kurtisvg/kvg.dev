---
title: "Agent Experience is the new DevX"
description: "Developer Experience was always a hard sell. Agent Experience comes with receipts."
summary: "Agents don't work like developers. They can't read the docs, iterate on a bad design, or grumble their way through a weird API. They just pay in tokens. Here's why APIs designed for agents are eating the world, and the principles to design for them."
categories: ["Agents", "Developer"]
tags: ["agents", "agx", "api-design", "cli", "devx", "mcp"]
date: 2026-05-25
draft: false
---

I’ve always been fascinated by the idea of “developer experience”. Projects or
libraries that are easy for developers to use – that are intuitive and make it
easy and fun to get things done. Writing code that works is one thing, but the
idea of creating something that’s pleasant to use felt like the real work. The
peak of this challenge was writing libraries or SDKs for other developers. The
shape of the API could not only be intuitive, but also steer users into making
the correct decisions, making it easy to do the more secure thing, or the more
scalable thing.

The benefits felt immediately obvious to me – the nicer the experience, the more
people would want to use your thing. If it was easier to use, it would be easy
to prototype with. Easier to sell to skeptical engineers. People would prefer
the nicer thing and tell their friends about it.

Unfortunately, I learned from working in Developer Relations that DevX is often
talked about, but rarely prioritized. One specific reason is that it’s often
hard to quantify the benefits – how do you measure the return on taking an extra
week to design a “good” API? Developer productivity is challenging to measure.
Did that task take 2 extra weeks because the platform is complicated, or because
it was a more junior developer? Often decisions on what platform to use are made
higher up the food chain (often by folks that are no longer developing). If your
CEO picks the platform with the crappy experience, you are stuck dealing with
the extra overhead anyway.

When I started working with Agents, I quickly realized that agents worked better
with APIs that were designed for them. APIs that were agent-friendly (or
‘agent-first’). Agents were better, faster, cheaper when using things that had a
good “Agent Experience”. And unlike Developer Experience, it was significantly
easier to quantify. Run an agent on a task with your API. Count tokens. Count
retries. Count failures. Time it. Run it again with a better-designed API and
compare. The gap is the dollar amount you've been failing to put on bad design
for years. I've started calling this Agent Experience (or AgX) and unlike
DevX, it comes with receipts.

## Developer Experience, but for Agents

So why do agents need their own version of this? Because they don't work like
developers.

When a developer picks up your API, they read the docs. Maybe find a sample, get
it running, modify it for their use case. They write some tests. They send it
off for review. Mistakes get caught. The whole loop is slow, deliberate, and
pretty forgiving of a messy API. A confused developer can grumble their way
through your weird API shape.

An agent is doing none of that. When you ask it to do something, it generates
some text that calls your tool. That's it. No reading the docs first, no running
a sample, no second draft. Yes, it could read documentation. It could iterate on
a tool call recursively until something works. But every one of those steps
costs tokens and time. The agent equivalent of a developer grumbling through
your weird API ends up eating a huge chunk of your weekly quota and takes twenty
minutes longer than it should.

This is one of the (several) reasons MCP came at an opportune time. The best MCP
servers aren't thin wrappers over existing APIs. They're what people now call
"agent-first" or "agent-friendly": fewer endpoints, flatter parameters, errors
that explain themselves. This is not a new concept – folks have been making this
case for over a year, in posts like *[Your API is not an MCP][your-api-not-mcp]*
and *[Why Can't I Just Use an API][why-not-api]*.

CLIs are having a similar moment for the same reason. A CLI is already designed
for someone typing into a terminal. Each call is hand-crafted by a human, so
default CLI design naturally gravitates toward being intuitive and concise.
Progressive disclosure is baked in (subcommands). Documentation is baked in
(`--help`). Output is shaped for the terminal window, which means it's already
compact and scannable. Agents inherit all of this for free. The CLI you wrote in
2018 might already be a better agent interface than the REST API you shipped
last year.

[your-api-not-mcp]: https://www.youtube.com/watch?v=eeOANluSqAE
[why-not-api]: https://auth0.com/blog/mcp-vs-api/

## Why should you care about AgX?

The pushback I hear most often is "the models will figure it out." Give it
another 12 months, throw a smarter model at your janky API, and it'll eventually
figure it out.

This is probably true. Models have gotten meaningfully better in the last 6-8
months. They handle tasks and complexity that would have stumped them eighteen
months ago. But how long will it take for a model to come along that understands
the janky API? And given that better models are also generally bigger (and more
expensive), how much more will it cost to figure it out?

There's a token crisis happening right now. Demand is wildly outpacing supply.
Subscription plans on Claude Code, Codex, and similar tools have been tightening
quotas for months. Reddit fills up every day with developers complaining their
plans don't go as far as they used to, or that the models feel dumber (which,
sometimes, they are – quantized versions get rolled out when capacity gets
squeezed). [Users are increasingly vocal about feeling
squeezed][users-squeezed]. This isn't resolving next month. The models keep
getting bigger and more expensive to run. The user bases keep growing. Not just
developers, everyone. Building enough datacenters to catch up seems to be a
multi-year problem.

So while you wait for the model to figure out a bad API, developers pay extra.
And so does every user of every agent that touches your APIs.

Even when the supply problem eventually resolves and models do get good enough
to muscle through anything, agent-friendly tools will still be faster. Still
cheaper. Still more secure, because an agent pointed at the right thing makes
fewer wrong calls. You're not necessarily designing for the worst model, you're
designing for the agent that lands on your product and needs to ship in 3
minutes, not 30.

If you're using agentic tools, you'll want to use the ones built on APIs
designed for them. They're faster, cheaper, and your agents will fail less
often. If you're building APIs, your users will notice the difference. They
already are.

[users-squeezed]:
    https://www.reddit.com/r/codex/comments/1tgshea/the_cycle_of_enshitification/

## What Makes for a Good AgX

Most of what makes a good API for developers also makes a good API for agents.
Flat parameters, clear errors, sensible defaults. You can get pretty far by
following any half-decent book or guide on API design, such as [Google's API
Improvement Proposals][google-aip]. However, DevX is honestly more of a
discipline (and not one all engineers follow) than a documented standard, which
makes finding canonical references tricky. And not everything maps cleanly to an
“agent-first” world.

Internally, my team has started an “MCP Style Guide”, modeled loosely on
Google's [Java][google-java-style] and [Go][google-go-style] style guides. We
wanted a reference engineers could consult with rules they could apply to make
their APIs better. As an added benefit, we’ve found that the style guide also
works great as instructions for LLMs, and has made it easier to automate reviews
of APIs using the guide.

Here are the principles we've found most useful. Some are MCP specific, but most
are applicable to CLIs, Skills, or anything else your agent might consume.

[google-aip]: https://google.aip.dev/
[google-java-style]: https://google.github.io/styleguide/javaguide.html
[google-go-style]: https://google.github.io/styleguide/go/

### Control Your Context

The number one complaint MCP users have today is often [context
bloat][context-bloat]. Every tool you wire into an agent's context window has a
cost. Every parameter you document, every example you include, every "additional
notes" section, every one of those tokens is being read into the model on every
turn, eating up your window.

This has a noticeable impact on agent performance. Anyone who's spent real time
with coding harnesses has watched the post-compaction drop-off: one minute the
agent is humming, the next it's lost the thread and needs reminders about
decisions made at the start of the session.

The trick is to expose information in *moderate* chunks that the agent can pull
in as needed (often called progressive disclosure). You need to follow the
goldilocks principle here: not too little, but not too much. The Agent Skills
specification provides [some guidance][skills-best-practices]: it caps skill
content at 500 lines or roughly 5000 tokens (call it 6-8 pages). My
recommendation is to treat that as the ceiling, not the target. Most of the time
you want to be in the 1-3 page range.

While MCP doesn’t do much to enable progressive disclosure, things like Skills
and CLIs have the concepts built into their standards. Subcommands hide
complexity behind a hierarchy. Output is shaped for a terminal, which forces
brevity.

In addition to being concise, make sure you are clear and unambiguous with your
instructions. This will help you make the most of what context you use, and help
the agent do the right thing the first time.

[context-bloat]: {{< ref "20260110-tool-bloat-ai-agents" >}}
[skills-best-practices]:
    https://agentskills.io/skill-creation/best-practices#aim-for-moderate-detail:~:text=The%20specification%20recommends%20keeping%20SKILL.md%20under%20500%20lines%20and%205,000%20tokens%20—%20just%20the%20core%20instructions%20the%20agent%20needs%20on%20every%20run.%20When%20a%20skill%20legitimately%20needs%20more%20content,%20move%20detailed%20reference%20material%20to%20separate%20files%20in%20references/%20or%20similar%20directories.

### Avoid Choice Paralysis

LLMs, like humans, get overwhelmed by too many options. The more decisions you
stack into a single moment, the more likely the agent picks the wrong one.

The rule of thumb I use is 5-8 choices at any given decision point. This applies
to things like the number of tools in an MCP server, the values in an enum, or
the subcommands grouped at each layer of a CLI hierarchy. Keep each skill lean –
it’s better to have more skills than one huge skill. If you find yourself with
30 of anything, that's usually a sign to group, not to add another.

These are ballparks. You can go over, but you should weigh the decision
carefully before you do. Every extra option is cognitive load that the agent
pays for on every single call from now on.

### Clear, Actionable Errors

The worst thing an API can do is return `400 Bad Request` with no explanation.
What made it bad? What should I change? Should I retry? The agent has no idea,
and neither would you.

Agents are surprisingly effective when they have a clear goal and can iterate
toward it. The [Ralph Wiggum][ralph-wiggum] loop is based on the idea that an
agent doesn't need to be the smartest; it just needs a clear goal and enough
iterations to get there. You can help poor Ralph exit the loop sooner by telling
it how to self-correct instead of letting it figure it out on its own.

A useful error message does two things: it tells the agent exactly what went
wrong, and it tells the agent what to do about it (or where to look). So if your
API expects a date in `YYYY-MM-DD` format and the agent sends `05-24-2026`, your
error shouldn't just say "invalid date." It should say "The 'date' field must be
in YYYY-MM-DD format. Retry with the correct format." Now the agent has a clear
next step instead of a guessing game.

One myth worth killing: errors don't need to be "machine-readable" in any
structured sense. JSON, XML, plain English; the LLM reads them equally well.
Just make sure the model can tell what broke and how to fix it.

[ralph-wiggum]: https://awesomeclaude.ai/ralph-wiggum

### Keep it simple (and typed!)

Don't make the agent jump through extra hoops to use your tool. Skip the complex
config objects. Skip the nested dictionaries. Keep your parameters flat.

A concrete example: Google APIs love resource URIs in the form
`projects/$PROJECT/regions/$REGION/resources/$RESOURCE`. We ran evals internally
comparing this style to taking `project`, `region`, and `resource` as three
separate flat parameters. The flat version performed measurably better. The
effect was more pronounced on cheaper models but still showed up on the frontier
ones.

While you're at it, use types. If you want an int, take an int, not a string the
agent has to format correctly. If a parameter only accepts certain values, make
it an enum and list the valid options in the description. RED, BLUE, GREEN, no
guessing about whether "Red" or "scarlet" is going to work. And when the agent
does pass the wrong value (it will), remember to return a list of valid options
in the error.

### Be Idiomatic

LLMs are statistical machines. They've seen a million CLIs and a million APIs
and have strong priors about how each one should look. When you do what everyone
else does, the agent's default behavior is correct. When you don't, the agent
has to fight its own training to use your thing.

For CLIs, that means following POSIX conventions. Single dash for
single-character short options (`-v`). Double dash for long options
(`--verbose`). Positional arguments where you'd expect them. None of this is
surprising to a human, and that's the point: it shouldn't be surprising to a
model either.

For tool calls, names tend to be snake_case verb-noun pairs over bare nouns:
`get_user`, `list_repositories`, `create_issue`. Use the standard CRUD
vocabulary (`get`, `list`, `create`, `update`, `delete`) instead of inventing
synonyms (`fetch_user`, `retrieve_user`). Stick to parameter names the agent has
seen a thousand times: `query`, `limit`, `offset`, `id`. Also, remember to be
consistent between tools. Don't ask for `project_id` in one tool and
`projectName` in another. If one tool calls it `project_id`, all of them should
follow suit.

If you need to do something against convention, have a *really* good reason. "I
think it reads better" is not a good enough reason.

### Human Readable Units

Yes, unix timestamps are a perfectly fine convention. However, almost no human
is going to tell their agent "do the thing at 1806559200." When the agent has to
create that timestamp to call your API, it has to do the math to convert from
human time. Same when your API returns a timestamp and the agent has to figure
out what it means. Generally, if you want to do math, use a calculator (not an
LLM).

Prefer human-readable formats. The `sleep` command is a good example: it accepts
`5m`, `6h`, `7d`, and most humans (and agents) can read those at a glance
without converting anything. Same idea for dates (`2026-05-24` beats a Unix
epoch), durations, file sizes, and anything else where the conventional machine
format requires a mental conversion.

The agent is reading and writing for humans most of the time. Format
accordingly.

## The Next Frontier of DevX

DevX was always important, but perhaps AgX will be even more so. More and more
developers will adopt AI, and their interactions with API surfaces will be less
direct, filtered through their agents instead.  

This is going to accelerate. Every month, a bigger share of API calls will come
from agents instead of humans typing code. Those agents work under constraints:
token budgets, quotas, context windows, and patterns absorbed from every API in
their training data. The tools that work well under those constraints will
dominate. The ones that don't will get ignored. 

If you're using agentic tools, pay attention to what they're built on. Your
weekly quota will thank you for picking the agent-friendly ones. If you're
building APIs, treat this as the early days of an agent-first future. The
developers who decide which APIs to call don't pick yours out of brand loyalty.
They pick the one their agent doesn't choke on.

An agent-first world is the next frontier. 
