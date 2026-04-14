# Open Agents — Setup & Usage Guide (English)

> **目錄 / Table of Contents**
> - [Prerequisites](#prerequisites)
> - [Repository Setup](#repository-setup)
> - [Environment Variables](#environment-variables)
> - [PostgreSQL / Neon Database](#postgresql--neon-database)
> - [Vercel OAuth Integration](#vercel-oauth-integration)
> - [GitHub App Integration](#github-app-integration)
> - [Local Development](#local-development)
> - [Deploy to Vercel](#deploy-to-vercel)
> - [Using the App](#using-the-app)
> - [Useful Commands Reference](#useful-commands-reference)
> - [Project Layout](#project-layout)

---

## Prerequisites

Before you start, make sure the following tools are installed on your machine:

| Tool | Minimum Version | Install |
|------|----------------|---------|
| **Bun** | 1.2.14+ | `curl -fsSL https://bun.sh/install \| bash` |
| **Git** | any | [git-scm.com](https://git-scm.com/) |
| **Node.js** | 20+ (optional, only for tooling compatibility) | [nodejs.org](https://nodejs.org/) |
| **Vercel CLI** (optional) | latest | `bun add -g vercel` |

> ⚠️ This project uses **Bun exclusively**. Do **not** use `npm`, `yarn`, or `pnpm` — they are not supported and will produce incorrect lock files.

---

## Repository Setup

### 1. Fork the repository

Go to <https://github.com/vercel-labs/open-agents> and click **Fork** so you have your own copy.

### 2. Clone your fork

```bash
git clone https://github.com/<YOUR_USERNAME>/open-agents.git
cd open-agents
```

### 3. Install dependencies

```bash
bun install
```

This installs all workspace packages defined in `apps/*` and `packages/*`.

---

## Environment Variables

The app reads its configuration from `apps/web/.env`. A template is already provided:

```bash
cp apps/web/.env.example apps/web/.env
```

Open `apps/web/.env` and fill in the values described in the sections below.

### Minimum values required to boot the app

```env
POSTGRES_URL=          # PostgreSQL connection string (see Neon section)
JWE_SECRET=            # 32-byte URL-safe base64 string
```

Generate `JWE_SECRET`:

```bash
openssl rand -base64 32 | tr '+/' '-_' | tr -d '=\n'
```

### Required to sign in

```env
ENCRYPTION_KEY=                        # 32-byte hex string (64 hex chars)
NEXT_PUBLIC_VERCEL_APP_CLIENT_ID=      # Vercel OAuth App Client ID
VERCEL_APP_CLIENT_SECRET=              # Vercel OAuth App Client Secret
```

Generate `ENCRYPTION_KEY`:

```bash
openssl rand -hex 32
```

### Required for GitHub repo access, pushes, and PRs

```env
NEXT_PUBLIC_GITHUB_CLIENT_ID=   # GitHub App Client ID
GITHUB_CLIENT_SECRET=           # GitHub App Client Secret
GITHUB_APP_ID=                  # GitHub App ID (numeric)
GITHUB_APP_PRIVATE_KEY=         # PEM contents with \n escaped, or base64-encoded PEM
NEXT_PUBLIC_GITHUB_APP_SLUG=    # URL slug of your GitHub App
GITHUB_WEBHOOK_SECRET=          # Random secret you choose when creating the App
```

### Optional

```env
REDIS_URL=                                     # Redis for skills metadata cache
KV_URL=                                        # Vercel KV alternative
VERCEL_PROJECT_PRODUCTION_URL=                 # Canonical production URL
NEXT_PUBLIC_VERCEL_PROJECT_PRODUCTION_URL=     # Same, exposed to the browser
VERCEL_SANDBOX_BASE_SNAPSHOT_ID=               # Override default sandbox snapshot
ELEVENLABS_API_KEY=                            # Voice transcription
```

---

## PostgreSQL / Neon Database

### Option A — Use Neon (recommended)

1. Go to <https://neon.tech/> and create a free account.
2. Create a new **project** and a **database** inside it.
3. Copy the connection string (starts with `postgresql://…`).
4. Paste it as `POSTGRES_URL` in your `.env`.

> Neon's **branching** feature integrates with Vercel: every preview deployment automatically gets its own isolated database branch so preview and production data never mix.

### Option B — Use any PostgreSQL server

Any PostgreSQL 14+ instance works. Just set `POSTGRES_URL` to the correct connection string.

### Migrations

Migrations run **automatically** at build time (`bun run build`). Every Vercel deploy (preview or production) runs pending migrations against its own database. You never need to run migrations manually in production.

For local development, run:

```bash
bun run --cwd apps/web db:generate   # generate a migration after editing schema.ts
bun run --cwd apps/web db:migrate    # apply pending migrations locally
```

> Always commit the generated `.sql` file alongside any `schema.ts` change.

---

## Vercel OAuth Integration

Open Agents uses **Vercel OAuth** for sign-in.

### 1. Create a Vercel OAuth App

1. Visit <https://vercel.com/account/tokens> → **Integrations** → **OAuth Apps** → **Create**.
2. Set the **Callback URL** to:
   - Production: `https://YOUR_DOMAIN/api/auth/vercel/callback`
   - Local dev: `http://localhost:3000/api/auth/vercel/callback`

### 2. Copy the credentials

```env
NEXT_PUBLIC_VERCEL_APP_CLIENT_ID=<Client ID>
VERCEL_APP_CLIENT_SECRET=<Client Secret>
```

---

## GitHub App Integration

Open Agents does **not** need a separate GitHub OAuth App. It uses the **GitHub App's user-authorization flow** for both installation-based repo access and user identity.

### 1. Create a GitHub App

Go to <https://github.com/settings/apps/new> and configure:

| Setting | Value |
|---------|-------|
| **Homepage URL** | `https://YOUR_DOMAIN` (or `http://localhost:3000` for local) |
| **Callback URL** | `https://YOUR_DOMAIN/api/github/app/callback` |
| **Setup URL** | `https://YOUR_DOMAIN/api/github/app/callback` |
| **Webhook URL** | `https://YOUR_DOMAIN/api/github/webhook` |
| **Webhook Secret** | a random secret you choose |
| Request user authorization during install | ✅ enabled |
| Make App public | ✅ (recommended for org installs) |

#### Permissions needed

- **Repository**: Contents (read & write), Pull requests (read & write), Metadata (read)
- **Account**: Email addresses (read)

### 2. Generate a private key

In the App settings page, scroll to **Private Keys** and click **Generate a private key**. Download the `.pem` file.

Encode it for the env var:

```bash
# Option 1 — base64 (recommended)
base64 -w 0 your-app.YYYY-MM-DD.private-key.pem

# Option 2 — escaped newlines (paste PEM with \n replacing real newlines)
```

### 3. Copy the credentials

```env
NEXT_PUBLIC_GITHUB_CLIENT_ID=<App Client ID>
GITHUB_CLIENT_SECRET=<App Client Secret>
GITHUB_APP_ID=<Numeric App ID>
GITHUB_APP_PRIVATE_KEY=<base64-encoded PEM or escaped PEM>
NEXT_PUBLIC_GITHUB_APP_SLUG=<the URL slug shown in App settings>
GITHUB_WEBHOOK_SECRET=<your chosen webhook secret>
```

---

## Local Development

```bash
# 1. Install dependencies (if not done already)
bun install

# 2. Copy and fill in the env file
cp apps/web/.env.example apps/web/.env
# edit apps/web/.env with your values

# 3. Start the web app
bun run web
```

The app is now available at <http://localhost:3000>.

### Quality checks

Run these before committing any changes:

```bash
bun run ci          # format check + lint + typecheck + tests + db schema check
bun run fix         # auto-fix formatting and lint issues
bun run typecheck   # TypeScript type-check all packages
bun test            # run all tests
```

---

## Deploy to Vercel

### Step-by-step

1. **Fork** the repo and push to your GitHub account.
2. Go to <https://vercel.com/new> → **Import** your fork.
3. In **Environment Variables**, add at minimum:

   ```env
   POSTGRES_URL=
   JWE_SECRET=
   ENCRYPTION_KEY=
   ```

4. Click **Deploy**. After the first deploy completes, note your production URL.
5. **Add Vercel OAuth** credentials (`NEXT_PUBLIC_VERCEL_APP_CLIENT_ID`, `VERCEL_APP_CLIENT_SECRET`) and redeploy.
6. **Add GitHub App** credentials and redeploy.
7. (Optional) Add `REDIS_URL` / `KV_URL`, `VERCEL_PROJECT_PRODUCTION_URL`, and `ELEVENLABS_API_KEY`.

> Vercel's Neon integration can provision `POSTGRES_URL` automatically if you add it during project import.

### One-click deploy button

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?project-name=open-agents&repository-name=open-agents&repository-url=https%3A%2F%2Fgithub.com%2Fvercel-labs%2Fopen-agents)

---

## Using the App

### Sign in

1. Open the deployed URL (or `http://localhost:3000`).
2. Click **Sign in with Vercel**. Authorize the OAuth App.

### Connect your GitHub account

1. After sign-in, go to **Settings** → **Integrations**.
2. Click **Install GitHub App** and select the repositories or organization you want the agent to access.

### Start a new session (chat with the agent)

1. Click **New session** on the home page.
2. Type a prompt describing what you want the agent to do, for example:

   ```
   Add a /health endpoint to the Express app in this repo and write a test for it.
   ```

3. Optionally select a **GitHub repository** the agent should work in.
4. Press **Enter** or click **Send**.

### Agent capabilities

| Tool | What it does |
|------|-------------|
| **file** | Read, write, and search files inside the sandbox |
| **shell** | Run shell commands (`npm install`, `git`, etc.) |
| **search** | Semantic and regex code search |
| **task** | Spawn sub-agents for parallel work |
| **skill** | Load reusable skill prompts from the skills registry |
| **web** | Fetch web pages (docs, issues, etc.) |

### Session sharing

Copy the URL of any session and share it. Recipients can **read** the session (messages + file diffs) but cannot send new messages.

### Auto-commit / Auto-PR

In session settings, enable **Auto-commit** or **Auto-PR** so the agent automatically pushes branches and opens pull requests after a successful run.

### Voice input (optional)

If `ELEVENLABS_API_KEY` is set, a microphone button appears in the chat input. Click it to record a voice prompt.

---

## Useful Commands Reference

```bash
bun run web                          # Start local dev server (http://localhost:3000)
bun run ci                           # Full CI check (format + lint + types + tests + db)
bun run check                        # Lint + format check only
bun run fix                          # Auto-fix lint + format
bun run typecheck                    # TypeScript type check all packages
bun test                             # Run all tests
bun test path/to/file.test.ts        # Run a specific test file
bun run --cwd apps/web db:generate   # Generate a Drizzle migration
bun run --cwd apps/web db:migrate    # Apply migrations locally
bun run --cwd apps/web db:check      # Validate schema matches migrations
bun run sandbox:snapshot-base        # Refresh the base sandbox snapshot on Vercel
```

---

## Project Layout

```text
open-agents/
├── apps/
│   └── web/               # Next.js app — chat UI, auth, workflows, API routes
│       ├── app/           # App Router pages and layouts
│       ├── lib/           # Server utilities, DB schema (Drizzle), auth helpers
│       └── .env.example   # Environment variable template
├── packages/
│   ├── agent/             # Agent runtime — tools, sub-agents, skills loader
│   ├── sandbox/           # Sandbox abstraction and Vercel sandbox integration
│   ├── shared/            # Shared types and utilities
│   └── tsconfig/          # Shared TypeScript configurations
├── docs/
│   └── agents/            # Architecture, code-style, and lessons-learned docs
├── scripts/               # Helper scripts (snapshot refresh, isolated test runner)
├── turbo.json             # Turborepo pipeline config
├── package.json           # Root workspace + scripts
└── bun.lock               # Bun lock file (commit this)
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `bun install` fails | Make sure you are on Bun ≥ 1.2.14. Run `bun upgrade`. |
| App boots but sign-in fails | Check `NEXT_PUBLIC_VERCEL_APP_CLIENT_ID` and `VERCEL_APP_CLIENT_SECRET`. Verify the OAuth callback URL matches exactly. |
| GitHub features unavailable | Ensure all `GITHUB_*` env vars are set and the GitHub App is installed on at least one repository. |
| Database migrations fail | Verify `POSTGRES_URL` is correct and the database user has `CREATE TABLE` permission. |
| Sandbox does not start | `VERCEL_SANDBOX_BASE_SNAPSHOT_ID` may need refreshing. Run `bun run sandbox:snapshot-base`. |
| Type errors after pulling | Run `bun install` first, then `bun run typecheck`. |
