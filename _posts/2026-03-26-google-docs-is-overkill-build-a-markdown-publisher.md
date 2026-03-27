---
layout: post
title: "Google Docs Is Overkill. Build a Markdown Publisher Instead."
date: 2026-03-26
tags: [tooling, markdown, ai-agents, mcp, architecture]
---

<div class="tldr">
<strong>TL;DR:</strong> Google Docs is a formatting tool disguised as a sharing tool. Most document sharing is simple: write something, get a link, send it. But Google Docs buries that under permission dialogs, formatting toolbars, and an API that requires OAuth gymnastics. Slamit is a markdown publishing platform that strips this down to the essentials: write markdown, publish, share a link. It has a REST API, MCP integration so Claude can publish documents directly, and expiring links with view tracking. The whole thing runs on DynamoDB, S3, and Fly.io for cents a month.
</div>

## The Problem

Here's what sharing a document with someone should look like: you write it, you get a link, you send the link. Three steps.

Here's what sharing a document via Google Docs looks like: you write it, you fight with the formatting toolbar, you click Share, you decide between "Restricted" and "Anyone with the link," you worry about whether the recipient has a Google account, you notice the doc takes three seconds to load, and you wonder why sharing a few paragraphs of text required a full-blown web application.

Google Docs solves a problem most people don't have. It's a real-time collaborative editor with revision history, commenting, suggested edits, and a permissions model complex enough to need its own documentation. That's great if you're writing a 50-page RFC with six contributors. It's absurd if you're sharing meeting notes or a project update.

The format itself is the issue. Google Docs stores content in a proprietary format that only renders inside Google's ecosystem. You can export to PDF or Word, but that's a lossy conversion. There's no clean way to version control it, diff it, or pipe it through a script. The API exists, but it requires OAuth2 setup and a [Docs API](https://developers.google.com/docs/api) that has hundreds of endpoints for manipulating document structure at the element level.

## What If You Just Used Markdown?

Markdown is the obvious answer for most document sharing. It's plain text, so it works with every tool in your stack. You can version control it, diff it, search it, template it. Every developer already knows it. Every modern platform renders it.

The missing piece isn't markdown itself. It's the sharing infrastructure. You can write markdown in any editor, but turning it into a shareable link with access control and view tracking requires plumbing that doesn't exist out of the box.

That's what [Slamit](https://slamit.in) does.

## Architecture

Slamit is a multi-tenant markdown publishing platform. The core flow is:

1. Write a document in markdown (web UI, API, or through Claude via MCP)
2. Publish it (markdown gets rendered to HTML, both stored in S3)
3. Share a link (auto-generated on publish, with configurable expiration)

Recipients see a clean, fast-loading rendered page. No sign-up required. No Google account needed. The link either works or it's expired.

```
          ┌──────────────┐          ┌─────────────────┐          ┌──────────────┐
          │              │          │                 │          │              │
          │  React SPA   │◄────────►│  Express API    │◄────────►│  DynamoDB    │
          │  (Vercel)    │  REST    │  (Fly.io)       │          │  (Single     │
          │              │          │                 │          │   Table)     │
          └──────────────┘          └────────┬────────┘          └──────────────┘
                                             │
          ┌──────────────┐                   │                   ┌──────────────┐
          │              │                   │                   │              │
          │  MCP Server  │◄──────────────────┤                   │  S3          │
          │  (Fly.io)    │  REST             └──────────────────►│  (Documents) │
          │  Python      │                                       │              │
          └──────────────┘                                       └──────────────┘
```

The stack is deliberately simple. Express API on Fly.io, DynamoDB for metadata, S3 for document storage, Clerk for auth, and a React SPA on Vercel for the dashboard. Everything lives in a single DynamoDB table using composite keys. The whole thing costs cents per month for a small team.

### Single Table Design

All metadata lives in one DynamoDB table with a composite key pattern:

```
TENANT#{orgId}  +  DOC#{docId}     → Document metadata
TENANT#{orgId}  +  LINK#{linkId}   → Link scoped to tenant
LINK#{linkId}   +  LINK#{linkId}   → Link lookup for public viewers
```

The dual-item pattern for links is worth calling out. Public viewers don't know the tenant ID. They just have a link ID. So each link gets two items: one scoped to the tenant (for listing all links in an org) and one standalone (for O(1) lookup when someone clicks a shared link). Two items, two access patterns, no table scans.

### Pre-Rendered HTML

When you publish a document, the markdown gets rendered to HTML via [marked](https://marked.js.org/) (with GFM support) and sanitized through DOMPurify. Both the source markdown and the rendered HTML get stored in S3. When someone opens a shared link, the API fetches the pre-rendered HTML directly. No rendering at request time. Sub-second page loads.

## Share Links With Expiration

Every document gets two share links on publish: a 24-hour quick share and a 30-day extended link. You can create additional links with custom expiration (1 hour, 7 days, 30 days, or permanent) or specify any duration in seconds via the API.

Links have built-in view tracking. Every time someone opens a shared link, the view count increments asynchronously. You can see exactly how many times each link was accessed.

Revocation is instant. Delete a link and it's gone. DynamoDB TTL handles cleanup of expired links automatically, with a defensive server-side check as a belt-and-suspenders measure.

## Claude Can Publish Documents

This is where it gets interesting for agentic workflows. Slamit has an MCP server that lets Claude publish and manage documents directly from a conversation.

The MCP server runs as a separate Python service (FastMCP with streamable-http transport) on Fly.io, configured to scale to zero when idle. It proxies requests to the main Express API using Clerk OAuth tokens that are encrypted and persisted in DynamoDB.

The available tools:

| Tool | What It Does |
|------|-------------|
| `publish_document` | Create a new document from markdown |
| `update_document` | Replace document content |
| `create_share_link` | Generate a new share link with optional expiry |
| `list_documents` | Show all documents in the org |
| `revoke_share_link` | Delete a share link |

This means you can ask Claude to draft a document, publish it, and give you a shareable link in a single conversation. No context switching. No copy-pasting into a separate tool.

## The API

Everything the dashboard does is available through the REST API with API key auth. Keys use a `mh_` prefix and 32-character nanoid, created through the dashboard and shown once in full.

```bash
# Publish a document
curl -X POST https://api.slamit.in/api/documents \
  -H "Authorization: Bearer mh_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"content": "# Meeting Notes\n\nDecisions made today..."}'

# Create a share link (expires in 1 hour)
curl -X POST https://api.slamit.in/api/documents/{docId}/links \
  -H "Authorization: Bearer mh_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{"expires_in": 3600}'
```

Eight endpoints total. Documents (CRUD), share links (create, list, revoke), and API keys (create, list, revoke). That's the entire API surface.

## Why Not Just Use Google Docs?

| | Google Docs | Slamit |
|---|------------|--------|
| **Format** | Proprietary WYSIWYG | Markdown (portable, diffable) |
| **Sharing** | Complex permissions UI | One link, optional expiry |
| **Recipient experience** | Needs Google account or clunky public link | No account, fast page load |
| **API** | OAuth2 + hundreds of endpoints | REST + API keys, 8 endpoints |
| **AI integration** | Limited | Native MCP for Claude |
| **View tracking** | None for external viewers | Built-in per link |
| **Link control** | Links live forever | Expiring links, instant revocation |
| **Load time** | 2-5 seconds | Pre-rendered HTML, sub-second |

Google Docs wins on real-time collaboration. If you need multiple people editing the same document simultaneously with comments and suggested edits, use Google Docs. That's what it's for.

But for the much more common case of "write something and share it," it's the wrong tool. You don't need a collaborative editor. You need a publishing pipeline.

## The Broader Pattern

This is the same pattern we keep seeing with agentic development. The tools we inherited from the pre-AI era were designed for humans operating GUIs. They have rich visual interfaces because they needed to. But that visual complexity becomes a liability when you want to automate or integrate.

Markdown is more useful than Google Docs not because it's better for humans (it arguably isn't, for non-technical users), but because it's composable. It's plain text that tools can read, write, transform, and pipe. An AI agent can generate markdown. A CI pipeline can publish it. A link with an expiration date can share it. No OAuth dance, no proprietary format, no three-second page loads.

The same logic applies to everything the [previous post on HTML primitives](/agentic-programming/2026/03/24/powerpoint-word-and-the-case-for-html-primitives.html) discussed. The tools that win in an agentic world are the ones built on open, composable formats. Not because openness is a virtue in itself, but because composability is what lets you build workflows that weren't anticipated by the original tool authors.

Write markdown. Publish. Share a link. That's it.
