---
name: new-post
description: Create a new blog post on a feature branch following CLAUDE.md conventions
disable-model-invocation: true
argument-hint: [topic]
---

# New Blog Post

## Input

The user may provide a topic as arguments. If not provided, ask for:

1. **Topic** (required): What the post is about.
2. **Direction/angle** (optional): The specific take or framing. If not provided, propose 2-3 angles and let the user choose.

## Process

1. **Create a feature branch** from main:
   - Branch name: `post/<slugified-topic>` (e.g., `post/testing-strategies-for-ai-agents`)
   - Run `git checkout main && git pull && git checkout -b post/<slug>`

2. **Read CLAUDE.md** before writing to ensure you follow the latest conventions.

3. **Read existing posts** in `_posts/` to match the voice and depth of previous writing.

4. **Write the post** as a new file in `_posts/`:
   - Filename: `YYYY-MM-DD-<slug>.md` using today's date
   - Include frontmatter with `layout: post`, `title`, `date`, and `tags`

5. **Follow CLAUDE.md conventions strictly:**
   - Start with a `<div class="tldr">` block
   - Follow the post structure: frame the question, evaluate options, provide context, introduce alternatives, state recommendation, zoom out
   - Writing tone: conversational and direct, no hype, no anxiety, no em dashes, no emojis, concise, opinionated but honest
   - Use styled components where appropriate (`callout`, `lead`, `comparison-grid`)
   - Specify language on all fenced code blocks
   - Link to official docs and primary sources
   - Use lowercase hyphenated tags

6. **Show the post to the user** for review before committing.

7. **Commit the post** to the feature branch with a message like: `Add post: <title>`

8. **Offer to open a PR** to main using `gh pr create`.

## Important

- Never push directly to main. Always use a feature branch.
- Do not create the PR unless the user confirms.
- The post should be substantial (1500+ words) unless the user requests something shorter.
