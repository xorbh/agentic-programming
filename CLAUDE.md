# Agentic Programming Blog

A Jekyll blog hosted on GitHub Pages at https://xorbh.github.io/agentic-programming/

## Git Workflow

**Never push directly to main.** All new posts and content changes must go through a pull request:

1. Create a feature branch from main (e.g., `post/orm-vs-raw-sql`, `fix/layout-styling`)
2. Commit changes to the feature branch
3. Open a PR with a clear title and description
4. Merge to main after review

Direct pushes to main are only acceptable for urgent fixes to broken deployments.

## Writing Tone

- **Conversational and direct.** Write like you're explaining something to a peer over coffee. Not academic, not marketing.
- **No hype or anxiety.** Don't use fear-based framing ("you're falling behind", "the future is here"). Don't manufacture urgency. Present ideas and let them stand on their own.
- **Opinionated but honest.** Take a position and back it with reasoning. Acknowledge tradeoffs. Don't pretend there's only one right answer when there isn't.
- **No em dashes.** Use periods, commas, or restructure the sentence instead.
- **No emojis** unless explicitly requested.
- **Concise.** Prefer short, direct sentences. Cut filler words. If a paragraph can be one sentence, make it one sentence.

## Post Structure

Every post should follow this general flow:

1. **TL;DR** at the top (using the `<div class="tldr">` component)
2. **Frame the question.** What problem or decision is the reader facing?
3. **Evaluate the options.** Present each approach with pros and cons. Use code examples where relevant.
4. **Provide context.** History, why things are the way they are, what has changed.
5. **Introduce newer alternatives** if they exist. Reference official sources and link to them.
6. **State your recommendation.** Be clear about what you'd actually use and why.
7. **Zoom out** if there's a broader principle at play.

Not every post needs all of these, but the flow should move from specific question to practical answer to broader insight.

## Post Frontmatter

```yaml
---
layout: post
title: "Post Title Here"
date: YYYY-MM-DD
tags: [tag1, tag2, tag3]
---
```

Always include relevant tags. Use lowercase, hyphenated tags (e.g., `ai-agents`, `raw-sql`).

## Styled Components

The layout supports these HTML components for visual variety:

- **TL;DR box:** `<div class="tldr"><strong>TL;DR:</strong> ...</div>`
- **Callout box:** `<div class="callout"><strong>Label:</strong> ...</div>` (amber background)
- **Lead paragraph:** `<p class="lead">...</p>` (slightly larger text)
- **Comparison grid:** 2x2 grid using `comparison-grid`, `comparison-card`, `comparison-pro`, `comparison-con` classes

## Code Examples

- Always specify the language in fenced code blocks for syntax highlighting (Rouge/Monokai theme)
- When comparing tools or approaches, show the same operation in each so the reader can compare directly
- Include comments in code that highlight the key differences

## Referencing External Sources

- Link to official documentation and blog posts from tool creators
- Use direct quotes (in italics) when a tool's own philosophy is relevant
- Prefer primary sources over third-party comparison articles

## Technical Setup

- **Ruby:** 3.3.7 via rbenv (`.ruby-version` in project root)
- **Jekyll:** via `github-pages` gem
- **Local preview:** `make serve` → http://127.0.0.1:4973/agentic-programming/
- **Deployment:** GitHub Actions on push to main, deploys to GitHub Pages
- **Assets:** stored in `/assets/` (logo, favicons)
