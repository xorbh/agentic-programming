---
name: post-slack
description: Generate a Slack message for sharing a blog post, saved to social/
disable-model-invocation: true
argument-hint: [post-name]
---

# Generate Slack Share Message

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

4. **Write a Slack message** following these rules:

   ### Tone and style

   - Professional but not stiff. Appropriate for a team channel or an engineering Slack.
   - Follow CLAUDE.md tone: no hype, no emojis, no em dashes. Concise and direct.
   - Write as someone sharing something relevant with colleagues, not broadcasting.

   ### Format

   - Use Slack mrkdwn formatting: `*bold*` for the post title on the first line.
   - 1-3 sentences summarizing why it's relevant or interesting.
   - URL on its own line at the end.
   - Keep it scannable. People skim Slack.

   ### Length

   Under 80 words.

5. **Save the file** to `social/slack-<post-slug>.md`
   - If the file already exists, ask the user before overwriting.

6. **Show the generated content** to the user for review and easy copy-paste.
