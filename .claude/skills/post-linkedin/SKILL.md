---
name: post-linkedin
description: Generate a LinkedIn post for an existing blog post, saved to social/
disable-model-invocation: true
argument-hint: [post-name]
---

# Generate LinkedIn Post

## Input

An optional argument specifying which post to promote. This can be:
- A filename (e.g., `2026-03-23-securing-ai-agents-on-production-servers.md`)
- A partial match on the title (e.g., `securing ai agents`)
- Nothing: infer from the current branch name

## Process

1. **Resolve the post** using the same logic: check argument, then branch name, then ask.

2. **Read the full blog post** from `_posts/`.

3. **Read the existing example** at `social/linkedin-securing-ai-agents.md` as a reference for tone and format.

4. **Construct the blog post URL:**
   - Base: `https://xorbh.github.io/agentic-programming`
   - Pattern: `/:year/:month/:day/:title/`
   - Example: `https://xorbh.github.io/agentic-programming/2026/03/23/securing-ai-agents-on-production-servers/`

5. **Write the LinkedIn post** following these rules:

   ### Tone and style (critical)

   - Follow the same voice as CLAUDE.md: conversational, direct, opinionated, honest.
   - Do NOT write with "LinkedIn energy." No short punchy one-liners stacked vertically. No listicle format. No "Here's what I learned:" followed by numbered takeaways. No "Let that sink in." None of that.
   - Write in connected prose. Use longer sentences that flow into each other. Think of it as a condensed version of the blog post's argument, not a highlight reel.
   - No emojis. No hashtags unless the user specifically asks.
   - No hype. No "game-changer", "mind-blowing", "you need to read this." Present the idea and let it stand.
   - No em dashes.

   ### Content

   - Distill the blog post's core argument into 3-5 paragraphs of connected prose.
   - Lead with the most interesting or counterintuitive insight, not a question or hook.
   - Include enough substance that the LinkedIn post is valuable on its own, not just a teaser.
   - End with the link to the full post. Frame it naturally (e.g., "Full writeup with configs and examples: [url]").

   ### Length

   150-300 words. LinkedIn truncates after roughly 210 characters, so make the first sentence count.

6. **Save the file** to `social/linkedin-<post-slug>.md` where `<post-slug>` is derived from the post filename.
   - If the file already exists, ask the user before overwriting.

7. **Show the generated content** to the user for review.
