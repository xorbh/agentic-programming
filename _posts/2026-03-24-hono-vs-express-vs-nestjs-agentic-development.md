---
layout: post
title: "Hono vs Express vs NestJS for Agentic Development"
date: 2026-03-24
tags: [ai-agents, node-js, hono, express, nestjs, api-frameworks]
---

<div class="tldr"><strong>TL;DR:</strong> Hono and Express are both solid for agentic development. The gap between them is small: Hono has better streaming and type-safe context, Express has a bigger ecosystem and more training data. The real outlier is NestJS, which costs 3-4x more agent turns per feature due to file count, module wiring, and abstraction choice overload (Guard vs Interceptor vs Middleware vs Pipe).</div>

<p class="lead">We built the same agentic API in all three frameworks, evolving it through six stages from a basic chat endpoint to a production system with auth, RAG, and multi-step tool orchestration. Here's what we observed.</p>

## The honest framing

This isn't really a three-way race. Hono and Express are both minimal frameworks that stay out of the agent's way. The differences between them are ergonomic, not architectural. NestJS is the outlier. It trades simplicity for enforced structure, and that trade-off changes the math when AI agents are writing the code.

We'll walk through six stages of evolution and track what matters for agentic development: files touched, agent failure modes, and context consumed.

---

## Stage 1: Bootstrap. "Build me a chat agent with tool calling"

### Hono

```typescript
// ~15 lines to a working streaming agent endpoint
import { Hono } from 'hono'
import { stream } from 'hono/streaming'
import Anthropic from '@anthropic-ai/sdk'

const app = new Hono()
const client = new Anthropic()

app.post('/chat', async (c) => {
  const { messages } = await c.req.json()
  const response = await client.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages,
    stream: true,
  })
  return stream(c, async (stream) => {
    for await (const event of response) {
      await stream.write(JSON.stringify(event) + '\n')
    }
  })
})

export default app
```

- Agent generates this in one shot, no errors
- Zero config files, no tsconfig fuss, works with Bun or Node

### Express

```typescript
import express from 'express'
import Anthropic from '@anthropic-ai/sdk'

const app = express()
app.use(express.json())
const client = new Anthropic()

app.post('/chat', async (req, res) => {
  const { messages } = req.body
  const response = await client.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages,
    stream: true,
  })
  res.setHeader('Content-Type', 'text/event-stream')
  for await (const event of response) {
    res.write(JSON.stringify(event) + '\n')
  }
  res.end()
})

app.listen(3000)
```

- Also one-shot, slightly more boilerplate (headers, `express.json()`, `listen()`)
- Streaming requires manually setting `Content-Type` and calling `res.write()`/`res.end()`. Not hard, but more to remember.
- Agent needs to remember async error handling isn't built in (Express 4)

### NestJS

```plaintext
src/
  app.module.ts
  chat/
    chat.module.ts
    chat.controller.ts
    chat.service.ts
    dto/
      chat-request.dto.ts
```

```typescript
// chat.controller.ts
@Controller('chat')
export class ChatController {
  constructor(private chatService: ChatService) {}

  @Post()
  @Sse()
  async chat(@Body() dto: ChatRequestDto): Observable<MessageEvent> {
    return this.chatService.streamChat(dto.messages)
  }
}

// chat.service.ts
@Injectable()
export class ChatService {
  private client = new Anthropic()

  streamChat(messages: Message[]): Observable<MessageEvent> {
    return new Observable((subscriber) => {
      // ... wrap async iterator into Observable
    })
  }
}

// chat.module.ts
@Module({
  controllers: [ChatController],
  providers: [ChatService],
})
export class ChatModule {}

// app.module.ts (also needs updating)
// dto (also needs creating)
// main.ts (also needs SSE configuration)
```

- Agent needs to generate/modify **6 files** minimum
- SSE in NestJS requires wrapping into RxJS Observables. This is non-trivial.
- High chance the agent gets the Observable/SSE wiring wrong on first try
- `nest new` scaffolding generates files the agent then needs to understand

**Stage 1 verdict:** Hono and Express are nearly identical in effort. Hono's streaming is slightly cleaner, but this is a wash. The real story is NestJS costing 3-4x more agent turns with more failure modes.

---

## Stage 2: Add Tools. "Add a database lookup tool and a web search tool"

The agent now needs to:

1. Define tool schemas
2. Handle the tool_use / tool_result loop
3. Add the actual tool implementations

### Hono

```typescript
// tools.ts (agent creates one file)
export const tools = [
  {
    name: 'search_db',
    description: 'Search the database',
    input_schema: {
      type: 'object',
      properties: { query: { type: 'string' } },
      required: ['query'],
    },
  },
  {
    name: 'web_search',
    description: 'Search the web',
    input_schema: {
      type: 'object',
      properties: { query: { type: 'string' } },
      required: ['query'],
    },
  },
]

export async function executeTool(name: string, input: any) {
  switch (name) {
    case 'search_db':
      return await searchDb(input.query)
    case 'web_search':
      return await webSearch(input.query)
  }
}
```

- Agent touches 2 files: new `tools.ts`, modify chat route
- The tool-use loop is just code. No framework abstractions in the way.
- Easy for the agent to reason about: it's a while loop

### Express

- Nearly identical to Hono. Same 2 files, same pattern.
- No framework difference at this stage

### NestJS

```
src/
  tools/
    tools.module.ts
    tools.service.ts
    tools.registry.ts        # where tools are registered
    implementations/
      search-db.tool.ts      # each tool as a class
      web-search.tool.ts
  chat/
    chat.service.ts          # modified to use ToolsService
    chat.module.ts           # modified to import ToolsModule
```

- The "NestJS way" is to make each tool an `@Injectable()` class
- Agent needs to create a registry pattern, wire modules together
- 5-7 files touched
- The structure NestJS forces is actually reasonable at this stage. The question is whether an AI agent will generate this structure correctly without guidance.
- Common agent failure: forgetting to add `ToolsModule` to imports in `ChatModule`

**Stage 2 verdict:** Hono/Express still simpler. NestJS structure is defensible but the agent spends more turns on wiring than on logic.

---

## Stage 3: Add Memory. "Add conversation persistence with Redis and vector search for RAG"

Now we need:

- Redis for session/conversation storage
- Vector store (Pinecone/pgvector) for RAG
- Middleware to load context before each request

### Hono

```typescript
// middleware/memory.ts
export const memoryMiddleware = createMiddleware(async (c, next) => {
  const sessionId = c.req.header('x-session-id')
  c.set('history', await redis.get(sessionId))
  c.set('context', await vectorSearch(c.req.body))
  await next()
  // persist updated history after response
})

app.use('/chat', memoryMiddleware)
```

- Clean middleware composition
- Agent adds 2-3 files: redis client, vector store client, middleware
- Hono's `c.set()` / `c.get()` for passing data is type-safe and simple

### Express

```typescript
app.use('/chat', async (req, res, next) => {
  req.history = await redis.get(req.headers['x-session-id'])
  req.context = await vectorSearch(req.body)
  next()
})
```

- Works fine but `req.history` has no type safety without augmenting the Request type
- Agent often forgets to augment Express Request interface. This doesn't break the build immediately, but causes TS errors on the next edit when the agent tries to access `req.history` from a different file.
- Async middleware error handling is a footgun in Express 4. An unhandled rejection in middleware silently hangs the request. Hono catches async errors by default.

### NestJS

```typescript
// memory/memory.module.ts (new module)
// memory/redis.service.ts (@Injectable)
// memory/vector-store.service.ts (@Injectable)
// memory/memory.interceptor.ts (or Guard, or Middleware... agent has to choose)
// chat/chat.service.ts (inject MemoryService)

@Injectable()
export class MemoryService {
  constructor(
    private redis: RedisService,
    private vectorStore: VectorStoreService,
  ) {}
  // ...
}
```

- DI actually starts paying off here. Swapping Redis for Dragonfly, or Pinecone for pgvector, is a config change.
- But the agent now faces a design decision: Interceptor vs Guard vs Middleware vs Pipe? Each has different lifecycle timing. **This is where agents make bad choices.** They pick one, it doesn't work right, they refactor to another.
- 5-6 new files, 2-3 modified files

**Stage 3 verdict:** Hono is cleanest. Express works but accumulates type-safety debt. NestJS's DI starts showing value but agent decision fatigue increases.

---

## Stage 4: Multi-tool Agent with Auth & Rate Limiting. "Add Clerk auth, rate limiting per user, and 10 more tools"

### Hono

```
src/
  index.ts
  middleware/
    auth.ts          # Clerk verification
    rate-limit.ts    # per-user rate limiting
    memory.ts
  tools/
    index.ts         # registry + executor
    search-db.ts
    web-search.ts
    ... 10 more files
  lib/
    redis.ts
    vector-store.ts
    anthropic.ts
```

- Still flat and readable. Agent navigates easily.
- But: no enforced structure. If the agent is sloppy, tools might end up scattered
- Middleware stacking order matters and the agent needs to get it right:
  ```typescript
  app.use('/chat', auth, rateLimit, memory)
  ```
- **Key risk**: without conventions, the codebase starts feeling ad-hoc. The agent might put rate limiting logic inside the chat handler instead of as middleware.

### Express

- Almost identical structure to Hono
- Same risks, same benefits
- More middleware packages available (express-rate-limit, etc.). The agent might reach for these, which saves code.
- **Accumulated debt**: by now, async error handling gaps may cause unhandled promise rejections in production

### NestJS

```
src/
  auth/
    auth.module.ts
    auth.guard.ts
    clerk.strategy.ts
  rate-limiting/
    rate-limiting.module.ts
    rate-limiting.guard.ts
  tools/
    tools.module.ts
    tools.registry.ts
    implementations/
      ... 12 tool files, each a class
  chat/
    chat.module.ts
    chat.controller.ts
    chat.service.ts
  memory/
    memory.module.ts
    ...
```

- **NestJS shines here**: Guards for auth, Interceptors for rate limiting, clean module boundaries
- Adding a new tool is mechanical: create class, register in module. An agent can do this reliably because the pattern is repetitive.
- But: **30+ files** at this point. Agent context window is filling up with boilerplate.
- **Critical issue for agentic dev**: when the agent needs to make a cross-cutting change (e.g., "add logging to all tool executions"), it needs to understand the decorator/module graph. In Hono, it's just adding a line in the tool executor function.

**Stage 4 verdict:** NestJS's structure helps but its file count hurts agentic development. Hono needs discipline but stays nimble.

---

## Stage 5: Production Hardening. "Add error handling, observability, graceful shutdown, health checks"

### Hono

```typescript
// Error handling: one middleware
app.onError((err, c) => {
  logger.error(err)
  return c.json({ error: 'Internal error' }, 500)
})

// Health check: one route
app.get('/health', (c) => c.json({ status: 'ok' }))

// Observability: one middleware
app.use('*', opentelemetryMiddleware)
```

- Agent adds ~3 small changes. Done.
- Graceful shutdown needs manual signal handling (trivial)

### Express

- Same as Hono, but error middleware has the awkward `(err, req, res, next)` 4-arg signature
- Agent sometimes forgets the 4th arg, breaking error handling silently

### NestJS

```typescript
// Exception filter (global)
@Catch()
export class AllExceptionsFilter implements ExceptionFilter { ... }

// Health module (@nestjs/terminus)
@Controller('health')
export class HealthController {
  constructor(private health: HealthCheckService) {}
  @Get()
  check() { return this.health.check([...]) }
}

// Interceptor for observability
@Injectable()
export class LoggingInterceptor implements NestInterceptor { ... }

// Shutdown hooks
app.enableShutdownHooks()
```

- More structured, but each concern is a new file + class + registration
- `@nestjs/terminus` for health checks is nice but another dependency
- Agent needs to know about `APP_FILTER`, `APP_INTERCEPTOR` global bindings

**Stage 5 verdict:** All three are fine. Hono is least code. NestJS is most structured. Express is in between.

---

## Stage 6: The Reckoning. "Refactor the tool system to support async multi-step tools with human-in-the-loop approval"

This is where agentic development gets real. A major architectural change.

### Hono

- Agent reads the flat `tools/` directory, understands the executor in minutes
- Refactors the tool interface to support `{ status: 'pending_approval' }` states
- Adds a WebSocket endpoint for approval notifications
- Touches ~5-8 files, all of which the agent can hold in context simultaneously
- **Risk**: if the agent's earlier code was poorly structured, this refactor is harder

### Express

- Same as Hono in practice
- WebSocket addition via `ws` or `socket.io`. Well-documented, agent knows these.

### NestJS

- Agent needs to:
  - Create a new Gateway (NestJS's WebSocket abstraction)
  - Modify the tool registry
  - Add a new state machine service
  - Update DTOs
  - Wire new module dependencies
  - Possibly add a Bull queue for async tool execution
- Touches **12-15 files**
- **The DI graph becomes the bottleneck**: the agent may introduce circular dependencies, forget to export a provider, or misconfigure the Gateway module
- **But**: if it gets it right, the result is cleaner and more maintainable

**Stage 6 verdict:** Hono wins for agentic refactoring speed. NestJS's abstraction layers actively slow down AI-driven architectural changes.

---

## Summary: The Agentic Development Curve

```
Complexity
    |
    |                                    NestJS ---- structure pays off
    |                                   /            but agent slows down
    |                                  /
    |               Express -------------- debt accumulates
    |              /
    |    Hono -------------------------------- stays lean, needs discipline
    |   /
    |  /
    | /
    +----------------------------------------- Stages
      1         2         3         4         5         6
    Boot     Tools    Memory    Scale    Harden   Refactor
```

| Factor                        | Hono       | Express    | NestJS                                          |
| ----------------------------- | ---------- | ---------- | ----------------------------------------------- |
| Agent first-try success rate  | **High**   | **High**   | Medium                                          |
| Files per feature             | 1-2        | 1-2        | 4-6                                             |
| Agent context consumed        | Low        | Low        | **High**                                        |
| Refactoring ease (by agent)   | **Easy**   | **Easy**   | Hard                                            |
| Structure at scale            | You enforce it | You enforce it | **Framework enforces it**                  |
| Streaming DX                  | **Native** | Manual headers | Observable wrapping                          |
| Type-safe context             | **Built-in** (`c.set`/`c.get`) | Requires interface augmentation | DI handles it |
| Async error handling          | **Built-in** | Manual (Express 4) | **Built-in**                          |
| Training data / ecosystem     | Growing    | **Massive** | Large                                          |
| Agent decision fatigue        | Low        | Low        | **High** (Guard vs Interceptor vs Middleware vs Pipe) |

## What actually makes code agent-friendly

Before the bottom line, it's worth naming the pattern. Across all six stages, the same properties made code easier for agents to work with:

1. **Low file-to-feature ratio.** Every additional file is context the agent has to load, understand, and keep consistent. Hono and Express average 1-2 files per feature. NestJS averages 4-6.
2. **Fewer abstraction choices.** When there's one obvious way to do something, agents get it right on the first try. When there are four (Guard vs Interceptor vs Middleware vs Pipe), they pick one, it doesn't fit, they refactor.
3. **Async safety by default.** Agents don't reliably add error-handling wrappers. Frameworks that catch async errors automatically (Hono, NestJS) avoid silent failures that Express 4 allows.
4. **Flat, greppable structure.** Agents navigate by searching. A flat `tools/` directory with one file per tool is easier to search than a nested module tree with decorators and DI registration.

These aren't Hono-specific principles. They apply to any framework in any language. FastAPI scores well on the same criteria. Spring Boot scores poorly for the same reasons NestJS does.

## Bottom line

**Hono and Express are both good choices.** The gap between them is real but small. Hono gives you native streaming, type-safe context passing, and async error handling out of the box. Express gives you the largest middleware ecosystem and the most training data in every AI model. If you're starting fresh, Hono is the slight edge. If you're already on Express, there's no compelling reason to switch for agentic development alone.

**NestJS is the wrong choice for agentic development** unless you already have a large NestJS codebase. The framework's value proposition, enforced architecture, becomes a liability when AI agents are writing the code. They spend more turns on wiring than on logic. The abstraction layers increase failure modes. And the file count per feature eats into the agent's context budget at every stage.

<div class="callout"><strong>The real insight:</strong> The framework matters less than the properties it produces. Low file count, few abstraction choices, async safety by default, flat structure. Pick any framework that gives you these, in any language, and agents will work well with it.</div>
