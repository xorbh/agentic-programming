Sharing a document should be three steps: write it, get a link, send it.

Google Docs turns that into: write it, fight the formatting toolbar, click Share, navigate the permissions dialog, hope the recipient has a Google account, wait three seconds for it to load. All for a few paragraphs of text.

Google Docs is a real-time collaborative editor with revision history, commenting, suggested edits, and a permissions model complex enough to need its own documentation. That's the right tool for a 50-page RFC with six contributors. It's the wrong tool for meeting notes or a project update.

So I built Slamit. Write markdown, publish, share a link. That's the whole product.

The interesting part: it has an MCP server, so Claude can publish and manage documents directly from a conversation. Draft something, publish it, get a shareable link. No context switching. No copy-pasting into another tool.

Under the hood: Express API on Fly.io, single-table DynamoDB, S3 for document storage, pre-rendered HTML for sub-second page loads. Share links with configurable expiration and view tracking. Eight API endpoints total. Costs cents a month.

This is the same pattern from my last post about PowerPoint and Word. The tools we inherited were designed for humans operating GUIs. That visual complexity becomes a liability when you want to automate or integrate. Markdown wins not because it's better for humans, but because it's composable. Plain text that tools can read, write, transform, and pipe.

Full post with architecture details, the DynamoDB single-table design, and why the dual-item pattern for share links avoids table scans: https://xorbh.github.io/agentic-programming/2026/03/26/google-docs-is-overkill-build-a-markdown-publisher/
