---
name: post-pdf
description: Convert a blog post to PDF using pandoc
disable-model-invocation: true
argument-hint: [post-name]
---

# Convert Post to PDF

## Input

An optional argument specifying which post to convert. This can be:
- A filename (e.g., `2026-03-23-securing-ai-agents-on-production-servers.md`)
- A partial match on the title (e.g., `securing ai agents`)
- Nothing: infer from the current branch name (e.g., branch `post/securing-ai-agents` matches a post with that slug)

## Process

1. **Resolve the post:**
   - If an argument is given, find the matching file in `_posts/`
   - If no argument, check the current branch name. If it looks like `post/<slug>`, find the post in `_posts/` whose filename contains that slug.
   - If ambiguous, list available posts and ask the user to pick.

2. **Read the post file** from `_posts/`.

3. **Strip YAML frontmatter** (everything between the opening and closing `---`) before passing to pandoc.

4. **Convert to PDF** using the `mcp__claude_ai_pandoc__markdown_to_pdf` tool. Save the output to the project root as `<post-slug>.pdf`.

5. **Report the result** with the path to the generated PDF.

## Notes

- HTML components (`<div class="tldr">`, `<div class="callout">`, etc.) may not render perfectly in PDF. Mention this to the user if the post contains them.
