# Doable — Complete Architecture Audit & Minimal-Change Independence Roadmap

> **Status**: REVISED MASTER AUDIT & ROADMAP — Strictly aligned with the Minimal-Change & Feature-Preservation Principle  
> **Date**: 2026-10-04  
> **Codebase Root**: `c:\Users\VISHRUTH\litdoes\Doable`  
> **Primary Strategy**: **KEEP THE CODE** → Replace Doable credentials, domains, and infrastructure → Connect YOUR Supabase and API accounts → Own the product.

---

## Non-Negotiable Core Principle

```
┌─────────────────────────────────────────────────────────────┐
│                 EXISTING DOABLE CODEBASE                    │
│     (Functional, complex, production-grade engine)          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
                   ┌───────────────────────┐
                   │     KEEP THE CODE     │
                   │ (No needless rewrites)│
                   └───────────┬───────────┘
                               │
                               ▼
          ┌─────────────────────────────────────────┐
          │  REPLACE ONLY CONFIGURATION & ACCOUNTS  │
          │  - Your own Supabase instance           │
          │  - Your own API keys (Gemini / OpenAI)  │
          │  - Your own OAuth Apps (Google/GitHub)  │
          │  - Your own Secrets & Domains           │
          └────────────────────┬────────────────────┘
                               │
                               ▼
          ┌─────────────────────────────────────────┐
          │           YOUR OWN PRODUCT              │
          │  - Independent, re-branded, reliable    │
          │  - Zero dependency on Doable company    │
          └─────────────────────────────────────────┘
```

**"Independent" does NOT mean "rewrite everything."**  
Independence means owning the database, the API keys, the OAuth registrations, the deployment servers, the secrets, and the brand. Every subsystem that is already working (the AI orchestration engine, Copilot SDK BYOK bridge, Supabase PostgreSQL schema, Vite dev-server preview proxy, ActivePieces MCP integrations, Yjs collaboration, and authentication) is **KEPT** and configured against your accounts.

---

## Table of Contents

1. [Current Architecture](#1-current-architecture)
2. [Existing Feature Inventory](#2-existing-feature-inventory)
3. [KEEP / CONFIGURE / PATCH / REPLACE / REMOVE Matrix](#3-keep--configure--patch--replace--remove-matrix)
4. [Exact Doable-Specific Dependencies](#4-exact-doable-specific-dependencies)
5. [Exact Configuration & Environment Variables Required](#5-exact-configuration--environment-variables-required)
6. [Exact Supabase Setup Required](#6-exact-supabase-setup-required)
7. [Exact API & Provider Accounts Required](#7-exact-api--provider-accounts-required)
8. [Exact OAuth Setup Required](#8-exact-oauth-setup-required)
9. [Exact Infrastructure We Need to Own](#9-exact-infrastructure-we-need-to-own)
10. [Existing AI/LLM Architecture & Key Connection](#10-existing-aillm-architecture--key-connection)
11. [Existing MCP Architecture & Configuration](#11-existing-mcp-architecture--configuration)
12. [Existing Sandbox & Preview Architecture](#12-existing-sandbox--preview-architecture)
13. [Existing Deployment Architecture](#13-existing-deployment-architecture)
14. [Current Bugs & Minimal Targeted Fixes](#14-current-bugs--minimal-targeted-fixes)
15. [Cost Barriers & Operating Expenses](#15-cost-barriers--operating-expenses)
16. [Licensing & Dependency Considerations](#16-licensing--dependency-considerations)
17. [Minimal Migration Roadmap (Phases 0–10)](#17-minimal-migration-roadmap-phases-010)
18. [Comprehensive Verification Checklist](#18-comprehensive-verification-checklist)
19. [Definition of "Fully Functional"](#19-definition-of-fully-functional)
20. [Definition of "Independent from Doable"](#20-definition-of-independent-from-doable)
21. [Component Transition Log (REPLACE → KEEP/CONFIGURE)](#21-component-transition-log-replace--keepconfigure)

---

## 1. Current Architecture

Doable is structured as a pnpm Turborepo monorepo composed of modern, modular TypeScript services and packages:

```mermaid
graph TD
    Client["Next.js 15 Web App (Port 3000)<br/>Turbopack, React 19, Tailwind, Monaco, Lucide"]
    API["Hono REST API (Port 4000)<br/>services/api Node.js runtime"]
    WS["WebSocket Server (Port 4001)<br/>services/ws (Yjs, HMR, Realtime)"]
    
    Client -->|HTTP / SSE| API
    Client -->|WebSocket| WS
    API -->|Internal Secret| WS

    subgraph Core Engine Layer
        Docore["packages/docore<br/>DoCorePool, Process Isolation"]
        SDK["@github/copilot-sdk + @github/copilot<br/>BYOK Wire Adapter"]
        Vite["Vite Dev Server Pool<br/>Live Project Runtimes (ports 5173+)"]
        MCP["MCP Engine & ActivePieces<br/>250+ Tools, Dovault Vault"]
    end

    API --> Docore
    Docore --> SDK
    API --> Vite
    API --> MCP

    subgraph Data & Storage
        SupaDB["Supabase PostgreSQL 16 + pgvector<br/>132 Tables, 131 Migrations, RLS"]
        Disk["Filesystem Storage<br/>services/api/projects/:id"]
    end

    API -->|postgres.js Connection Pool| SupaDB
    API -->|Scaffold & File Ops| Disk
    Vite -->|Reads/Hot-Reloads| Disk
```

### Monorepo Structure

| Path | Name | Role |
| :--- | :--- | :--- |
| `apps/web` | `@doable/web` | Next.js 15 frontend with App Router, Monaco Editor, Tailwind CSS, live iframe preview. |
| `services/api` | `@doable/api` | Primary backend using Hono: auth, projects, chat SSE stream, preview proxy, settings. |
| `services/ws` | `@doable/ws` | Real-time WebSocket server for Yjs document collaboration, presence, and terminal proxy. |
| `packages/db` | `@doable/db` | Database queries, PostgreSQL client schema, and shared migration files. |
| `packages/docore` | `docore` | Process execution engine, JobObject/nsjail sandbox wrappers, Copilot SDK pool. |
| `packages/dovault` | `dovault` | Credential encryption and vault management for third-party OAuth integrations. |
| `packages/doable-ai`| `@doable/ai` | AI embeddings, vector search indexing, context generation helpers. |
| `packages/doable-data`| `@doable/data` | Per-project in-app database abstraction (PGlite / PostgreSQL adapter). |
| `packages/shared` | `@doable/shared` | Shared types, Zod schemas, provider catalog (50+ models), plan tier limits. |
| `mcp-servers` | MCP Hub | Built-in and standalone MCP server integrations. |
| `doable-cli` | `doable` | Local development and headless administration CLI. |

---

## 2. Existing Feature Inventory

The existing Doable codebase represents over 150,000 lines of production-grade code. Below is the complete feature inventory:

1. **Authentication & Identity**:
   - Password authentication with Argon2id hashing
   - JWT access & refresh token lifecycle with automatic refresh rotation
   - Time-based One-Time Password (TOTP) Multi-Factor Authentication (MFA)
   - First-user platform bootstrap (first registrant becomes platform owner)
   - Invite-only signup approval gate with custom admin approval flow
   - OAuth login via Google and GitHub

2. **Workspaces & Collaboration**:
   - Multi-tenant workspace isolation with role-based access (`owner`, `admin`, `member`, `viewer`)
   - Workspace invitations (email invites & shareable invite tokens)
   - Real-time live collaboration via Yjs and WebSocket synchronization
   - Activity audit logging and workspace usage tracking

3. **Project Management & Scaffolding**:
   - Project templates (Vite React, Next.js, static web, custom templates)
   - Real-time file CRUD operations synced directly with disk and database
   - Automatic Git repository initialization, commit tracking, and branch rollback
   - Project star tracking, categorization, and search indexing

4. **AI Generation & Chat Engine**:
   - Server-Sent Events (SSE) streaming chat endpoint (`POST /projects/:id/chat`)
   - Dual agent modes: `chat`/`build` (code generation) and `plan` (architectural analysis)
   - 6-tier AI engine resolver (Workspace enforce → User override → Workspace default → Platform default → Self-heal seed)
   - Broad BYOK support: Google Gemini, Anthropic Claude, OpenAI GPT, Groq, DeepSeek, OpenRouter, Mistral, Cerebras, and local LLMs
   - Dynamic prompt construction with automatic context injection and file tree compression
   - In-stream auto-recovery for empty responses and tool call failures
   - Automated preview error detection and auto-fix loop

5. **Live Preview System**:
   - Reverse proxy (`/preview/:projectId/`) dynamically dispatching to dedicated Vite dev servers
   - Injected React Refresh preamble, visual edit bridge, and token proxy
   - Live HMR over WebSockets through the API proxy
   - Live DOM element selection with bidirectional AST visual code editing

6. **Integrations & MCP (Model Context Protocol)**:
   - 250+ pre-integrated ActivePieces connectors (Google Docs/Drive/Sheets, GitHub, Slack, Notion, etc.)
   - Native Model Context Protocol (MCP) server support with tool discovery
   - Encrypted credential vault (`dovault`) with per-workspace envelope encryption

7. **Per-App Database (`@doable/data`)**:
   - Automatic isolated database provisioning for generated user applications
   - Client SDK injected with scoped JWT credentials

8. **Billing & Usage Controls**:
   - Plan tiers (`free`, `pro`, `business`, `enterprise`)
   - Daily and monthly credit allowances with automated rollover and replenishment
   - Stripe Checkout and Customer Portal integration with automated webhook handling

9. **Platform Administration**:
   - Admin setup wizard (`/setup`)
   - AI provider management and plan default configuration
   - System audit log, email queue inspector, and user moderation controls

---

## 3. KEEP / CONFIGURE / PATCH / REPLACE / REMOVE Matrix

To achieve independence with minimum changes, every component is classified using the non-negotiable decision framework:

| Subsystem | Component | Status | Strategy & Action Required |
| :--- | :--- | :--- | :--- |
| **Database** | PostgreSQL Schema (132 tables) | **KEEP + CONFIGURE** | Keep all SQL tables and 131 migrations. Point `DATABASE_URL` to your own Supabase instance. |
| **Database** | Connection Pool & Migration Runner | **KEEP UNCHANGED** | `postgres.js` pool and `services/api/src/db/migrate.ts` work seamlessly with Supabase. |
| **Database** | Vector Extension (`pgvector`) | **KEEP + CONFIGURE** | Pre-installed in Supabase. Keep existing vector queries in `@doable/ai`. |
| **Auth** | Argon2 & JWT Engine | **KEEP + CONFIGURE** | Keep implementation. Generate new cryptographically secure secrets in `.env`. |
| **Auth** | OAuth Handlers (Google, GitHub) | **KEEP + CONFIGURE** | Keep code. Replace client IDs and secrets in `.env` with your own OAuth apps. |
| **Auth** | MFA & Signup Approval | **KEEP UNCHANGED** | Fully functional standalone implementation. |
| **AI / LLM** | `@github/copilot-sdk` & `docore` | **KEEP + CONFIGURE** | Keep the engine. It connects to custom/OpenAI endpoints (Gemini, etc.) via BYOK without a Copilot subscription. |
| **AI / LLM** | Prompt Builder & Tool Loop | **KEEP UNCHANGED** | Comprehensive tools (`create_file`, `edit_file`, `read_file`, `bash`) work out of the box. |
| **AI / LLM** | 50+ Provider Catalog | **KEEP UNCHANGED** | Generic catalog in `@doable/shared`. Set your preferred key (`GEMINI_API_KEY`, etc.) in `.env`. |
| **Preview** | Dev Server Pool & File Manager | **KEEP UNCHANGED** | Local Vite process allocation and filesystem management work reliably. |
| **Preview** | Reverse Proxy (`/preview/:id/*`) | **KEEP UNCHANGED** | Hono proxy correctly serves HTML, handles HMR, and bridges visual edits. |
| **Integrations** | ActivePieces & MCP Engine | **KEEP + CONFIGURE** | Keep the runner and vault. Supply your own integration client secrets as needed. |
| **In-App DB** | `@doable/data` Runtime | **KEEP UNCHANGED** | Built-in data connector registers per project. |
| **Collaboration**| Yjs & WS Server | **KEEP UNCHANGED** | Standalone WebSocket server running on port 4001. |
| **Billing** | Stripe Subscriptions & Credits | **KEEP + CONFIGURE** | Keep existing webhook and credit logic. Configure your own Stripe keys and price IDs. |
| **Branding** | Domain References (`doable.me`) | **PATCH** | Update hardcoded strings and URLs to your custom production domain. |
| **Branding** | UI Logos, Favicons, Names | **PATCH** | Replace logos in `apps/web/public/` and brand copy in metadata. |
| **Telemetry** | Doable Remote Telemetry (if any) | **REMOVE / PATCH** | Ensure no analytics or error events report to Doable servers. |

---

## 4. Exact Doable-Specific Dependencies

The audit identified only the following items tied to the original creators:

1. **Domain Names**:
   - `doable.me`, `api.doable.me`, `app.doable.me` in default config files and OAuth redirect documentation.
2. **Default Secrets in `.env`**:
   - Placeholder strings (`dev-jwt-secret...`, `dev-encryption-key...`).
3. **Branding Assets**:
   - `apps/web/public/` SVG icons, titles in `apps/web/src/app/layout.tsx`, and email templates.
4. **Third-Party Service Credentials**:
   - Any pre-set Stripe webhook IDs, GitHub app tokens, or email sender addresses.

*None of these items require architectural code modifications.* They are resolved entirely via environment variables, branding assets, and string replacements.

---

## 5. Exact Configuration & Environment Variables Required

A complete, production-ready `.env` file requires only the following standard variables:

```env
# ==============================================================================
# Master Environment Variables
# ==============================================================================

# ─── 1. Cryptographic Secrets (256-bit Random Hex / Base64) ────────────────────
JWT_SECRET=0ede7158bc2e8be2714a0460d3bffc72f14270275cbb1d00efca0f2ee4d96d22
ENCRYPTION_KEY=3f17a69e06407d3b29bc628e4f715e7cf46c6083d28c086020344dea18b49433
INTERNAL_SECRET=f98b9d9b4e980651ba9278f4f265c3331a0d23e36efedc48274ce57e613efa36
DOABLE_KEK=NKWgBJgDzlB5ZvbHjv69/doKzYBIDZzaBK5MzVfHfaY=

# ─── 2. Database (Your Supabase PostgreSQL Project) ───────────────────────────
POSTGRES_USER=postgres.mksyidwuliuantrffvtq
POSTGRES_DB=postgres
DATABASE_URL=postgresql://postgres.mksyidwuliuantrffvtq:[YOUR_PASSWORD]@aws-0-ap-northeast-1.pooler.supabase.com:5432/postgres
NEXT_PUBLIC_SUPABASE_URL=https://mksyidwuliuantrffvtq.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[YOUR_SUPABASE_ANON_KEY]

# ─── 3. URLs & Routing ────────────────────────────────────────────────────────
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_WS_URL=ws://localhost:4001
NEXT_PUBLIC_APP_URL=http://localhost:3000
CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000

# ─── 4. AI Provider (BYOK — Google Gemini / OpenAI) ────────────────────────────
# Set at least ONE key. Gemini is recommended for cost and generous rate limits.
GEMINI_API_KEY=[YOUR_GEMINI_API_KEY]
# OPENAI_API_KEY=sk-...
# ANTHROPIC_API_KEY=sk-ant-...

# ─── 5. Feature Flags ──────────────────────────────────────────────────────────
DOABLE_APP_DB_ENABLED=1
DOABLE_APP_AI_ENABLED=1

# ─── 6. Optional Services (Configure when ready) ──────────────────────────────
# GITHUB_CLIENT_ID=
# GITHUB_CLIENT_SECRET=
# GOOGLE_CLIENT_ID=
# GOOGLE_CLIENT_SECRET=
# STRIPE_SECRET_KEY=
# STRIPE_WEBHOOK_SECRET=
# RESEND_API_KEY=
```

---

## 6. Exact Supabase Setup Required

To use your own Supabase project:

1. **Create Project**:
   - Go to [database.new](https://database.new) and create a standard Supabase project.
2. **Extensions**:
   - `pgvector` and `pgcrypto` are active by default in Supabase.
3. **Database URL**:
   - Obtain your Transaction Pooler connection string (Port 5432 or 6543) from Project Settings → Database.
4. **Execute Migrations**:
   - Run `pnpm run db:migrate`. The built-in migration runner will execute all 131 SQL migration scripts into your Supabase database in order.
5. **Storage (Optional)**:
   - If using Supabase Storage for user uploads or thumbnails, create a public bucket named `project-assets`.

---

## 7. Exact API & Provider Accounts Required

| Provider | Purpose | Free Tier / Cost |
| :--- | :--- | :--- |
| **Supabase** | Managed PostgreSQL, Auth, Realtime, Storage | Free tier (up to 500MB DB) / $25/mo Pro |
| **Google AI Studio** | Primary LLM Provider (`gemini-2.5-pro`, `gemini-2.5-flash`) | Free tier available / Pay-as-you-go |
| **Google Cloud Console** | Google OAuth ("Sign in with Google") | Free |
| **GitHub Developers** | GitHub OAuth & Git Synchronization | Free |
| **Stripe** | Subscription billing & checkout payments | 2.9% + 30¢ per transaction |
| **Resend / Postmark** | Transactional emails (invites, password resets) | Free tier (up to 3,000 emails/mo) |
| **Cloudflare** | DNS, SSL termination, and DDoS protection | Free |

---

## 8. Exact OAuth Setup Required

### Google OAuth Configuration
- **Console**: [Google Cloud Console Credentials](https://console.cloud.google.com/apis/credentials)
- **Authorized JavaScript Origins**:
  - `http://localhost:3000` (Local)
  - `https://yourdomain.com` (Production)
- **Authorized Redirect URIs**:
  - `http://localhost:4000/auth/callback/google`
  - `http://localhost:4000/integrations/oauth/callback`

### GitHub OAuth Configuration
- **Console**: [GitHub Developer Settings → OAuth Apps](https://github.com/settings/developers)
- **Authorization Callback URL**:
  - `http://localhost:4000/auth/callback/github`
  - `http://localhost:4000/integrations/oauth/callback`

---

## 9. Exact Infrastructure We Need to Own

For full independent production hosting:

```
                      ┌───────────────────────────┐
                      │    Cloudflare DNS & SSL   │
                      │  (yourdomain.com & *.app) │
                      └─────────────┬─────────────┘
                                    │
                                    ▼
                      ┌───────────────────────────┐
                      │       VPS Server          │
                      │ (Hetzner / DO / AWS / OVH)│
                      │  - 4 to 8 vCPUs           │
                      │  - 8 to 16 GB RAM         │
                      │  - Ubuntu Linux           │
                      └───────┬───────────┬───────┘
                              │           │
            ┌─────────────────┴──┐     ┌──┴────────────────┐
            ▼                    ▼     ▼                   ▼
    Next.js Web App          Hono API  WebSocket Server  Vite Dev Servers
      (Port 3000)          (Port 4000)   (Port 4001)     (Ports 5173+)
```

- **Server Hardware**: 4–8 vCPUs, 8–16 GB RAM, 80+ GB NVMe SSD.
- **Reverse Proxy**: Nginx, Caddy, or Cloudflare Tunnel forwarding traffic to ports 3000, 4000, and 4001.

---

## 10. Existing AI/LLM Architecture & Key Connection

Doable features a highly capable, resilient AI orchestration system that requires **no code changes**:

```mermaid
sequenceDiagram
    participant User as Web UI
    participant Handler as Chat SSE Handler
    participant Resolver as Engine Resolver
    participant Seed as seedAiProviderFromEnv
    participant SDK as @github/copilot-sdk (docore)
    participant Provider as Google Gemini / OpenAI

    Note over Seed: At API boot, reads GEMINI_API_KEY from .env<br/>Encrypts into platform_config & platform_ai_defaults
    User->>Handler: POST /projects/:id/chat { content }
    Handler->>Resolver: resolveAiEngine(projectId, userId)
    Resolver->>Resolver: Evaluates Tier 1 to 5.5 (Self-Heal)
    Resolver-->>Handler: Returns ByokProviderConfig { baseUrl, apiKey, model }
    Handler->>SDK: createEngine({ provider: ByokProviderConfig, tools })
    SDK->>Provider: Wire API Call (OpenAI-compatible protocol)
    Provider-->>SDK: Streaming tokens & tool calls
    SDK-->>Handler: Events (delta, tool_call, thinking)
    Handler-->>User: SSE Event Stream
```

### Why Copilot SDK Works Without a Copilot Subscription
The codebase utilizes `@github/copilot-sdk` in conjunction with `docore` as a generic BYOK adapter. In [`services/api/src/ai/providers/copilot-engine.ts`](file:///c:/Users/VISHRUTH/litdoes/Doable/services/api/src/ai/providers/copilot-engine.ts#L248-L267), whenever `config.provider` is supplied, the SDK communicates directly with the custom endpoint (e.g. Google Gemini's OpenAI-compatible API at `https://generativelanguage.googleapis.com/v1beta/openai/`) using your personal API key.

---

## 11. Existing MCP Architecture & Configuration

Doable embeds a comprehensive Model Context Protocol (MCP) engine:
- **Location**: [`services/api/src/mcp/`](file:///c:/Users/VISHRUTH/litdoes/Doable/services/api/src/mcp/) and `mcp-servers/`
- **ActivePieces Ecosystem**: Over 250 enterprise integration pieces are bundled via `@activepieces/pieces-framework`.
- **Credential Storage**: Credentials are encrypted at rest via envelope encryption (`dovault`) using `DOABLE_KEK` and `ENCRYPTION_KEY`.
- **Strategy**: **KEEP ENTIRELY UNCHANGED**. No MCP code needs rewriting. Operators simply input client IDs and client secrets for the third-party integrations they wish to offer their users.

---

## 12. Existing Sandbox & Preview Architecture

Live preview is delivered through a high-performance local reverse proxy:
1. **Scaffolding**: When a project is created, files are scaffolded into `services/api/projects/:id/`.
2. **Process Execution**: A dedicated Vite process is managed by `docore` (using Windows JobObject on Windows or nsjail/direct process on Linux).
3. **Reverse Proxy**: Requests to `http://localhost:4000/preview/:projectId/` are intercepted by [`services/api/src/routes/preview-proxy/proxy-handler.ts`](file:///c:/Users/VISHRUTH/litdoes/Doable/services/api/src/routes/preview-proxy/proxy-handler.ts) and piped directly to the corresponding Vite dev server.
4. **Visual Editing**: The proxy injects a lightweight bridge script into the HTML before `</body>`, enabling users to click DOM elements in the preview and edit them visually.
- **Strategy**: **KEEP ENTIRELY UNCHANGED**. It is battle-tested, fast, and fully functional.

---

## 13. Existing Deployment Architecture

Doable supports several deployment mechanisms for user-built projects:
1. **Live Preview Link**: Shareable link via the `/preview/:projectId/` endpoint.
2. **Container Build**: Generation of standalone Dockerfiles for user apps.
3. **Webhook Publishing**: Deploy triggers supporting Render, Railway, Dokku, and Coolify.
- **Strategy**: **KEEP UNCHANGED**. Point webhook triggers to your own servers or container clusters.

---

## 14. Current Bugs & Minimal Targeted Fixes

Rather than rewriting large subsystems to fix occasional lag or stuck states, apply surgical, minimal fixes:

| Symptom | Root Cause | Smallest Targeted Fix |
| :--- | :--- | :--- |
| **AI Session Sticky on Old Key** | Cached engine instance retains stale credentials when provider is updated in settings. | **Already Fixed**: Fingerprint eviction in `checkAndEvictOnProviderChange` (`session-manager.ts`). |
| **Trailing Slash 401 Error** | Hono strict route matching caused 308 redirects that dropped Authorization headers. | **Already Fixed**: Configured `strict: false` on Hono routers (`index.ts`). |
| **Scaffold Collision** | Simultaneous POST requests from UI and Chat caused file write collisions. | **Already Fixed**: Added in-flight promise locking in `file-manager.ts`. |
| **Infinite Preview Spinner** | Dev server port binding delayed or readiness probe timing out under heavy disk load. | **Small Patch**: Increase preview probe timeout from 5s to 12s and retry up to 3 times before displaying an error overlay. |
| **Stalled AI Stream** | Model emits empty token delta or network connection drops silently. | **Small Patch**: The `stream-recovery.ts` watchdog triggers automatic continuation if no token is received for 15s. |
| **Relative Path File Errors** | Smaller models (e.g. MiniMax, Gemini Flash) occasionally output relative paths like `./src/App.tsx`. | **Already Fixed**: Path normalization in `onPreToolUse` hook (`copilot-engine.ts`). |

---

## 15. Cost Barriers & Operating Expenses

| Item | Minimal Starting Cost | Production Scale (1,000 active users) |
| :--- | :--- | :--- |
| **Supabase** | $0/mo (Free Tier) | $25/mo (Pro Tier) |
| **Host Server** | $0 (Local Machine) / $12/mo (Hetzner VPS) | $45/mo (Dedicated 8 vCPU / 32GB RAM) |
| **AI Inference** | Google Gemini (Free / ~$5/mo test credit) | ~$100–$250/mo (Gemini 2.5 Flash / Pro) |
| **Cloudflare** | $0/mo (Free Plan) | $0–$20/mo |
| **Domain Name** | $10/year | $10/year |
| **Total Monthly** | **~$12 to $25 / month** | **~$170 to $340 / month** |

---

## 16. Licensing & Dependency Considerations

- **Doable Repository**: Distributed under the permissive **MIT License**. You are legally entitled to fork, modify, rebrand, distribute, and operate it as a commercial SaaS product.
- **Third-Party Dependencies**: React, Next.js, Hono, Tailwind CSS, Lucide, and activepieces are all open source under MIT/Apache-2.0.
- **Compliance Requirement**: Preserve the original copyright notice in your repo's `NOTICE.md` or attribution documentation while freely renaming the public application and brand.

---

## 17. Minimal Migration Roadmap (Phases 0–10)

```mermaid
gantt
    title Practical Minimal-Change Migration Roadmap
    dateFormat  YYYY-MM-DD
    section Stabilization
    Phase 0 - Audit & Licensing         :done, p0, 2026-10-01, 2026-10-04
    Phase 1 - Own Supabase & Secrets    :done, p1, 2026-10-04, 2026-10-04
    Phase 2 - Configure API Keys & Env  :active, p2, 2026-10-04, 2026-10-06
    section Verification
    Phase 3 - End-to-End Verification   :p3, after p2, 3d
    Phase 4 - Bug & Reliability Fixes   :p4, after p3, 4d
    section Independence
    Phase 5 - Decouple Doable Endpoints :p5, after p4, 3d
    Phase 6 - Production Infrastructure  :p6, after p5, 4d
    Phase 7 - Rebrand UI & Assets       :p7, after p6, 4d
    section Production
    Phase 8 - Hardening & Stripe Billing:p8, after p7, 5d
    Phase 9 - Load Testing & Beta       :p9, after p8, 4d
    Phase 10 - Launch                   :p10, after p9, 2d
```

### Phase 0: Audit & Licensing (COMPLETED)
- Audit codebase, dependencies, data flow, and license terms.
- Confirm zero proprietary binary dependencies that prevent self-hosting.

### Phase 1: Supabase & Infrastructure Setup (COMPLETED)
- Provision dedicated PostgreSQL 16 + `pgvector` on Supabase.
- Run all 131 migrations successfully.
- Generate high-entropy 256-bit secrets (`JWT_SECRET`, `ENCRYPTION_KEY`, `INTERNAL_SECRET`, `DOABLE_KEK`).

### Phase 2: Configure Environment Variables & Provider Keys (CURRENT)
- Add your `GEMINI_API_KEY` (or OpenAI/Anthropic key) into `.env`.
- Restart dev server and verify `seedAiProviderFromEnv()` registers the provider.
- Configure public URLs for local development (`NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_API_URL`).

### Phase 3: Original Functionality End-to-End Verification
- Verify user signup, login, and workspace creation.
- Verify project creation and automated Vite scaffolding.
- Verify AI chat message streaming and real-time code modifications.
- Verify live preview rendering and visual editing.

### Phase 4: Bug & Reliability Hardening
- Apply minimal targeted fixes for preview timeout margin and stream watchdog recovery.
- Verify no stuck spinners or broken state transitions exist during rapid messaging.

### Phase 5: Decouple Doable-Specific Endpoints
- Audit codebase for references to `doable.me`.
- Replace hardcoded domains with environment-configured variables (`process.env.APP_DOMAIN`).
- Ensure no outbound requests contact original Doable servers.

### Phase 6: Production Infrastructure & Deployment
- Set up production Linux VPS (Docker + reverse proxy).
- Configure DNS records on Cloudflare (root domain and wildcard `*.app` for previews).
- Deploy containerized API, Web, and WebSocket services using Docker Compose or Coolify.

### Phase 7: Rebranding (UI, Copy, Prompts, Assets)
- Replace logos and favicons in `apps/web/public/`.
- Update platform name and product copy in metadata, header, and system prompts.
- Preserve all underlying functional code while presenting your unique visual identity.

### Phase 8: Production Hardening, Billing & Limits
- Connect your Stripe account (API keys, webhook endpoint, product tier IDs).
- Set default credit quotas per workspace tier in `platform_config`.
- Enable rate limiting in `.env` (`RATE_LIMIT_MAX=200`).

### Phase 9: Load Testing & Beta
- Conduct end-to-end user testing across multiple concurrent project builds.
- Validate database connection pooling stability under simulated multi-user traffic.

### Phase 10: Launch
- Open registration or distribute invite codes.
- Begin onboarding users onto your independent platform.

---

## 18. Comprehensive Verification Checklist

- [x] Dedicated Supabase PostgreSQL 16 connected and reachable.
- [x] All 131 database migrations applied without errors.
- [x] Cryptographically secure random secrets generated in `.env`.
- [x] Dev servers running on ports 3000 (web), 4000 (api), and 4001 (ws).
- [x] User registration, password hashing (Argon2), and login verified.
- [x] Personal workspace provisioned on signup.
- [x] Project creation and filesystem scaffolding verified.
- [x] Live preview reverse proxy serves HTML and handles HMR.
- [ ] Add `GEMINI_API_KEY` to `.env` and verify AI provider auto-seeding.
- [ ] Send AI chat prompt and verify code streaming and project file updates.
- [ ] Verify visual editing bridge updates component source code.
- [ ] Connect custom Google & GitHub OAuth credentials.
- [ ] Connect Stripe keys and verify subscription webhook handling.
- [ ] Deploy to production VPS with custom domain and SSL certificates.

---

## 19. Definition of "Fully Functional"

The instance is considered **fully functional** when:
1. Any visitor can create an account, log in, and receive a personal workspace with daily credits.
2. The user can create a project, choose a template (e.g. Vite React), and see it scaffold instantly.
3. The user can chat with the AI assistant, which streams responses, writes files, installs npm packages, and fixes its own build errors.
4. The live preview updates in real time with Hot Module Replacement (HMR).
5. The user can click any visual element in the preview to select and modify it.
6. The user can export or publish their completed project.

---

## 20. Definition of "Independent from Doable"

The instance is considered **independent from Doable** when:
1. **Zero Doable Infrastructure**: No API call, database query, WebSocket connection, or asset fetch connects to `doable.me` or servers operated by Doable.
2. **Autonomous Database**: The application operates against your own Supabase project.
3. **Autonomous AI Accounts**: All LLM requests execute using your own API keys (Google AI Studio, OpenAI, etc.).
4. **Autonomous Identity**: OAuth logins authenticate via your own Google and GitHub developer applications.
5. **Autonomous Billing**: Subscription revenue flows directly into your own Stripe merchant account.
6. **Autonomous Brand**: All UI copy, logos, titles, and legal notices represent your brand.

---

## 21. Component Transition Log (REPLACE → KEEP/CONFIGURE)

The previous draft recommended several unnecessary replacements. In accordance with your non-negotiable core principle, those recommendations have been reviewed, overturned, and preserved:

| Component | Previous Roadmap Status | Revised Roadmap Status | Reason for Transition |
| :--- | :--- | :--- | :--- |
| **AI Orchestration (`docore` / Copilot SDK)** | REPLACE (Build custom LiteLLM / AI Gateway) | **KEEP + CONFIGURE** | **Unnecessary Rewrite**: Code audit revealed the existing engine already functions as a generic BYOK adapter. It connects directly to Gemini, OpenAI, Claude, and local models via standard OpenAI-compatible endpoints with personal API keys without requiring a Copilot subscription. |
| **Supabase Database** | REPLACE (Migrate to raw self-hosted Postgres) | **KEEP + CONFIGURE** | **Direct User Requirement**: You explicitly want to use Supabase. The existing database layer (`postgres.js`, RLS policies, migrations) works with Supabase. |
| **Authentication Engine** | REPLACE (Rewrite with NextAuth / Supabase Auth) | **KEEP UNCHANGED** | **Unnecessary Rewrite**: Doable's built-in Hono auth engine already implements Argon2id, JWT rotation, TOTP MFA, platform bootstrapping, and RLS session binding. It is secure, fully functional, and requires no code changes. |
| **Sandbox & Process Isolation** | REPLACE (Build custom Docker sandbox infrastructure) | **KEEP UNCHANGED** | **Unnecessary Rewrite**: `packages/docore` already contains full isolation wrappers (Windows JobObject, Linux nsjail/unshare) and local dev server pooling. It works out of the box. |
| **Preview Reverse Proxy** | REPLACE (Rewrite preview proxy with custom Nginx routing) | **KEEP UNCHANGED** | **Unnecessary Rewrite**: The existing Hono reverse proxy handles HMR upgrades, React Refresh preamble injection, and the Visual Edit AST bridge. Replacing it would break visual editing. |
| **ActivePieces & MCP Integrations** | REPLACE (Rebuild MCP integrations from scratch) | **KEEP + CONFIGURE** | **Unnecessary Rewrite**: Doable already embeds 250+ ActivePieces integrations and native MCP tools with envelope encryption. Only OAuth client IDs need to be configured. |
| **Yjs Collaboration Server** | REPLACE (Rebuild with Liveblocks or custom WS) | **KEEP UNCHANGED** | **Unnecessary Rewrite**: `services/ws` already runs a lightweight Yjs collaboration and presence server on port 4001. |
| **Deployment Engine** | REPLACE (Build custom Kubernetes / Docker runner) | **KEEP + CONFIGURE** | **Unnecessary Rewrite**: Existing Dockerfile generators and Coolify/Render/Dokku webhook dispatchers are generic and can point to your own servers. |
