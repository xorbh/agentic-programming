---
layout: post
title: "ORMs vs Raw SQL for Agentic Coding"
date: 2026-03-18
tags: [orm, sql, architecture, ai-agents]
---

<div class="tldr">
<strong>TL;DR:</strong> Traditional ORMs were built to compensate for human limitations that AI agents don't share. For agentic coding, go with raw SQL if you can enforce good conventions, or use Drizzle if you want type safety without the abstraction overhead. Drizzle is the closest thing to raw SQL that still gives you compile-time guardrails. More broadly, any abstraction that exists purely for human ergonomics is worth questioning in an agentic world.
</div>

## The Question

When building applications with AI coding agents, one of the first architectural decisions is how to handle database access. Should you use a full ORM, a typed query builder, or write raw SQL?

The answer matters more than you might think. It directly affects how reliably an agent can produce correct code.

## Evaluating the Options

### Raw SQL: Full Control, No Safety Net

SQL is the lingua franca of databases. AI agents know it extremely well from training data, and there is no abstraction layer to hallucinate.

- **No API surface drift.** ORM APIs change between versions. SQL is stable.
- **Transparent execution.** What you see is what runs. No hidden N+1 queries or unexpected JOINs.
- **Easy debugging.** When an agent writes a bad query, you can paste it into a DB client and see exactly what went wrong.

But the biggest problem is **no type safety**. An agent can write a query that references columns that don't exist, and you won't know until runtime. This is particularly dangerous in agentic workflows where code is being generated and executed rapidly.

- Schema drift between your TypeScript types and actual DB schema
- Repetitive boilerplate for mapping rows to objects
- No compile-time feedback loop for the agent to self-correct

### Traditional ORMs: Safety With Baggage

The killer feature is **type safety catching agent mistakes at compile time**. If an agent references a non-existent field, the build fails immediately rather than silently breaking at runtime.

- Schema as code provides a single source of truth that agents can read
- CRUD operations are one-liners, reducing opportunities for mistakes

But traditional ORMs like TypeORM come with baggage:

- Decorator-heavy syntax that agents get subtly wrong
- Large API surfaces with more room for errors
- Hidden behavior (lazy loading, cascades) that creates hard-to-diagnose bugs
- Complex queries where agents fight the ORM instead of just writing SQL

<div class="comparison-grid">
<div class="comparison-card comparison-pro">
<h4>Where Raw SQL Wins</h4>
<p>Complex analytical queries, database-specific features, performance-critical paths, anything where you need to see exactly what runs.</p>
</div>
<div class="comparison-card comparison-con">
<h4>Where Raw SQL Hurts</h4>
<p>Basic CRUD at scale, schema changes that silently break queries, any workflow where the agent needs fast automated feedback.</p>
</div>
<div class="comparison-card comparison-pro">
<h4>Where ORMs Win</h4>
<p>Type safety, schema-as-code, simple CRUD, preventing SQL injection structurally rather than by convention.</p>
</div>
<div class="comparison-card comparison-con">
<h4>Where ORMs Hurt</h4>
<p>API sprawl, implicit behaviors, version-specific quirks, complex queries that fight the abstraction.</p>
</div>
</div>

## A Brief History: Why ORMs Exist

The "impedance mismatch" between relational databases and object-oriented programming languages has been called the [Vietnam of Computer Science](https://blog.codinghorror.com/object-relational-mapping-is-the-vietnam-of-computer-science/). In the late 1990s and early 2000s, developers were writing enormous amounts of boilerplate: hand-rolling `ResultSet` mappings in Java, manually tracking dirty fields, building SQL strings by concatenating user input (yes, really). It was tedious, error-prone, and a constant source of security vulnerabilities.

ORMs emerged to solve several concrete problems:

1. **Boilerplate reduction.** Mapping rows to objects and back was repetitive grunt work. Hibernate (2001), ActiveRecord (2004), and their successors automated this entirely. A developer could define a class and get CRUD operations for free.

2. **SQL injection prevention.** By parameterizing queries behind the scenes, ORMs eliminated an entire class of security vulnerabilities that plagued hand-written SQL string concatenation.

3. **Database portability.** In the enterprise world of the 2000s, switching between Oracle, SQL Server, and PostgreSQL was a real concern. ORMs abstracted away dialect differences, letting teams write database-agnostic code.

4. **Developer productivity.** The Rails boom (2005-2010) proved that convention-over-configuration ORMs let small teams ship faster. ActiveRecord's `has_many :through` was more accessible than teaching every developer on the team to write correct JOIN queries.

5. **Maintaining mental context.** Developers could stay in one language and one paradigm. No switching between Python and SQL, no keeping two mental models in sync. Everything was objects.

### Why They Became Dominant

ORMs won because they optimized for the scarcest resource: **developer attention**. In a world where developers were expensive and in short supply, most applications had straightforward CRUD patterns, teams had mixed SQL proficiency, and shipping speed mattered more than query optimization, it made perfect sense to trade some runtime performance and query control for massive gains in productivity and code safety.

The rise of web frameworks cemented this. Django shipped with its ORM. Rails *was* ActiveRecord in many developers' minds. .NET had Entity Framework. Spring had Hibernate. If you used the framework, you used the ORM. It wasn't even a decision. It was the default.

### Do Those Reasons Still Hold?

Here's where it gets interesting. Let's revisit each original motivation through the lens of an AI agent writing the code:

| Original Reason | Still Valid? | Why / Why Not |
|---|---|---|
| **Boilerplate reduction** | Mostly no | Agents don't get bored or make typos in repetitive code. Generating 50 lines of row-mapping boilerplate is trivial for an LLM. |
| **SQL injection prevention** | Partially | Agents *can* use parameterized queries correctly, but they can also slip into string interpolation if the codebase doesn't enforce it. ORMs still provide a structural guarantee here. |
| **Database portability** | Rarely | This was always somewhat theoretical, and it matters even less now. Most projects pick one database and stay there. |
| **Developer productivity** | Differently | The "productivity" equation flips. An agent doesn't need a simpler API to go faster. It needs a *predictable* API with strong type feedback to go *correctly*. |
| **Maintaining mental context** | No | Agents have no cognitive fatigue from context-switching between SQL and application code. They are equally fluent in both. |

The fundamental shift: ORMs were designed to compensate for human limitations. Limited attention, limited memory, inconsistent SQL knowledge, susceptibility to repetitive-task errors. AI agents don't share those limitations. They don't get tired of writing boilerplate. They don't forget to parameterize a query because they're rushing before a deadline.

What agents *do* struggle with is **ambiguity in APIs**. A sprawling ORM with multiple ways to express the same query, implicit behaviors, and version-specific quirks creates more surface area for hallucination. The simpler and more explicit the interface, the more reliably an agent can use it.

## A New Option: Typed Query Builders

This is why the answer isn't simply "drop ORMs, go back to raw SQL." A newer generation of tools sits between full ORMs and raw SQL, keeping the type safety while shedding the complexity:

| Tool | Approach |
|---|---|
| **Prisma** | Declarative schema file, generated type-safe client, small API |
| **Drizzle** | SQL-like syntax with full TypeScript type safety |

These are the two dominant tools in the TypeScript database space today, and what's notable is that they arrived at similar designs from opposite directions. Their creators are explicit about why.

Prisma's philosophy is [developer-first](https://www.prisma.io/blog/prisma-orm-manifesto): *"We design tools with developers in mind, prioritizing usability, productivity, and empowering teams to build great products with ease."* Their approach is schema-first: you define your data model in a dedicated file and Prisma generates a strongly typed client from it. The API surface is deliberately small.

Drizzle comes from the opposite end. Their [core principle](https://orm.drizzle.team/docs/overview) is: *"If you know SQL, you know Drizzle."* Where traditional data frameworks force you to *"build projects around them and not with them,"* Drizzle positions itself as a library with zero dependencies that integrates into your workflow rather than dictating it. It guarantees one SQL query per operation, preventing hidden N+1 problems.

Prisma has also written about what they call ["The ORM Convergence"](https://www.prisma.io/blog/convergence), arguing that despite starting from opposite philosophies, tools like Prisma and Drizzle are arriving at the same architectural patterns: schema-as-code, type-safe query APIs, first-class relation support, and raw SQL escape hatches. Their take: *"Certain patterns aren't arbitrary preferences... they're optimal solutions that emerge"* when you tackle production-scale problems. This convergence itself is evidence that the typed query builder pattern isn't a fad. It's where the ecosystem is settling.

And the AI angle isn't lost on these teams. Prisma explicitly markets their tools as providing ["guardrails for both human and AI-written code"](https://www.prisma.io/blog/announcing-prisma-postgres-for-ai-coding-agents), noting that in the era of AI-accelerated development, *"developers still need to understand the code they're shipping"* because *"abstractions have leaks."* They've even built an [MCP server](https://www.prisma.io/docs/ai) so AI agents can provision databases and run queries directly.

To see why these tools matter for agentic coding in practice, let's walk through the same operations with each tool and compare them to raw SQL.

### Defining the Schema

With **Prisma**, the schema lives in its own file (`schema.prisma`). The agent reads this file to understand the data model, and Prisma generates a fully typed client from it:

```prisma
// schema.prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  body      String
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}
```

With **Drizzle**, the schema is TypeScript itself. The types are the schema:

```typescript
// schema.ts
import { pgTable, serial, text, boolean, integer, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: text('email').unique().notNull(),
  name: text('name'),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  body: text('body').notNull(),
  published: boolean('published').default(false).notNull(),
  authorId: integer('author_id').references(() => users.id).notNull(),
});
```

This distinction matters for agents. Prisma's schema is a single, purpose-built file: easy to locate and parse. Drizzle's schema is regular TypeScript: it lives alongside application code and the agent can inspect the types directly without a generation step.

### Basic CRUD

Here's the same "create a user and fetch their posts" flow in each approach:

**Prisma:**

```typescript
// Create a user
const user = await prisma.user.create({
  data: { email: 'ada@example.com', name: 'Ada' },
});

// Get user with posts
const userWithPosts = await prisma.user.findUnique({
  where: { id: user.id },
  include: { posts: true },
});
```

**Drizzle:**

```typescript
// Create a user
const [user] = await db.insert(users)
  .values({ email: 'ada@example.com', name: 'Ada' })
  .returning();

// Get user with posts (explicit join, no magic)
const userWithPosts = await db
  .select()
  .from(users)
  .leftJoin(posts, eq(posts.authorId, users.id))
  .where(eq(users.id, user.id));
```

**Raw SQL:**

```typescript
// Create a user
const { rows: [user] } = await pool.query(
  'INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *',
  ['ada@example.com', 'Ada']
);

// Get user with posts
const { rows } = await pool.query(
  'SELECT u.*, p.id as post_id, p.title, p.body FROM users u LEFT JOIN posts p ON p.author_id = u.id WHERE u.id = $1',
  [user.id]
);
```

Notice the tradeoffs. Prisma's `include: { posts: true }` is the most concise, but hides the JOIN. The agent can't see or control what SQL runs. Drizzle reads almost like SQL but with type checking at every step. Raw SQL is fully transparent but returns untyped `any` rows.

### Where Type Safety Catches Agent Mistakes

This is where it gets practical. Imagine an agent writes this Prisma code:

```typescript
// Agent tries to filter by a column that doesn't exist
const users = await prisma.user.findMany({
  where: { username: 'ada' },  // TypeScript error: 'username' doesn't exist on UserWhereInput
});
```

The compiler catches it instantly. The agent sees the error, reads the generated types, and corrects to `email`. The same mistake in Drizzle:

```typescript
const result = await db.select().from(users)
  .where(eq(users.username, 'ada'));  // Property 'username' does not exist on type
```

Same outcome: immediate feedback. But with raw SQL:

```typescript
const { rows } = await pool.query(
  'SELECT * FROM users WHERE username = $1',  // TypeScript is happy, it's just a string
  ['ada']
);
// Blows up at runtime: column "username" does not exist
```

TypeScript can't help here. The query is a string. It could say anything. The agent won't discover the mistake until the query actually runs, which might be deep into a multi-step workflow.

### Complex Queries: The Escape Hatch

ORMs shine for CRUD but struggle with complex queries. Both Prisma and Drizzle handle this gracefully by letting you drop to raw SQL when needed, while still preserving some type safety:

**Prisma's `$queryRaw`:**

```typescript
const activeAuthors = await prisma.$queryRaw<{ name: string; post_count: bigint }[]>`
  SELECT u.name, COUNT(p.id) as post_count
  FROM "User" u
  JOIN "Post" p ON p."authorId" = u.id
  WHERE p.published = true
  GROUP BY u.id
  HAVING COUNT(p.id) > 5
  ORDER BY post_count DESC
`;
```

**Drizzle's SQL template:**

```typescript
const activeAuthors = await db
  .select({
    name: users.name,
    postCount: sql<number>`count(${posts.id})`,
  })
  .from(users)
  .innerJoin(posts, eq(posts.authorId, users.id))
  .where(eq(posts.published, true))
  .groupBy(users.id)
  .having(sql`count(${posts.id}) > 5`)
  .orderBy(sql`count(${posts.id}) desc`);
```

Drizzle's approach is notable here: the query is complex, but column references like `users.name` and `posts.id` are still type-checked. An agent can't reference a non-existent column even in a complex aggregation. Prisma's raw query is just a tagged template where the type annotation is manual and not verified.

### What This Means for Agents

The examples above reveal a pattern. For agentic coding, the ideal tool:

1. **Has a small, consistent API.** Fewer methods to hallucinate. Prisma has roughly 10 query methods. Drizzle maps closely to SQL keywords. Compare this to TypeORM's hundreds of decorator and query options.
2. **Catches mistakes at compile time.** The agent gets feedback *before* anything runs. Both Prisma and Drizzle deliver this.
3. **Makes the schema easy to find and read.** Agents need to discover the data model quickly. A single `schema.prisma` file or a `schema.ts` export is far easier to locate than decorators scattered across dozens of entity files.
4. **Allows raw SQL when the abstraction gets in the way.** No tool covers every query pattern. The escape hatch prevents the agent from fighting the ORM.

The pattern that works best: **use the typed layer for basic CRUD, drop to raw SQL for complex queries.** Both Prisma (`$queryRaw`) and Drizzle (`sql` template) support this escape hatch.

## The Abstraction Tax

<p class="lead">This post has been about ORMs and SQL, but the underlying question is bigger than databases.</p>

Most software tools we rely on today exist because humans needed them. Word processors give us menus and toolbars so we don't have to remember formatting commands. Drag-and-drop form builders let us avoid writing HTML. Presentation software provides a canvas because laying out slides in code would be painfully slow for a human.

These intermediate layers share a common design philosophy: **hide the underlying system behind an interface optimized for human cognition.** They trade direct control for approachability. For decades, that tradeoff made sense.

But AI agents don't need approachability. They don't need menus, drag handles, or visual canvases. They work directly with the underlying representations (SQL, HTML, LaTeX, configuration-as-code) with zero cognitive overhead. An agent doesn't benefit from a form builder standing between it and raw HTML. It doesn't need PowerPoint to produce a slide deck when it can generate the markup directly. Every intermediate layer that exists to make things *easier for humans* becomes, for an agent, an unnecessary translation step. One more API to learn. One more surface for hallucination. One more place where intent gets lost.

This doesn't mean every abstraction is wasteful. Abstractions that provide **structural guarantees** (type safety, schema validation, injection prevention) still earn their keep because they catch mistakes regardless of who made them. The ones that are purely about human ergonomics are the ones worth questioning.

<div class="callout">
<strong>The litmus test:</strong> Does this abstraction exist to make the work <em>easier to think about</em>, or to make the work <em>harder to get wrong</em>? Agents don't need the first. They very much need the second.
</div>

## My Recommendation

If you're starting a project where AI agents will be writing database code, I'd go one of two ways:

**Use raw SQL** if your team is comfortable enforcing parameterized queries by convention and your project is small enough that the lack of compile-time schema validation won't bite you. Agents are fluent in SQL. There is nothing to hallucinate, nothing to misremember. It's the most direct path between intent and execution.

**Use [Drizzle](https://orm.drizzle.team/)** if you want the safety net. Of all the tools in this space, Drizzle sits closest to raw SQL while still giving you full TypeScript type safety. Its query syntax mirrors SQL keywords, so an agent that knows SQL already knows Drizzle. There is no code generation step, no schema DSL to learn, no hidden queries. It's ~7.4kb with zero dependencies, which makes it practical for serverless. And when the typed API gets in the way, you drop to raw SQL with `sql` template literals that still type-check your column references.

Prisma is a solid tool with a mature ecosystem, but for agentic coding specifically, its schema-first code generation approach and its own query language add a layer of indirection that agents don't need. Drizzle's SQL-native design means the agent is always thinking in SQL, whether it's using the typed API or the escape hatch.

<div class="callout">
<strong>The short version:</strong> Write raw SQL if you can. If you want type safety (and you probably should), reach for Drizzle. It's the thinnest useful layer between your agent and the database.
</div>

And if you zoom out past databases: any tool that sits between an agent and the underlying system *solely* to make things friendlier for humans is a tool worth re-evaluating. The age of abstractions-as-convenience may be ending. The age of abstractions-as-guardrails is just beginning.
