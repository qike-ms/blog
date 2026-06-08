---
date: 2026-06-08
type: blogpost
status: ready-to-publish
tags: [agents, tools, workflow, building, llm, poc, iterative-design]
title: Design Through Iterative POCs (For Small Tools, Anyway)
---

# Design through iterative POCs

How I turn ideas into tools I actually use. Cut the lengthy upfront design debates. Start dogfooding the POC.

## The old loop

I used to spend days talking to agents before any code got written. I'd draft a design doc. I'd argue about the interface. I'd insist on TDD. I'd build out a comprehensive eval set so the agent could run in a loop for hours or days, and hand me back a perfect, beautiful piece of software at the end.

I never got the perfect, beautiful piece of software. Not once.

What I always got was something that compiled, passed every test I asked agents to write, and missed three things I only noticed the first time I tried to use it for real. The design doc was wrong because I was guessing what I'd need. The eval set was wrong because I was guessing what success looked like. The agent did what I asked, mostly. Sometimes it took shortcuts I'd never have spotted from reading the diff. Either way, the cracks only showed when rubber hit the road.

The amount of agent time I burned writing the wrong thing very correctly was enormous.

## The new loop

Now I do this. Idea lands. I capture it. Then I ask the agent to build the smallest possible proof-of-concept that does the thing end-to-end. A few scripts. A config file if it earns its keep. Tests for the behavior I actually care about, because tests are cheap and they catch regressions. What I leave out is *excess abstraction*: no premature interfaces, no plugin systems, no "what if we want to swap this out later," no factories built to make a future I haven't seen easier. My rule of thumb is to wait for three or more real occurrences of a pattern before pulling it into an abstraction. Two looks like a coincidence. Three is a shape. And AI agents refactor fast enough now that waiting costs almost nothing. When the third occurrence shows up, the agent collapses all three into the right abstraction in one pass. Working, opinionated, concrete.

Then I use it. Every time it falls short, I tell the agent: "this didn't work because X." The agent fixes the bug or implements the feature. Then I dogfood it again.

After about ten real uses, I know what the thing actually wants to be. *Then* I let the agent harden it. The interface settles, because by now I know what I actually reach for. The design doc, when it gets written at all, gets written after the design is already in the code.

It's the same artifact at the end. The path is reversed.

## Where this works and where I'm not sure yet

This pattern works well so far for small projects and local features. For these, ten uses gives you the whole spec. The blast radius is small. The user (me) can describe the gap in one sentence. Rollback is free.

**I do not yet know if this works for large systems.** A distributed scheduler, a multi-tenant API, a billing pipeline. These have invariants you can't safely discover through "ship and patch." If the POC corrupts data on use seven, you don't get to iterate. There are real people downstream. The patches are not cheap.

So treat this post as a claim about small surface area. The agent-as-pair model lets you skip a lot of upfront design time. Whether it scales to large systems, I genuinely don't know. We will see. Right now I'd still write a design doc for anything where the failure mode includes the words "data loss" or "production outage."

## A concrete arc

The clearest example I have is a multi-agent dispatcher I built, a tool that routes chat messages to different coding agents and shares context between them. It went through roughly two dozen iterations over about three weeks. Here's the shape of the arc:

- **Day 1 (initial scaffold).** One topic, one backend, no concurrency story, no resets, no context sharing. It worked for me, on one chat, badly. The commit message was literally "initial scaffold." That was the POC.
- **Day 1, six hours later.** I asked two reviewer agents to look at it. They flagged scope gaps. Backends weren't stateless. Context wasn't shared between them. "This is going to bite you." I documented the gaps rather than fixing them, because I wanted to confirm they actually mattered through use before paying to fix them.
- **A few days in.** The first big rewrite went in. Backends became stateless. Switching between agents mid-conversation preserved context. This happened *after* I'd used the quick version enough to know the reviewers were right. If I'd fixed it on Day 1, I'd have built the wrong abstraction. By the time I rewrote it I knew the shape.
- **Same day, multiple ships.** A metrics endpoint, a rewind command, a per-topic system-prompt override. These were features I only realized I needed by using the thing for two days.
- **The week after.** Tool passthrough, a native-tools bug fix in one of the backend integrations, a plugin adapter for another, a safe compactor wiring pass. Each one was a real bug or gap I hit, not a hypothetical I planned for.
- **About two weeks in.** The project's developer guide grew a numbered list of safety invariants, over a dozen of them, each one a regression I caught and didn't want to re-litigate. That list is the design doc, and it got written *after* the system existed, because that's when I knew what the invariants needed to be.

If I'd started with a spec, I'd have predicted maybe a third of those invariants. The rest are things I only learned by shipping POC 1 and breaking it many different ways.

## What makes this possible

A few things have to be in place.

The first is playbook as durable memory. When the agent fixes a bug, it doesn't just patch the code. It adds the pitfall to a playbook file. Next time I or any other agent I'm running hits the same problem, the playbook already warns about it. The tool gets sharper by remembering, not by being rewritten. Without this, every agent starts from zero every session, and the POC loop never compounds.

The second is low capture friction. If I had to think about *where* to put an idea or *how* to start the POC, I wouldn't capture half of them. Slash commands and templated scaffolds move the capture cost below the "I'll do it later" threshold.

The third is multi-agent review. Every change benefits from a second and third coding agent (different vendor/model/harness) looking at it and discussing among themselves until they converge. The reviews catch what a single agent alone can't: security holes, concurrency bugs, edge cases, the invariants I haven't tripped over yet.

## Anti-patterns I keep wanting to fall back into

Habits the old loop trained into me, that I have to actively suppress.

I want to gold-plate POC 1. The urge to abstract before I've seen the pattern three times is strong. This has been built into my DNA through many years of software engineering work. Resist. Hardcode the values. Inline the helper. One user (me) is fine. Wait for three occurrences. Then ask the agent to refactor. That part is cheap now in a way it wasn't a year ago, and the agent collapsing three concrete instances into one good abstraction beats me predicting the abstraction from one instance.

I want to build a comprehensive eval set before any code. Tests for the behavior I already know matters are cheap and worth writing on day one. The thing I have to suppress is the dream of an exhaustive eval set built upfront, the one that lets the agent run autonomously for a week and hand me back perfect software. That eval set has the same blind spots as the design doc. I'm guessing what success looks like before I've seen the thing run. Write tests for what you actually know; let usage tell you what else to test.

## What's actually different

The biggest shift isn't technical. It's psychological. I used to feel like I had to *deserve* a tool before I built it. Justify the time. Picture the users. Defend the scope. Now I just ship POC 1 the moment the idea lands, because ten minutes of agent time is cheap and I'll know within a day whether the tool is worth keeping.

Most ideas die after POC 1, or get absorbed into an existing tool. That's fine, that's the loop working. The ones that survive get used, and the agent makes them better while I'm using them. Over time the tools I rely on every day are the ones I built without thinking about it.

And here's the part I didn't expect when I started working this way: those accumulated POCs *become* the MVP. There's no separate "now we build the real version" milestone. POC 1 is unusable to anyone but me. POC 3 has a config file and tests. Somewhere around POC 8 or 9 I notice I'd be comfortable handing it to another person, and *that* is the MVP. By POC 10 it has the right abstractions, because by then I've seen the pattern three times and the agent has refactored it cleanly. Same artifact, just kept iterating.

The agent didn't make me a better designer. It made design cheaper.

For small tools. For now. We will see how far this scales.
