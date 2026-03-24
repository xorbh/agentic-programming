---
name: post-whatsapp
description: Generate a WhatsApp message for sharing a blog post, saved to social/
disable-model-invocation: true
argument-hint: [post-name]
---

# Generate WhatsApp Share Message

## Input

An optional argument specifying which post to share. This can be:
- A filename (e.g., `2026-03-23-securing-ai-agents-on-production-servers.md`)
- A partial match on the title (e.g., `securing ai agents`)
- Nothing: infer from the current branch name

## Process

1. **Resolve the post** using the same logic: check argument, then branch name, then ask.

2. **Read the full blog post** from `_posts/`.

3. **Construct the blog post URL:**
   - Base: `https://xorbh.github.io/agentic-programming`
   - Pattern: `/:year/:month/:day/:title/`

4. **Write a WhatsApp share message** following these rules:

   ### Tone and style

   - Casual and brief. This is something you'd send to a friend or a small group.
   - Follow CLAUDE.md tone: no hype, no emojis, no em dashes. Direct and honest.
   - Write as if you're telling someone "hey, I wrote something you might find interesting" rather than promoting content.

   ### Format

   - 2-4 sentences maximum. One short paragraph.
   - Summarize the core point of the post in plain language.
   - End with the URL on its own line so WhatsApp generates a link preview.
   - No formatting (no bold, italic, or markdown). Plain text only.

   ### Length

   Under 100 words.

5. **Save the file** to `social/whatsapp-<post-slug>.md`
   - If the file already exists, ask the user before overwriting.

6. **Show the generated content** to the user for review and easy copy-paste.
