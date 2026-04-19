---
title: "Building AI Agents That Don't Fall Apart: A Year in the Async Runtime Trenches"
description: "The industry obsesses over which model to use. The actual engineering challenge — async tool dispatch, cancellation, state correlation — is where agents succeed or fail in production."
pubDate: 2026-04-15
author: "Yahor Bezzubau"
tags: ["ai", "agents", "architecture", "llm", "runtime"]
---

Our AI agent was hallucinating tool call IDs at a 5% rate, and it took us three weeks to figure out why. The model was fine. Prompts were fine. The issue was a 36-character UUID that the model couldn't reliably hold in working memory across async turns. We replaced it with sequential integers. Problem solved.

That bug was one of maybe a dozen moments over the past year where I realized: **the runtime is the hard part.** Not the model, not the prompts, not the tool definitions. The execution layer — the thing that dispatches tool calls, tracks their lifecycle, handles results arriving out of order, and deals with cancellation — is where AI agents actually succeed or fail in production.

I've been building this layer full-time: the async runtime that sits between the LLM and the outside world. The industry mostly talks about model capabilities, reasoning benchmarks, prompt engineering. What actually trips you up is concurrency, state management, and observability. Here's what I learned.

## The runtime is the event loop

The mental model that finally clicked: the **runtime is the event loop**, and the **LLM is a coroutine**. The model yields tool calls like a generator yields values. The runtime dispatches them, manages their lifecycle, and resumes the model when results arrive.

```typescript
// 🚫 Synchronous: blocks until ALL tools complete
while (!done) {
  const { text, toolCalls } = await model.generate(context);
  const results = await runtime.dispatch(toolCalls); // model sits idle here
  context.append(results);
}

// ✅ Async: model keeps working while tools execute
while (!done) {
  const { text, toolCalls } = await model.generate(context);
  runtime.dispatchAsync(toolCalls); // fire and continue
  context.appendPending(toolCalls.map(tc => tc.id));
  // results injected into context as they arrive
}
```

That synchronous `await` is where most agent frameworks stop. The model calls a tool, the runtime executes it, everything blocks until the result comes back. Clean, correct, and completely inadequate once your tools do anything interesting.

A web search takes 2 seconds. A code sandbox takes 10. A file upload takes... who knows. The model sits idle while users stare at a spinner. After switching to async dispatch, agents that ran multiple tools per turn went from sequential latency (sum of all tool times) to parallel latency (max of all tool times). For a typical multi-tool query, that's the difference between 12 seconds and 3.

Going async fixes the latency problem. It also creates four new ones.

## Problem 1: State correlation (or, why UUIDs break)

In async execution, the model fires off a tool call, gets back an ID, and keeps working. When the result arrives later, the model correlates by ID. This means the model has to *remember* an identifier across multiple turns of conversation.

We used UUIDs. `f47ac10b-58cc-4372-a567-0e02b2c3d479`. Standard engineering practice. The model couldn't reproduce them — it would get close, off by one hex digit, and the runtime would silently drop the correlation. 5% of async flows failed this way.

```typescript
// Fragile: model hallucinates hex characters
{ tool_call_id: "f47ac10b-58cc-4372-a567-0e02b2c3d479" }

// Robust: model can count
{ tool_call_id: 3 }
```

Incremental integers fixed it instantly. The broader lesson: **design your agent protocols for how models actually manage state** — structure and sequence, not arbitrary token recall. Every interface between the runtime and the model is a potential hallucination surface. Keep those surfaces small.

## Problem 2: What to do while you wait

When the model needs the result of tool call #2 but it hasn't arrived yet, you have a choice. Block and wait? Or let the model keep going and correct when the real data arrives?

We went with optimistic execution. The model proceeds with its best guess, and when the real result lands, it adjusts. The correction loop is surprisingly robust. In practice, we inject a system marker when a result arrives that may contradict earlier reasoning: `[Note: tool #2 returned X — update any prior assumptions accordingly]`. It's not elegant, but models respond well to explicit correction cues.

**The cost of waiting is almost always higher than the cost of occasional correction.** A model that moves forward feels fast. A model that blocks feels broken.

The caveat: optimistic execution is wrong for irreversible operations. The runtime needs to know the difference:

```typescript
const tools = {
  web_search:  { handler: search, reversible: true  },  // optimistic OK
  send_email:  { handler: send,   reversible: false },  // must block
  db_write:    { handler: write,  reversible: false },  // must block
};
```

For reversible tools, dispatch and continue. For irreversible ones, the model explicitly waits. Don't learn this the hard way.

## Problem 3: Cancellation (and a race condition)

Once you have multiple async operations in flight, the model routinely realizes one of them is no longer relevant. An earlier result made a later query pointless. The user changed their mind. You need a way to abort.

```typescript
{ tool: "cancel", args: { tool_call_id: 3, reason: "user changed query" } }
```

Without this, stale results pollute the context. With it, the model actively manages its own execution — drops dead-end paths, conserves context window, stays focused.

But there's a race condition nobody warns you about: the model emits `cancel(3)` while tool 3's result is already in flight. The runtime processes the cancel, marks it done, then the result arrives. Deliver it? Discard it? We went with discard-and-log — if the model decided to cancel, it's already moved on, and injecting an unexpected result causes more confusion than help. But you *must* log it. Silent discards are debugging nightmares.

## Problem 4: You can't debug what you can't see

This is where teams spend the most time and have the worst tooling. In a sync agent, you read the trace top to bottom. In an async agent, everything interleaves:

```
┌─ Turn 1 ─────────────────────────────────────────────────────────┐
│ Model: "I'll search for the docs and check the API status"       │
│ → dispatch tool #1: web_search("project X docs")     [PENDING]   │
│ → dispatch tool #2: http_get("api.example.com/status") [PENDING] │
│ Pending: [#1, #2]                                                │
├─ Turn 2 ─────────────────────────────────────────────────────────┤
│ ← tool #2 completed: { status: "degraded", latency: 2400ms }    │
│ Pending: [#1]                                                    │
│ Model: "API is degraded. Let me also check the status page..."   │
│ → dispatch tool #3: web_search("project X status page") [PENDING]│
│ Pending: [#1, #3]                                                │
├─ Turn 3 ─────────────────────────────────────────────────────────┤
│ ← tool #1 completed: { results: [...docs links...] }            │
│ Pending: [#3]                                                    │
│ Model: "Found the docs. The status page search is redundant now."│
│ → cancel tool #3: "docs already contain status info"             │
│ Pending: []                                                      │
│ ← tool #3 result arrived (DISCARDED — already cancelled)         │
│ Model: "Here's what I found: ..."                                │
└──────────────────────────────────────────────────────────────────┘
```

Every turn shows the full async state: what's pending, what just completed, what the model knew versus what was still in flight. You can see tool #2 returning before tool #1, tool #3 being dispatched and then cancelled, and the late-arriving result being discarded. This trace has all four problems I've described in a single three-turn interaction.

Without this view, debugging is like debugging a concurrent program with only `print()`. Technically possible. Practically maddening.

## Build the trace viewer first

I used to think building an agent meant starting with the prompt chain and tool definitions. I was wrong.

The first thing I'd build in any new agent system is the async trace viewer. Before the tools, before the prompts, before the model integration. You can iterate on prompts in minutes. You can swap models in an afternoon. But you cannot fix what you cannot see, and in an async agent, you can't see anything by default.

Agents are not chatbots with extra steps. They're concurrent systems that happen to use an LLM as their scheduler. The model is the part that gets all the attention. The runtime is the part that determines whether it actually works.
