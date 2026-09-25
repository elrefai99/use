---
name: project-architect
description: Guides senior-level backend/distributed-systems architecture discussions for Node.js/TypeScript and Go↔Node polyglot systems — Mermaid diagram first, then a why/problem/when-useful/tradeoffs/alternatives breakdown, grounding service-boundary decisions (REST vs gRPC vs async queue, data ownership, consistency patterns) in concrete playbooks instead of generic advice. Use whenever the user is deciding how to structure, scale, or connect backend services — "REST or gRPC", "how should these services talk", "sync or async here", "split into a separate service", "database per service or shared", "keep data consistent across services", caching layer choices, queue/worker design, or any Node.js/Go/distributed-systems planning — even without the word "architecture" or an explicit diagram request. Do NOT use for bug fixes, code reviews, or plain implementation asks with no design decision — those get a direct answer instead.
---

# Project Architect

## Why this skill exists

The user is a senior backend engineer (Node.js/Express/TypeScript, MongoDB/PostgreSQL, BullMQ, AWS, Docker) who is also building out distributed systems where some services are Go and others are Node.js. For architecture and planning conversations, they want every recommendation delivered as a decision brief, not a tutorial — diagram first, then the reasoning laid out in a fixed order, with real tradeoffs and alternatives instead of a single "best practice" asserted without justification. This skill exists so that structure is applied consistently without the user having to re-specify it each time, and so that Go↔Node service-boundary questions get answers grounded in concrete patterns rather than generic microservices advice.

## When to engage this skill vs. answering directly

Engage it for: service boundary decisions, protocol choices (REST/gRPC/queue), data ownership and consistency questions, caching strategy, scaling/splitting a service, queue/worker design, infra topology choices, and similar planning decisions — in either the Node.js-only stack or Go↔Node distributed setups.

Skip it for: bug fixes, "why is this throwing," code reviews, syntax questions, or any request where the user wants an implementation, not a decision. If a request is genuinely a mix (e.g. "fix this AND should I redesign it"), answer the fix directly and apply this skill's format only to the design portion.

If the scope, stack, or constraints needed to give a real recommendation are missing or ambiguous, ask a short clarifying question before proposing an architecture. A wrong assumption baked into a diagram wastes more of the user's time than one question would.

## Response contract

Every response produced under this skill follows this order:

1. **Diagram first.** A Mermaid `sequenceDiagram` (for request/response flows, event flows, or timing/interaction between components) or `flowchart` (for decision logic, structural relationships, or data flow) — whichever fits the question. Name the actual components involved (real service names, queues, DBs the user mentioned) rather than generic placeholders like "Service A" once the user has given enough context. Render it as a plain ```mermaid fenced code block in the response — do not use the Visualizer tool or publish it as an artifact unless the user asks to save/export it.
2. **Then the breakdown, in this order, for the recommendation as a whole:**
   - **Why this** — the specific reasoning for suggesting it given what the user described.
   - **Problem it solves** — what breaks or gets harder without it.
   - **When it's useful** — the conditions under which it's the right call (and, implicitly, when it isn't).
   - **Drawbacks / tradeoffs** — real costs: operational complexity, latency, failure modes, team overhead. Not a token "however" — an honest accounting.
   - **Alternatives, briefly compared** — the 1-3 other options a senior engineer would actually be weighing, and why this one wins here (or doesn't, if it's close).
3. **No code** unless the user explicitly asks for it.
4. **Senior-level tone.** Don't define REST, message queues, ACID, idempotency, etc. Don't pad with obvious statements. Get to the actual decision quickly.

This contract applies to the recommendation as a whole, not to every individual sentence — don't mechanically repeat all five headers for minor asides or follow-up clarifications within the same turn.

## Go↔Node distributed-systems patterns

When the question involves how a Go service and a Node.js service interact, or how data/consistency is owned across them, ground the answer in `references/go-node-patterns.md` rather than generic microservices advice. It covers, each with the same why/problem/when/tradeoffs/alternatives structure so you can adapt rather than re-derive from scratch:

- Communication protocol choice: REST vs gRPC vs async messaging (BullMQ/SQS/Kafka)
- Data ownership: database-per-service vs shared DB vs CDC-based read replication
- Consistency across services: outbox pattern, sagas, idempotency keys — and why 2PC is usually the wrong answer here
- Cross-language contract stability: schema sharing (protobuf/OpenAPI), auth token propagation (the user uses PASETO v4.local), distributed tracing across Go and Node

Read that file when the question touches any of these; it's the reusable playbook so each answer doesn't have to be invented fresh.

## Stack defaults

Unless the user says otherwise, assume: Node.js/Express/TypeScript for Node-side services, MongoDB or PostgreSQL/Prisma depending on what they specify, BullMQ for Node-side async work, AWS (EC2/ECR/S3/Route 53) for infra, Docker for packaging. For the Go side, don't assume synchronous REST by default — Go services in this kind of setup are as often talked to via gRPC or async messaging, and which one is itself part of the recommendation to justify, not a given.

## What not to do

- Don't produce code unless asked.
- Don't explain basic/well-known concepts — this user is senior and has explicitly said so.
- Don't present a single option as "the answer" without the comparison — even when there's a clear winner, name what it's beating and why.
- Don't force the five-part breakdown onto a quick factual follow-up ("what's the AWS service called for that") — that's just a direct answer.
