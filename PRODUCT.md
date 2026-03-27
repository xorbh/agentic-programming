# Slamit — A Lightweight Google Docs Replacement

## The Problem

Google Docs is powerful, but it's overkill for most document sharing. You just want to write something in markdown, get a link, and send it. Instead, you fight with formatting toolbars, manage permissions through three different dialogs, and deal with bloated pages that take seconds to load. Worse, there's no good API story — integrating Google Docs into your workflow requires OAuth gymnastics and a byzantine API.

Slamit strips document sharing down to its essence: **write markdown, publish, share a link**. That's it.

---

## What It Does

Slamit is a multi-tenant markdown publishing platform. You write a document in markdown, publish it through the dashboard or API, and get shareable links with built-in expiration. Recipients see a clean, fast-loading rendered page — no sign-up required. Claude can publish and manage documents directly through the MCP integration.

### Core Capabilities

- **Publish markdown documents** via web UI, API, or Claude (MCP)
- **Generate share links** with configurable expiration (1 hour to permanent)
- **Track views** on every shared link
- **Manage access** — revoke links instantly, create new ones anytime
- **Organize by team** — Clerk organizations provide multi-tenant isolation
- **API-first design** — every action available via REST API with API key auth

---

## Architecture

```
                                  ┌─────────────────────┐
                                  │   Clerk (Auth)       │
                                  │   Orgs + Users       │
                                  │   JWKS + Webhooks    │
                                  └──────────┬──────────┘
                                             │
          ┌──────────────┐          ┌────────▼────────┐          ┌──────────────┐
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

### Components

| Component | Technology | Deployed To | Purpose |
|-----------|-----------|-------------|---------|
| **Frontend** | React 19 + Vite | Vercel | Dashboard for managing documents, links, and API keys |
| **API Server** | Express + TypeScript | Fly.io (Mumbai) | REST API, auth, document lifecycle, link management |
| **MCP Server** | Python + FastMCP | Fly.io (Mumbai) | Claude integration — publish/manage docs from chat |
| **Database** | DynamoDB (single table) | AWS ap-south-1 | All metadata: tenants, documents, links, users, API keys |
| **Object Store** | S3 | AWS ap-south-1 | Raw markdown + pre-rendered HTML for each document |
| **Auth** | Clerk | Hosted | Organization-based multi-tenancy, JWT auth, webhooks |
| **IaC** | Terraform | N/A | DynamoDB tables, S3 buckets, IAM roles |

---

## Data Model

Everything lives in a single DynamoDB table using a composite key pattern (`PK` + `SK`). This gives us flexible access patterns without multiple tables.

### Item Types

```
TENANT#{orgId}  +  META                    → Tenant metadata (name, S3 bucket, prefix)
TENANT#{orgId}  +  DOC#{docId}             → Document metadata (title, author, S3 key)
TENANT#{orgId}  +  LINK#{linkId}           → Link metadata scoped to tenant (for listing)
TENANT#{orgId}  +  USER#{clerkUserId}      → User record (email, role)
TENANT#{orgId}  +  APIKEY#{apiKey}         → API key scoped to tenant
LINK#{linkId}   +  LINK#{linkId}           → Link lookup for public viewers (no tenant leak)
APIKEY#{apiKey} +  APIKEY#{apiKey}         → API key lookup (fast auth without knowing tenant)
```

**Why the dual-item pattern for links and API keys?** Public viewers and API callers don't know the tenant ID. The standalone items (`LINK#x + LINK#x`, `APIKEY#x + APIKEY#x`) allow fast O(1) lookups. The tenant-scoped items allow listing all links/keys for a given org. Two items, two access patterns, no scans.

### Key Design Decisions

- **Single table design** — one DynamoDB table, minimal infrastructure, all queries are key-based
- **TTL on links** — `expires_at` attribute enables DynamoDB to auto-delete expired links (defensive server-side check too)
- **GSI on SK** — `SK-index` enables user-to-tenant lookups when Clerk JWT doesn't include org_id
- **S3 dual storage** — each document stored as both `source.md` and `rendered.html`, so public views are fast (no rendering at request time)
- **Point-in-time recovery** enabled for disaster scenarios

---

## Authentication

Two auth strategies, one middleware:

### 1. Clerk JWT (Web Dashboard)

```
User → Clerk → JWT with org_id → API verifies via JWKS → Tenant resolved
```

- Frontend uses `@clerk/clerk-react` for sign-in and org switching
- JWTs are RS256-signed, verified against Clerk's JWKS endpoint (cached 10min)
- Organization ID in JWT maps directly to tenant ID in DynamoDB
- Webhook events auto-provision tenants and users on org creation

### 2. API Keys (Programmatic Access)

```
Client → Authorization: Bearer mh_xxx → Lookup APIKEY#mh_xxx → Tenant resolved
```

- Format: `mh_` prefix + 32-character nanoid
- Created through dashboard, shown once in full, then masked (`mh_xxxxx...xxxx`)
- Each key maps to a tenant via the standalone `APIKEY#` item
- Ideal for CI/CD, scripts, and server-to-server integration

Both strategies converge to the same result: `req.tenant` and `req.clerkUserId` attached to the request.

---

## Document Lifecycle

### Publishing

```
Markdown input
    │
    ├─► marked (GFM) → HTML → DOMPurify sanitization
    │
    ├─► Upload to S3: {prefix}/{docId}/source.md
    ├─► Upload to S3: {prefix}/{docId}/rendered.html
    │
    ├─► DynamoDB: create DOC# item
    │
    └─► Auto-create 2 share links (24h quick share + 30d extended)
```

Title is auto-extracted from the first `# Heading` in the markdown. Documents can be updated (new markdown replaces both S3 objects) or deleted (cascading delete of all links).

### Viewing (Public)

```
GET /l/{linkId}
    │
    ├─► Lookup LINK#{linkId} → get tenant_id, doc_id, check expiry
    ├─► Fetch rendered.html from S3
    ├─► Increment view count (async, non-blocking)
    │
    └─► Return clean HTML page with metadata (og:title, author, date)
```

No authentication required. The rendered page is minimal, fast, and includes proper OpenGraph tags for rich link previews in Slack, Twitter, etc.

### Link Management

| Action | Endpoint |
|--------|----------|
| Auto-create on publish | `POST /api/documents` (returns links) |
| Create custom link | `POST /api/documents/:docId/links` |
| List document's links | `GET /api/documents/:docId/links` |
| Revoke link | `DELETE /api/links/:linkId` |

Expiry presets in the UI: 1 hour, 24 hours, 7 days, 30 days, or never. API accepts `expires_in` in seconds for any custom duration.

---

## MCP Integration (Claude)

The MCP server lets Claude publish and manage documents directly from a conversation. It runs as a separate Python service using FastMCP with streamable-http transport.

### Available Tools

| Tool | What It Does |
|------|-------------|
| `publish_document` | Create a new document from markdown |
| `list_documents` | Show all documents in the org |
| `get_document` | Fetch document details + all share links |
| `update_document` | Replace document content |
| `delete_document` | Remove a document and all its links |
| `create_share_link` | Generate a new share link with optional expiry |
| `list_share_links` | Show all links for a document |
| `revoke_share_link` | Delete a share link |

### Auth Flow

The MCP server uses Clerk OAuth to authenticate users. Tokens are persisted in DynamoDB (encrypted with Fernet) so users don't re-authenticate on every conversation. The server validates JWTs against Clerk's JWKS before proxying requests to the main API.

---

## Security

| Layer | Mechanism |
|-------|-----------|
| **Transport** | HTTPS enforced on all Fly.io services |
| **Auth** | RS256 JWT verification + API key validation |
| **Content** | DOMPurify sanitization on all rendered markdown |
| **Storage** | S3 server-side encryption (KMS), all public access blocked |
| **Tenant isolation** | All queries scoped by tenant PK — no cross-tenant access possible |
| **Rate limiting** | 30 req/min publish, 300 req/min view (per IP) |
| **API key handling** | Keys masked in UI/API responses, stored with tenant association |
| **MCP tokens** | Fernet-encrypted at rest in DynamoDB with TTL |
| **Link expiry** | DynamoDB TTL + defensive server-side check (belt and suspenders) |
| **Webhook verification** | Clerk webhook signatures verified before processing |

---

## Infrastructure

### Terraform-Managed Resources

- **DynamoDB table** — on-demand billing, PITR enabled, TTL on `expires_at`
- **S3 bucket** — versioning, KMS encryption, public access blocked, multipart upload cleanup
- **IAM roles** — least-privilege for API (DynamoDB CRUD + S3 read/write) and deploy user

### Deployment

- **API**: Fly.io, 2+ machines, auto-scaling (soft limit 200, hard limit 250 connections), health checks on `/health`
- **MCP Server**: Fly.io, scales to zero when idle (min 0 machines)
- **Frontend**: Vercel, SPA with catch-all rewrite to `index.html`
- **Region**: Mumbai (ap-south-1) for all services

---

## Why This Over Google Docs

| | Google Docs | Slamit |
|---|------------|--------|
| **Format** | WYSIWYG, proprietary | Markdown (portable, version-control friendly) |
| **Sharing** | Complex permissions UI | One link, optional expiry |
| **Recipient experience** | Requires Google account (or clunky "anyone with link") | No account needed, fast page load |
| **API** | OAuth2 + Docs API (hundreds of endpoints) | REST API + API keys (8 endpoints) |
| **AI integration** | Limited | Native MCP — Claude can publish/manage documents |
| **View tracking** | None for external viewers | Built-in view counts per link |
| **Link control** | Link lives forever once shared | Expiring links, instant revocation |
| **Self-hostable** | No | Yes (Fly.io + AWS) |
| **Load time** | 2-5s for a typical doc | Pre-rendered HTML, sub-second |
| **Cost** | Free (with Google lock-in) | Pay-per-use AWS + Fly.io (cents/month for small teams) |

---

## API Reference

All authenticated endpoints require `Authorization: Bearer {token}` (Clerk JWT or `mh_*` API key).

### Documents

```
POST   /api/documents              Create document (returns auto-generated links)
GET    /api/documents              List all documents
GET    /api/documents/:docId       Get document with links
PUT    /api/documents/:docId       Update document content
DELETE /api/documents/:docId       Delete document and all links
```

### Share Links

```
POST   /api/documents/:docId/links   Create share link
GET    /api/documents/:docId/links   List links for document
DELETE /api/links/:linkId            Revoke share link
```

### API Keys

```
POST   /api/keys                   Create API key
GET    /api/keys                   List API keys (masked)
DELETE /api/keys/:apiKey           Revoke API key
```

### Public

```
GET    /l/:linkId                  View rendered document
GET    /l/:linkId/raw              Get raw markdown
```

---

## Getting Started

### Prerequisites

- Node.js 20+
- AWS account (DynamoDB + S3)
- Clerk account (auth)
- Fly.io account (deployment)

### Local Development

```bash
# Backend
cp .env.example .env   # fill in AWS, Clerk credentials
npm install
npm run dev

# Frontend
cd client
cp .env.example .env   # fill in Clerk key + API URL
npm install
npm run dev
```

### Deploy

```bash
# Infrastructure
cd terraform
terraform init
terraform apply

# Backend
fly deploy

# MCP Server
cd mcp-server
fly deploy

# Frontend
vercel deploy
```

---

*Slamit: Write markdown. Share a link. Done.*
