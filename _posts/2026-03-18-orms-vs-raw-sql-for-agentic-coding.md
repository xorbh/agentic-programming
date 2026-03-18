---
layout: post
title: "ORMs vs Raw SQL for Agentic Coding"
date: 2026-03-18
tags: [orm, sql, architecture, ai-agents]
---

When building applications with AI coding agents, one of the first architectural decisions is how to handle database access. Should you use a full ORM, a typed query builder, or write raw SQL? The answer matters more than you'd think — it directly affects how reliably an agent can produce correct code.

## The Case for Raw SQL

SQL is the lingua franca of databases. AI agents know it extremely well from training data, and there's no abstraction layer to hallucinate.

- **No API surface drift** — ORM APIs change between versions; SQL is stable
- **Transparent execution** — what you see is what runs. No hidden N+1 queries or unexpected JOINs
- **Easy debugging** — when an agent writes a bad query, you can paste it into a DB client and see exactly what's wrong

## The Case Against Raw SQL

The biggest problem: **no type safety**. An agent can write a query that references columns that don't exist, and you won't know until runtime. This is particularly dangerous in agentic workflows where code is being generated and executed rapidly.

- Schema drift between your TypeScript types and actual DB schema
- Repetitive boilerplate for mapping rows to objects
- No compile-time feedback loop for the agent to self-correct

## The Case for ORMs

The killer feature is **type safety catching agent mistakes at compile time**. If an agent references a non-existent field, the build fails immediately rather than silently breaking at runtime.

- Schema as code provides a single source of truth that agents can read
- CRUD operations are one-liners, reducing opportunities for mistakes

But traditional ORMs like TypeORM come with baggage:

- Decorator-heavy syntax that agents get subtly wrong
- Large API surfaces with more room for errors
- Hidden behavior (lazy loading, cascades) that creates hard-to-diagnose bugs
- Complex queries where agents fight the ORM instead of just writing SQL

## The Sweet Spot: Typed Query Builders

The best option for agentic coding sits between full ORMs and raw SQL:

| Tool | Approach |
|---|---|
| **Prisma** | Declarative schema file, generated type-safe client, small API |
| **Drizzle** | SQL-like syntax with full TypeScript type safety |
| **Kysely** | Type-safe SQL query builder, closest to raw SQL |

These tools give you the fast feedback loop of type safety (agent writes wrong code → compiler catches it → agent fixes it) without the sprawling API surface of a full ORM.

The pattern that works best: **use the typed layer for basic CRUD, drop to raw SQL for complex queries**. Both Prisma (`$queryRaw`) and Drizzle (`sql` template) support this escape hatch.

## Bottom Line

**Typed query builder > Full ORM > Raw SQL** for agentic coding reliability.

The key insight is that type safety creates a feedback loop. Without it, bad queries silently ship. With it, the agent gets immediate signal that something is wrong and can self-correct — which is exactly how you want agentic workflows to behave.
