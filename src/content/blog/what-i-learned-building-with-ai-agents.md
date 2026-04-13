---
title: "What I've Learned Building with AI Agents in 2026"
description: "The interesting problems in AI agents aren't about generating code. They're about runtime, data, permissions, and context. Notes from the trenches at JetBrains."
pubDate: 2026-04-13
tags: ["ai-agents", "architecture", "typescript"]
---

Everyone's talking about AI agents. Most of the conversation is about the wrong things.

The hype cycle right now is: "look, this agent wrote an entire app!" or "Claude Code just solved my LeetCode hard!" That's impressive, sure. But after spending months building agent infrastructure at JetBrains, I can tell you — the code generation part is the *easy* problem. The hard problems are everything that happens around it.

## The real bottleneck isn't intelligence — it's runtime

Here's something I didn't expect: the moment you give an agent the ability to not just generate code but actually *do things* — store data, call APIs, manage state — you discover that we have almost zero infrastructure for this.

Think about it. A human developer has:
- A database with schemas, migrations, and access control
- Environment variables and secrets management
- CI/CD pipelines with rollback
- Monitoring and alerting
- Authentication and authorization at every layer

An AI agent has... a context window and a prayer.

When we started thinking about what agents actually need at Kineto, we kept coming back to the same gap: there's no "operating system" for agents. No standardized way for an agent to say "I need to store this data, with this schema, and only allow these other agents to read it."

## The TypeScript observation

Here's a concrete architectural thing I've noticed. TypeScript's type system is accidentally perfect for agent-native data.

When you define a Zod schema or a TypeScript interface, you're essentially creating a machine-readable contract. Agents can parse these. They can validate against them. They can even *generate* them from natural language descriptions.

```typescript
// This isn't just a type — it's a contract agents can reason about
const BookClubSchema = z.object({
  members: z.array(z.object({
    name: z.string(),
    role: z.enum(['admin', 'member', 'guest']),
  })),
  currentBook: z.object({
    title: z.string(),
    votes: z.number().min(0),
  }),
});
```

But here's the non-obvious part: the schema alone isn't enough. An agent needs to know not just the *shape* of data, but the *rules*. What happens when `votes` exceeds a threshold? Who's allowed to modify `currentBook`? What events should be triggered when a new member joins?

This is where most frameworks fall apart. LangChain gives you chains. CrewAI gives you crews. But nobody gives you **data objects with embedded logic, permissions, and event hooks that agents can discover and interact with programmatically.**

## Permissions are the unsexy killer feature

Nobody wants to talk about permissions. It's not a demo-worthy feature. But it's the thing that makes the difference between a toy and a production system.

Consider: you have a personal finance agent. It can read your transactions, categorize spending, and suggest budgets. Great. Now your partner wants their agent to see shared expenses but not your individual spending. Now your accountant's agent needs read-only access to everything, but only during tax season.

This is not a prompt engineering problem. This is a systems engineering problem. You need:

1. **Scoped delegation tokens** — Agent A can grant Agent B access to specific fields, with an expiry
2. **Field-level access control** — Not just "can read" but "can read these specific properties"
3. **Audit trails** — Who accessed what, when, through which agent
4. **Revocability** — Instantly cut off an agent's access

We're basically reinventing RBAC and OAuth for a world where the "users" are AI models. And honestly? The patterns from the last 20 years of web development apply surprisingly well — they just need to be adapted for non-human actors.

## The MCP bet

One pattern I'm bullish on is MCP (Model Context Protocol). The idea is simple: instead of every agent framework inventing its own tool-calling convention, you standardize it. Agent connects to an MCP server, discovers available tools, and interacts through a common protocol.

It's like how REST APIs standardized web services. Or how LSP standardized IDE language support. Once you have a protocol, you get composability for free.

The interesting question is: what sits behind the MCP server? Right now, most implementations are thin wrappers around existing APIs. But I think the real value is in **structured data objects** that are MCP-native — objects that were *designed* to be discovered, queried, and mutated by agents, with schemas, logic, and permissions baked in.

## What I'm watching

Three things I'm paying attention to over the next 6 months:

1. **Schema languages for agents** — Will something emerge as the "SQL for agent data"? Or will everyone just use JSON Schema / Zod and call it a day?

2. **Multi-agent coordination** — What happens when two agents try to modify the same resource? We solved this for databases (transactions, MVCC). We haven't solved it for agent workflows.

3. **The IDE as agent interface** — This is a JetBrains bias, but I genuinely think the developer IDE is an underrated surface for agent interaction. Developers already live there. Agents should too.

## The uncomfortable truth

Here's the uncomfortable truth about AI agents in 2026: the models are good enough. They've been good enough for a while. What's missing is everything *around* the model — the runtime, the data layer, the permission system, the protocol.

It's not a model problem. It's an infrastructure problem.

And infrastructure problems are my favorite kind.

---

*If you're thinking about similar problems, I'd love to hear from you. Find me on [GitHub](https://github.com/BezzubovEgor) or [Telegram](https://t.me/ybezzubau).*
