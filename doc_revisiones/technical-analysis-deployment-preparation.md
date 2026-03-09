# Prompt Database - Technical Analysis and Deployment Preparation Guide

**Document Version:** 2.0
**Analysis Date:** March 9, 2026
**Analyst Role:** Technical Analysis Specialist, Documentation and Deployment Preparation
**Review Status:** Updated with verified official documentation (March 2026)

---

## Document Revision Notes

This document has been reviewed and updated against official documentation from:
- Vercel (vercel.com/docs)
- Next.js (nextjs.org/docs)
- Prisma ORM (prisma.io/docs)
- Neon (neon.tech/docs)

Key changes in this revision:
- Confirmed Vercel Postgres discontinuation (December 2024) and migration to Neon
- Updated Vercel deployment requirements (SQLite incompatible with serverless)
- Added Next.js 15 breaking changes awareness
- Corrected database recommendations for serverless deployments

---

## Executive Summary

**Prompt Database** is a full-stack web application for managing and organizing AI prompts. The project is built on **Next.js 14 (App Router)** with **TypeScript**, using **SQLite** as the database with **Prisma ORM** for data access.

### Key Findings

| Aspect | Status | Notes |
|--------|--------|-------|
| **Code Quality** | ✅ Good | TypeScript strict mode, ESLint, Prettier configured |
| **Testing** | ⚠️ Basic | Jest configured with limited test coverage |
| **Docker Support** | ✅ Production-ready | Multi-stage build, optimized images |
| **Database** | ⚠️ SQLite only | Suitable for single-instance; **NOT compatible with Vercel serverless** |
| **Deployment Scripts** | ✅ Present | deploy.sh, setup-server.sh, docker-entrypoint.sh |
| **CI/CD** | ❌ Not detected | No GitHub Actions, GitLab CI, or similar |
| **Environment Config** | ⚠️ Minimal | No `.env.example` file found |
| **Documentation** | ✅ Good | README.md, DOCKER.md, DEPLOYMENT.md present |
| **Vercel Compatibility** | ❌ Not compatible | Requires database migration to PostgreSQL + config changes |

### Deployment Readiness Assessment

The project demonstrates **moderate-to-good deployment readiness** for single-server or small-scale deployments. Key limitations include:

1. **SQLite database** - Not suitable for high-concurrency or distributed deployments; **NOT compatible with Vercel serverless**
2. **Hardcoded server IP** - Deployment scripts reference specific server (46.224.72.129)
3. **No CI/CD pipeline** - Manual deployment process via git push to remote server
4. **Limited monitoring** - Basic health checks only, no metrics or alerting
5. **No secrets management** - Environment variables in docker-compose.yml, not externalized
6. **Vercel incompatibility** - Requires database migration (SQLite → PostgreSQL) and configuration changes

---

## Project General Description

### Purpose

Prompt Database is designed to help users:
- Create, edit, and delete AI prompts
- Organize prompts with hierarchical categories and tags
- Filter and search prompts by multiple criteria (category, tags, platform, status, language)
- Track prompt usage (count and last used date)
- Export/import prompts in JSON format
- Mark prompts as favorites
- Copy prompts to clipboard with automatic usage tracking

### Target Users

- Developers working with AI assistants (Cursor, ChatGPT)
- Teams managing prompt libraries
- AI prompt engineers and researchers

### Core Features

| Feature | Description |
|---------|-------------|
| CRUD Operations | Full create, read, update, delete for prompts, categories, and tags |
| Hierarchical Categories | Tree-structured category organization |
| Multi-tag Support | Many-to-many relationship between prompts and tags |
| Full-text Search | Search across title, description, and body |
| Filtering | By category, tags, platform, status, language, favorites |
| Usage Tracking | Automatic count and timestamp tracking |
| Export/Import | JSON format for backup and migration |
| Favorite System | Quick access to frequently used prompts |

---

## Functional and Technical Code Analysis

### Application Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Browser/Client                        │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP/HTTPS
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Reverse Proxy (nginx / Traefik)            │
│              - Path-based routing                        │
│              - SSL termination (optional)                │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Next.js Application (Port 3000)            │
│  ┌──────────────────────────────────────────────────┐  │
│  │  App Router (app/)                               │  │
│  │  ├── layout.tsx (Root layout)                    │  │
│  │  ├── page.tsx (Home → redirects to /prompts)     │  │
│  │  ├── api/ (API Routes)                           │  │
│  │  │   ├── prompts/route.ts                        │  │
│  │  │   ├── categories/route.ts                     │  │
│  │  │   ├── tags/route.ts                           │  │
│  │  │   ├── export/prompts/route.ts                 │  │
│  │  │   └── import/prompts/route.ts                 │  │
│  │  ├── prompts/ (Prompt pages)                     │  │
│  │  ├── categories/ (Category pages)                │  │
│  │  └── tags/ (Tag pages)                           │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Components                                      │  │
│  │  ├── layout/ (Topbar, Sidebar)                   │  │
│  │  ├── prompt/ (PromptList, PromptForm, Filters)   │  │
│  │  └── ui/ (shadcn/ui components)                  │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  lib/                                            │  │
│  │  ├── prisma.ts (Database client)                 │  │
│  │  └── utils.ts (Utility functions)                │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────┬────────────────────────────────────┘
                     │ Prisma Client
                     ▼
┌─────────────────────────────────────────────────────────┐
│              SQLite Database (prod.db / dev.db)         │
│              Location: /app/data/ (Docker)              │
└─────────────────────────────────────────────────────────┘
```

### Code Structure Analysis

#### 1. **Next.js App Router Pattern**

The project uses Next.js 14's App Router (`app/` directory), which is the modern approach for Next.js applications.

**Key Files:**
- `app/layout.tsx` - Root layout with Inter font, Topbar, and Sidebar
- `app/page.tsx` - Home page that redirects to `/prompts`
- `app/globals.css` - Global styles with TailwindCSS

**API Routes Structure:**
```
app/api/
├── prompts/
│   ├── route.ts (GET, POST)
│   ├── [id]/
│   │   └── route.ts (GET, PUT, DELETE)
│   └── [id]/usage/
│       └── route.ts (PATCH - track usage)
├── categories/
│   ├── route.ts (GET, POST)
│   └── [id]/route.ts (PUT, DELETE)
├── tags/
│   ├── route.ts (GET, POST)
│   └── [id]/route.ts (PUT, DELETE)
├── export/
│   └── prompts/route.ts (GET - export JSON)
└── import/
    └── prompts/route.ts (POST - import JSON)
```

#### 2. **Database Schema (Prisma)**

**Location:** `prisma/schema.prisma`

**Models:**

| Model | Description | Key Fields |
|-------|-------------|------------|
| `Prompt` | Main entity for AI prompts | id, title, body, type, platform, language, useCase, status, categoryId, usageCount |
| `Category` | Hierarchical category structure | id, name, slug, parentId (self-relation), sortOrder |
| `Tag` | Tags for organizing prompts | id, name, slug |
| `PromptTag` | Join table for many-to-many | promptId, tagId (composite key) |

**Important Schema Notes:**

```prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["native", "linux-musl-openssl-3.0.x", "linux-musl-arm64-openssl-3.0.x"]
}
```

The `binaryTargets` configuration ensures Prisma works in Docker containers on both x86_64 and ARM64 architectures.

**Enum Handling:**
SQLite doesn't support native enums, so the schema uses String fields with `@default` values:
- `type`: "SYSTEM", "USER", "TOOL"
- `platform`: "CHATGPT", "CURSOR", "MIDJOURNEY", "SUNO", "OTHER"
- `status`: "DRAFT", "TESTED", "PRODUCTION"

#### 3. **API Implementation Pattern**

All API routes follow a consistent pattern:

```typescript
import { NextRequest, NextResponse } from "next/server"
import { prisma } from "@/lib/prisma"
import { z } from "zod"

// Zod schema for validation
const schema = z.object({...})

export async function GET(request: NextRequest) {
  try {
    // Query logic
    return NextResponse.json(data)
  } catch (error) {
    return NextResponse.json(
      { error: "Error message" },
      { status: 500 }
    )
  }
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const data = schema.parse(body)
    // Mutation logic
    return NextResponse.json(data, { status: 201 })
  } catch (error) {
    // Error handling
  }
}
```

**Validation:** Uses Zod for runtime type validation with detailed error responses.

**Error Handling:** Consistent try-catch blocks with appropriate HTTP status codes.

#### 4. **Component Architecture**

**Layout Components:**
- `components/layout/Topbar.tsx` - Top navigation with search, export/import
- `components/layout/Sidebar.tsx` - Sidebar with filters (category, tags, platform, status)

**Prompt Components:**
- `components/prompt/PromptList.tsx` - Display prompts in list/grid
- `components/prompt/PromptForm.tsx` - Create/edit form
- `components/prompt/PromptFilters.tsx` - Filter controls

**UI Components (shadcn/ui):**
- button.tsx, input.tsx, textarea.tsx, select.tsx
- dialog.tsx, card.tsx, badge.tsx, label.tsx

#### 5. **Utility Functions**

**`lib/prisma.ts`:**
```typescript
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

**Purpose:** Prevents multiple Prisma client instances in development (hot reload).

**`lib/utils.ts`:**
```typescript
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

export function getApiUrl(path: string): string {
  const basePath = process.env.NEXT_PUBLIC_BASE_PATH || '/prompt-database'
  const cleanPath = path.startsWith('/') ? path : `/${path}`
  return `${basePath}${cleanPath}`
}
```

**Purpose:** 
- `cn()` - TailwindCSS class merging utility
- `getApiUrl()` - API URL construction with basePath support

---

## Project Structure

```
/workspaces/prompt-database-1/
├── .dockerignore              # Docker ignore patterns
├── .eslintrc.json            # ESLint configuration (Next.js core-web-vitals)
├── .gitignore                # Git ignore patterns
├── .prettierrc               # Prettier formatting rules
├── deploy.sh                 # Deployment script for remote server
├── DEPLOYMENT.md             # Deployment instructions
├── docker-compose.dev.yml    # Docker Compose for development
├── docker-compose.yml        # Docker Compose for production
├── docker-entrypoint.sh      # Container entrypoint script
├── DOCKER.md                 # Docker usage guide
├── Dockerfile                # Production Docker image
├── Dockerfile.dev            # Development Docker image
├── jest.config.js            # Jest test configuration
├── jest.setup.js             # Jest setup file
├── next.config.js            # Next.js configuration
├── nginx.conf                # Nginx reverse proxy configuration
├── package.json              # Node.js dependencies and scripts
├── package-lock.json         # Locked dependency versions
├── postcss.config.js         # PostCSS configuration
├── README.md                 # Project documentation
├── setup-server.sh           # Server setup script
├── start-docker.sh           # Local Docker startup script
├── tailwind.config.ts        # TailwindCSS configuration
├── traefik-prompt-database.yml  # Traefik routing configuration
├── tsconfig.json             # TypeScript configuration
│
├── app/                      # Next.js App Router
│   ├── api/                  # API routes
│   │   ├── categories/       # Category endpoints
│   │   ├── export/           # Export endpoint
│   │   ├── import/           # Import endpoint
│   │   ├── prompts/          # Prompt endpoints
│   │   └── tags/             # Tag endpoints
│   ├── categories/           # Category pages
│   ├── prompts/              # Prompt pages
│   ├── tags/                 # Tag pages
│   ├── globals.css           # Global styles
│   ├── layout.tsx            # Root layout
│   ├── not-found.tsx         # 404 page
│   └── page.tsx              # Home page
│
├── components/               # React components
│   ├── layout/               # Layout components
│   │   ├── Sidebar.tsx
│   │   └── Topbar.tsx
│   ├── prompt/               # Prompt-related components
│   │   ├── PromptFilters.tsx
│   │   ├── PromptForm.tsx
│   │   └── PromptList.tsx
│   └── ui/                   # shadcn/ui components
│       ├── badge.tsx
│       ├── button.tsx
│       ├── card.tsx
│       ├── dialog.tsx
│       ├── input.tsx
│       ├── label.tsx
│       ├── select.tsx
│       └── textarea.tsx
│
├── lib/                      # Utility modules
│   ├── prisma.ts             # Prisma client singleton
│   └── utils.ts              # Utility functions
│
├── prisma/                   # Prisma ORM
│   ├── schema.prisma         # Database schema
│   └── seed.ts               # Database seeding script
│
├── public/                   # Static assets
│
├── tests/                    # Test files
│   ├── api/                  # API tests
│   │   └── prompts.test.ts
│   └── components/           # Component tests
│
└── doc_revisiones/           # Documentation revisions (created for this analysis)
```

---

## Technologies and Dependencies Identified

### Core Technologies

| Technology | Version | Purpose | Official Documentation |
|------------|---------|---------|----------------------|
| **Next.js** | 14.2.5 | React framework (App Router) | https://nextjs.org/docs |
| **React** | 18.3.1 | UI library | https://react.dev |
| **TypeScript** | 5.5.4 | Type-safe JavaScript | https://www.typescriptlang.org/docs |
| **Node.js** | 20.x (required) | Runtime environment | https://nodejs.org/docs |

**Note:** Next.js 15 was released with breaking changes (async request APIs, fetch not cached by default). The project uses Next.js 14.2.5 which is stable and supported. Upgrading to Next.js 15 requires code changes. See: https://nextjs.org/docs/app/building-your-application/upgrading/version-15

### Database & ORM

| Technology | Version | Purpose | Official Documentation |
|------------|---------|---------|----------------------|
| **Prisma** | 5.19.1 | Database ORM | https://www.prisma.io/docs |
| **SQLite** | Built-in | Database engine | https://www.sqlite.org/docs |

**⚠️ Critical Note for Vercel Deployment:**

SQLite is **NOT compatible** with Vercel's serverless architecture:
- Vercel serverless functions have a **read-only file system** (except `/tmp` which is ephemeral)
- SQLite requires file write access for the database file and write-ahead log
- Multiple serverless instances cannot share a single SQLite file

**Prisma's recommended databases for serverless deployments:**
- Neon Serverless Postgres (recommended for Vercel)
- PlanetScale (MySQL-compatible)
- AWS Aurora Serverless
- CockroachDB Serverless

Source: https://www.prisma.io/docs/orm/overview/databases

### Styling & UI

| Technology | Version | Purpose | Official Documentation |
|------------|---------|---------|----------------------|
| **TailwindCSS** | 3.4.7 | Utility-first CSS | https://tailwindcss.com/docs |
| **shadcn/ui** | Latest | UI component library | https://ui.shadcn.com/docs |
| **Radix UI** | 1.x | Accessible primitives | https://www.radix-ui.com/docs |
| **lucide-react** | 0.427.0 | Icon library | https://lucide.dev/guide |

### Validation & Utilities

| Technology | Version | Purpose | Official Documentation |
|------------|---------|---------|----------------------|
| **Zod** | 3.23.8 | Schema validation | https://zod.dev |
| **clsx** | 2.1.1 | Class name utility | https://github.com/lukeed/clsx |
| **tailwind-merge** | 2.5.2 | Tailwind class merging | https://github.com/dcastil/tailwind-merge |
| **class-variance-authority** | 0.7.0 | Class variance utilities | https://cva.style |

### Development & Testing

| Technology | Version | Purpose | Official Documentation |
|------------|---------|---------|----------------------|
| **Jest** | 29.7.0 | Testing framework | https://jestjs.io/docs |
| **Testing Library** | 16.0.0 | React testing utilities | https://testing-library.com/docs |
| **ESLint** | 8.57.0 | Code linting | https://eslint.org/docs |
| **Prettier** | Configured | Code formatting | https://prettier.io/docs |
| **Autoprefixer** | 10.4.19 | CSS vendor prefixes | https://github.com/postcss/autoprefixer |
| **PostCSS** | 8.4.40 | CSS processing | https://postcss.org/docs |
| **tsx** | 4.16.2 | TypeScript execution | https://github.com/esbuild-kit/tsx |

### Containerization & Deployment

| Technology | Version | Purpose | Official Documentation |
|------------|---------|---------|----------------------|
| **Docker** | Latest | Container runtime | https://docs.docker.com |
| **Docker Compose** | 2.x | Multi-container orchestration | https://docs.docker.com/compose |
| **nginx** | Latest | Reverse proxy | https://nginx.org/en/docs |
| **Traefik** | Latest | Reverse proxy (alternative) | https://doc.traefik.io/traefik |

### Package Dependencies (package.json)

**Production Dependencies:**
```json
{
  "@prisma/client": "^5.19.1",
  "@radix-ui/react-dialog": "^1.1.1",
  "@radix-ui/react-dropdown-menu": "^2.1.1",
  "@radix-ui/react-label": "^2.1.0",
  "@radix-ui/react-select": "^2.1.0",
  "@radix-ui/react-slot": "^1.1.0",
  "@radix-ui/react-tabs": "^1.1.0",
  "class-variance-authority": "^0.7.0",
  "clsx": "^2.1.1",
  "lucide-react": "^0.427.0",
  "next": "14.2.5",
  "react": "^18.3.1",
  "react-dom": "^18.3.1",
  "tailwind-merge": "^2.5.2",
  "tailwindcss-animate": "^1.0.7",
  "zod": "^3.23.8"
}
```

**Development Dependencies:**
```json
{
  "@testing-library/jest-dom": "^6.4.2",
  "@testing-library/react": "^16.0.0",
  "@testing-library/user-event": "^14.5.2",
  "@types/jest": "^29.5.12",
  "@types/node": "^20.14.12",
  "@types/react": "^18.3.3",
  "@types/react-dom": "^18.3.0",
  "autoprefixer": "^10.4.19",
  "eslint": "^8.57.0",
  "eslint-config-next": "14.2.5",
  "jest": "^29.7.0",
  "jest-environment-jsdom": "^29.7.0",
  "postcss": "^8.4.40",
  "prisma": "^5.19.1",
  "tailwindcss": "^3.4.7",
  "tsx": "^4.16.2",
  "typescript": "^5.5.4"
}
```

---

## Deployment Readiness and Compatibility Analysis

### Current Deployment Configuration

#### 1. **Docker Production Setup**

**File:** `Dockerfile`

**Build Strategy:** Multi-stage build for optimized image size

**Stages:**
1. `base` - Node.js 20 Alpine base
2. `deps` - Install dependencies
3. `builder` - Generate Prisma Client and build Next.js
4. `runner` - Production runtime image

**Key Features:**
```dockerfile
# Output: 'standalone' in next.config.js enables minimal production build
output: 'standalone'

# Prisma binary targets for multi-architecture support
ENV PRISMA_CLI_BINARY_TARGETS=linux-musl-openssl-3.0.x,linux-musl-arm64-openssl-3.0.x

# Non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Health check endpoint
healthcheck:
  test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/prompt-database/api/prompts"]
```

**Assessment:** ✅ Well-configured for production

---

#### 2. **Docker Compose Production**

**File:** `docker-compose.yml`

**Configuration:**
```yaml
services:
  app:
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=file:/app/data/prod.db
      - NODE_ENV=production
      - NEXT_PUBLIC_BASE_PATH=/prompt-database
    volumes:
      - ./data:/app/data
    networks:
      - web  # External (Traefik)
      - prompt-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prompt-db.rule=PathPrefix(`/prompt-database`)"
```

**Assessment:** ⚠️ Configured for specific deployment (Traefik, hardcoded paths)

---

#### 3. **Reverse Proxy Configurations**

**nginx Configuration (`nginx.conf`):**
```nginx
server {
    listen 80;
    server_name 46.224.72.129;

    location /prompt-database {
        proxy_pass http://localhost:3000;
        # ... proxy headers
    }
}
```

**Traefik Configuration (`traefik-prompt-database.yml`):**
```yaml
http:
  routers:
    prompt-database-http:
      rule: "PathPrefix(`/prompt-database`)"
      entryPoints:
        - web
```

**Assessment:** ⚠️ Hardcoded IP address (46.224.72.129), needs parameterization

---

### Deployment Compatibility Assessment

#### ✅ Compatible Deployment Scenarios

| Scenario | Compatibility | Notes |
|----------|---------------|-------|
| **Single-server Docker** | ✅ Excellent | Designed for this use case |
| **Local development** | ✅ Excellent | docker-compose.dev.yml provided |
| **Subfolder deployment** | ✅ Excellent | basePath configured |
| **SQLite-compatible hosts** | ✅ Excellent | Any host with file system |
| **VPS (Ubuntu/Debian)** | ✅ Excellent | Scripts tested on Ubuntu |
| **Railway, Render, Fly.io** | ✅ Excellent | These platforms support SQLite |

#### ⚠️ Requires Adaptation

| Scenario | Compatibility | Required Changes |
|----------|---------------|------------------|
| **Multi-instance deployment** | ⚠️ Low | Replace SQLite with PostgreSQL/MySQL |
| **Kubernetes** | ⚠️ Medium | Create K8s manifests, Helm chart |
| **Cloud-native (AWS, GCP, Azure)** | ⚠️ Medium | Externalize config, use managed DB |
| **CI/CD integration** | ⚠️ Medium | Add GitHub Actions/GitLab CI workflows |

#### ❌ Not Compatible (Without Major Changes)

| Scenario | Issue |
|----------|-------|
| **Vercel serverless** | SQLite not supported; requires PostgreSQL migration + config changes |
| **Edge computing** | Next.js standalone doesn't support edge runtime with Prisma |
| **High-concurrency production** | SQLite file locking limits concurrent writes |
| **Distributed deployments** | SQLite is single-file, not shareable across instances |

---

### Critical Deployment Dependencies

#### 1. **Database Considerations**

**Current:** SQLite (file-based)

**Limitations (per SQLite documentation):**
- Single writer at a time (file locking)
- Not suitable for high-write workloads
- Database file must be on shared storage for multi-instance

**⚠️ Vercel Serverless Incompatibility:**

Vercel serverless functions have a **read-only file system**. SQLite cannot be used because:
1. SQLite requires write access to the database file (`prod.db`)
2. SQLite creates a write-ahead log file (`prod.db-journal`) during operations
3. The `/tmp` directory is writable but **ephemeral** (data deleted after function execution)
4. Multiple serverless instances cannot share a single SQLite file

Source: https://vercel.com/docs/functions/serverless-functions

**Recommendation for Vercel Deployment:**
Migrate to PostgreSQL using a serverless-compatible provider:

```prisma
datasource db {
  provider = "postgresql"  // or "mysql"
  url      = env("DATABASE_URL")
}
```

**Prisma's Recommended Serverless Databases:**
- **Neon Serverless Postgres** - Recommended for Vercel (auto-integrates via Marketplace)
- **PlanetScale** - MySQL-compatible, serverless branching
- **Supabase** - PostgreSQL with real-time features
- **AWS Aurora Serverless** - PostgreSQL/MySQL-compatible

Source: https://www.prisma.io/docs/orm/overview/databases

---

#### 2. **Environment Variables**

**Required Variables:**

| Variable | Default | Required For |
|----------|---------|--------------|
| `DATABASE_URL` | `file:/app/data/prod.db` | Database connection |
| `NODE_ENV` | `production` | Runtime mode |
| `NEXT_PUBLIC_BASE_PATH` | `/prompt-database` | Subfolder routing |

**Missing `.env.example` File:**

The project references `.env.example` in README.md but the file doesn't exist. This should be created:

```bash
# .env.example
DATABASE_URL="file:./dev.db"
NODE_ENV=development
NEXT_PUBLIC_BASE_PATH=/prompt-database
```

---

#### 3. **Next.js Configuration**

**File:** `next.config.js`

```javascript
const nextConfig = {
  reactStrictMode: true,
  output: 'standalone',  // Creates minimal server
  basePath: process.env.NEXT_PUBLIC_BASE_PATH || '/prompt-database',
  assetPrefix: process.env.NEXT_PUBLIC_BASE_PATH || '/prompt-database',
}
```

**Implications:**
- `output: 'standalone'` - Creates minimal Node.js server in `.next/standalone`
- `basePath` - All routes prefixed with `/prompt-database`
- This configuration is **fixed at build time** - changing basePath requires rebuild

---

### Build and Runtime Requirements

#### Minimum Requirements

| Resource | Requirement |
|----------|-------------|
| **Node.js** | 20.x or higher |
| **RAM** | 512 MB minimum, 1 GB recommended |
| **Disk** | 500 MB for application + database size |
| **CPU** | 1 core minimum |

#### Build Requirements

| Resource | Requirement |
|----------|-------------|
| **RAM** | 2 GB recommended (Next.js build is memory-intensive) |
| **Disk** | 1 GB for node_modules and build artifacts |

---

## Detected Deployment Tools, Scripts, and Mechanisms

### 1. **Deployment Scripts**

#### `deploy.sh` - Remote Deployment Script

**Purpose:** Deploy code to remote server (46.224.72.129)

**Functionality:**
```bash
# Push to remote git repository
git push hertzner main

# SSH to server and:
# 1. Pull latest code
# 2. Rebuild containers
# 3. Run migrations
# 4. Restart application
```

**Usage:**
```bash
./deploy.sh --key=/path/to/private/key
./deploy.sh --setup --key=/path/to/private/key
```

**Assessment:** ⚠️ Hardcoded server IP, requires git remote "hertzner"

---

#### `setup-server.sh` - Server Initialization

**Purpose:** Initial server setup

**Functionality:**
1. Install nginx
2. Configure reverse proxy
3. Install Docker and Docker Compose
4. Build and start containers
5. Initialize database

**Assessment:** ✅ Comprehensive setup script

---

#### `start-docker.sh` - Local Docker Startup

**Purpose:** Quick local development setup

**Functionality:**
1. Check Docker is running
2. Build and start containers
3. Run migrations
4. Optional database seeding

**Assessment:** ✅ Useful for local development

---

#### `docker-entrypoint.sh` - Container Entrypoint

**Purpose:** Initialize container on startup

**Functionality:**
```bash
#!/bin/sh
# 1. Create data directory
# 2. Initialize database if not exists
# 3. Fix permissions
# 4. Switch to non-root user
# 5. Execute main command
```

**Assessment:** ✅ Follows best practices (non-root user, permission handling)

---

### 2. **Docker Configurations**

#### Production: `Dockerfile` + `docker-compose.yml`

**Features:**
- Multi-stage build
- Non-root user
- Health checks
- Volume persistence
- Traefik labels

#### Development: `Dockerfile.dev` + `docker-compose.dev.yml`

**Features:**
- Hot reload (volume mounts)
- Development database
- Direct port exposure (3300)

---

### 3. **NPM Scripts**

**From package.json:**

| Script | Command | Purpose |
|--------|---------|---------|
| `dev` | `next dev` | Start development server |
| `build` | `next build` | Build for production |
| `start` | `next start` | Start production server |
| `lint` | `next lint` | Run ESLint |
| `db:push` | `prisma db push` | Push schema to DB (dev) |
| `db:migrate` | `prisma migrate dev` | Run migrations (dev) |
| `db:generate` | `prisma generate` | Generate Prisma Client |
| `db:seed` | `tsx prisma/seed.ts` | Seed database |
| `test` | `jest` | Run tests |
| `test:watch` | `jest --watch` | Run tests in watch mode |

---

### 4. **Testing Infrastructure**

**Configuration:**
- Jest with Next.js integration
- Testing Library for React components
- Test environment: jsdom for components, node for API tests

**Test Files:**
- `tests/api/prompts.test.ts` - API endpoint tests
- `tests/components/` - Component tests (directory exists, contents not analyzed)

**Assessment:** ⚠️ Limited test coverage (only prompts API tested)

---

### 5. **CI/CD Status**

**Current State:** ❌ No CI/CD pipeline detected

**Evidence:**
- No `.github/workflows/` directory
- No `.gitlab-ci.yml`
- No `Jenkinsfile`
- Deployment is manual via `deploy.sh`

**Recommendation:** Implement CI/CD for:
- Automated testing
- Build verification
- Deployment automation
- Security scanning

---

## Risks, Limitations, and Detected Issues

### 🔴 Critical Risks

#### 1. **SQLite for Production**

**Risk:** Database locking and concurrency issues

**Impact:**
- Single writer at a time
- Potential data corruption under high write load
- Not suitable for multi-instance deployment

**Mitigation:** Migrate to PostgreSQL for production

---

#### 2. **Hardcoded Server Configuration**

**Risk:** Deployment scripts reference specific server (46.224.72.129)

**Files Affected:**
- `deploy.sh`
- `setup-server.sh`
- `DEPLOYMENT.md`
- `nginx.conf`

**Impact:** Scripts not reusable for other deployments

**Mitigation:** Use environment variables or configuration files

---

#### 3. **Missing `.env.example` File**

**Risk:** Unclear environment variable requirements

**Impact:** Deployment may fail due to missing configuration

**Mitigation:** Create `.env.example` with all required variables

---

### ⚠️ Moderate Limitations

#### 4. **No CI/CD Pipeline**

**Risk:** Manual deployment process, no automated testing

**Impact:**
- Human error in deployment
- No automated quality gates
- Slower deployment cycles

**Mitigation:** Implement GitHub Actions or similar

---

#### 5. **Limited Test Coverage**

**Risk:** Only API prompts tested

**Impact:**
- Undetected regressions
- Component behavior not verified
- Integration issues may go unnoticed

**Mitigation:** Expand test coverage to all critical paths

---

#### 6. **Fixed basePath at Build Time**

**Risk:** `NEXT_PUBLIC_BASE_PATH` is baked into build

**Impact:** Cannot change deployment path without rebuild

**Mitigation:** Document requirement clearly; consider runtime configuration

---

#### 7. **No Secrets Management**

**Risk:** Sensitive data in environment variables or config files

**Impact:**
- Credentials may be exposed
- No rotation mechanism
- Difficult multi-environment management

**Mitigation:** Use secrets management (Vault, AWS Secrets Manager, etc.)

---

#### 8. **No Monitoring or Alerting**

**Risk:** Limited visibility into application health

**Current:** Basic Docker health check only

**Impact:**
- Issues may go undetected
- No performance metrics
- Reactive rather than proactive operations

**Mitigation:** Add monitoring (Prometheus, DataDog, etc.) and alerting

---

### ℹ️ Minor Issues

#### 9. **Documentation Inconsistencies**

**Issue:** README mentions `.env.example` but file doesn't exist

**Impact:** Confusion during setup

---

#### 10. **Mixed Language in Documentation**

**Issue:** Some documentation in Dutch (DOCKER.md comments)

**Impact:** May confuse non-Dutch speakers

---

#### 11. **No API Documentation**

**Issue:** API endpoints not documented (OpenAPI/Swagger)

**Impact:** Difficult for external integration

**Mitigation:** Add OpenAPI specification

---

#### 12. **No Rate Limiting**

**Issue:** API endpoints have no rate limiting

**Impact:** Potential for abuse or DoS

**Mitigation:** Add rate limiting middleware

---

## Pre-deployment Requirements

### Technical Prerequisites

#### 1. **Runtime Environment**

| Requirement | Details |
|-------------|---------|
| **Node.js** | Version 20.x or higher |
| **Package Manager** | npm, yarn, or pnpm |
| **Operating System** | Linux (tested), macOS, Windows (WSL recommended) |
| **Memory** | Minimum 512 MB, recommended 1 GB |
| **Disk Space** | 500 MB minimum |

---

#### 2. **Docker Environment (Recommended)**

| Requirement | Details |
|-------------|---------|
| **Docker** | Version 20.10 or higher |
| **Docker Compose** | Version 2.0 or higher |
| **Disk Space** | 1 GB for images and containers |

---

#### 3. **Database**

**Current (Development/Small-scale):**
- SQLite (file-based, no setup required)

**For Production (Recommended):**
- PostgreSQL 12+ or MySQL 8+
- Database user with CREATE, ALTER permissions
- Connection string for `DATABASE_URL`

---

#### 4. **Reverse Proxy (Optional)**

**Options:**
- nginx (configured in `nginx.conf`)
- Traefik (configured in `traefik-prompt-database.yml`)
- Caddy
- HAProxy

**Requirements:**
- Domain name (for SSL)
- SSL certificates (Let's Encrypt recommended)
- Port 80/443 access

---

### User-Provided Resources and Configuration

The following items **must be created, configured, or confirmed** by the user before deployment:

---

#### 🔐 1. **SSH Key for Server Access**

**What:** SSH private key for server authentication

**Where to Obtain:**
- Generate new: `ssh-keygen -t ed25519 -C "your_email@example.com"`
- Or use existing key

**Configuration:**
- Store in `.ssh/hertzner_key` (referenced in `deploy.sh`)
- Add public key to server's `~/.ssh/authorized_keys`

**Purpose:** Secure deployment to remote server

**Action Required:** ✅ Create/configure SSH key

---

#### 🔐 2. **Environment Variables File**

**What:** `.env` file with deployment-specific values

**Where:** Project root

**Template (create `.env.example` first):**
```bash
# Database
DATABASE_URL="file:./dev.db"
# For PostgreSQL:
# DATABASE_URL="postgresql://user:password@localhost:5432/promptdb?schema=public"

# Runtime
NODE_ENV=production

# Application
NEXT_PUBLIC_BASE_PATH=/prompt-database
```

**Purpose:** Configure application for specific environment

**Action Required:** ✅ Create `.env` and `.env.example` files

---

#### 🌐 3. **Domain and SSL Configuration**

**What:** Domain name and SSL certificates (for production)

**Options:**
- Use existing domain
- Obtain free SSL from Let's Encrypt
- Use cloud provider's SSL service

**Configuration:**
- Update `nginx.conf` with domain
- Configure SSL certificates
- Update `NEXT_PUBLIC_BASE_PATH` if using different path

**Purpose:** Secure HTTPS access

**Action Required:** ⚠️ Required for production, optional for development

---

#### 🗄️ 4. **Database Backup Strategy**

**What:** Backup mechanism for database

**Current:** Manual backup via `cp data/prod.db`

**Recommended:**
- Automated backup script
- Remote backup storage
- Backup rotation policy

**Purpose:** Data recovery in case of failure

**Action Required:** ⚠️ Implement automated backup for production

---

#### 📊 5. **Monitoring and Logging**

**What:** Application monitoring and log aggregation

**Options:**
- Basic: Docker logs + log rotation
- Advanced: Prometheus + Grafana
- Cloud: DataDog, New Relic, AWS CloudWatch

**Purpose:** Operational visibility and alerting

**Action Required:** ⚠️ Recommended for production

---

#### 🔧 6. **Git Remote Configuration**

**What:** Git remote for deployment

**Current:** Remote named "hertzner" (in `deploy.sh`)

**Configuration:**
```bash
git remote add hertzner ssh://root@46.224.72.129/opt/prompt-database
```

**Purpose:** Git-based deployment

**Action Required:** ✅ Configure or update remote name in scripts

---

#### 📝 7. **Server Access and Permissions**

**What:** SSH access to deployment server

**Requirements:**
- Root or sudo access
- Docker installation (or run `setup-server.sh`)
- Port 80/443 open (for nginx/Traefik)
- Port 3000 accessible (for direct access)

**Purpose:** Deploy and run application

**Action Required:** ✅ Ensure server access

---

#### 🏗️ 8. **Build Configuration Review**

**What:** Review and adjust build configuration

**Files to Review:**
- `next.config.js` - basePath, output mode
- `docker-compose.yml` - ports, volumes, networks
- `nginx.conf` - server_name, locations

**Purpose:** Ensure configuration matches deployment target

**Action Required:** ✅ Review and adjust configuration

---

#### 📦 9. **Container Registry (Optional)**

**What:** Docker image registry for deployment

**Options:**
- Docker Hub
- GitHub Container Registry
- AWS ECR
- Google Container Registry
- Azure Container Registry

**Purpose:** Store and distribute Docker images

**Action Required:** ⚠️ Optional (current deployment builds on server)

---

#### 🔑 10. **API Keys and External Services**

**What:** Any external service credentials

**Current:** None detected in codebase

**Future Considerations:**
- Email service (for notifications)
- Analytics (Google Analytics, etc.)
- Authentication providers

**Purpose:** External service integration

**Action Required:** ℹ️ Not currently required, plan for future

---

### Configuration Checklist

Before deployment, ensure the following:

- [ ] SSH key generated and configured
- [ ] `.env` file created with correct values
- [ ] `.env.example` file created for documentation
- [ ] Domain and SSL configured (production only)
- [ ] Database backup strategy implemented
- [ ] Git remote configured
- [ ] Server access verified
- [ ] Build configuration reviewed
- [ ] Monitoring/logging planned (production)
- [ ] Container registry configured (optional)

---

## Recommended Next Steps

### Immediate Actions (Required for Deployment)

#### 1. **Create Missing Configuration Files**

**Priority:** 🔴 Critical

**Actions:**
```bash
# Create .env.example
cat > .env.example << EOF
DATABASE_URL="file:./dev.db"
NODE_ENV=development
NEXT_PUBLIC_BASE_PATH=/prompt-database
EOF

# Create .env from example
cp .env.example .env
```

---

#### 2. **Parameterize Deployment Scripts**

**Priority:** 🔴 Critical

**Actions:**
- Replace hardcoded IP (46.224.72.129) with environment variable
- Make SSH key path configurable
- Document required git remote

**Example:**
```bash
# deploy.sh - Add at top
SERVER="${SERVER_HOST:-root@46.224.72.129}"
SSH_KEY="${SSH_KEY_PATH:-${SCRIPT_DIR}/.ssh/hertzner_key}"
```

---

#### 3. **Review and Test Docker Deployment**

**Priority:** 🔴 Critical

**Actions:**
```bash
# Test local deployment
docker-compose down -v
docker-compose up -d --build
docker-compose exec -T app npx prisma migrate deploy
docker-compose logs -f app
```

**Verify:**
- Application starts without errors
- Health check passes
- Database persists across restarts

---

#### 4. **Expand Test Coverage**

**Priority:** 🟡 Important

**Actions:**
- Add tests for all API endpoints
- Add component tests
- Add integration tests
- Set up test coverage reporting

---

### Short-term Improvements (Recommended)

#### 5. **Implement CI/CD Pipeline**

**Priority:** 🟡 Important

**Recommended:** GitHub Actions

**Workflow:**
```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to server
        run: ./deploy.sh
```

---

#### 6. **Database Migration Plan**

**Priority:** 🟡 Important (for production scale)

**Actions:**
1. Evaluate PostgreSQL migration need
2. Update Prisma schema
3. Create migration script
4. Test with production-like data
5. Document migration process

---

#### 7. **Add Monitoring**

**Priority:** 🟡 Important

**Minimum:**
- Enhanced health check endpoint
- Log aggregation
- Error tracking (Sentry)

**Recommended:**
- Prometheus metrics
- Grafana dashboards
- Alerting rules

---

### Long-term Enhancements (Optional)

#### 8. **Kubernetes Support**

**Priority:** 🟢 Optional

**Actions:**
- Create Kubernetes manifests
- Develop Helm chart
- Document K8s deployment
- Test scaling scenarios

---

#### 9. **API Documentation**

**Priority:** 🟢 Optional

**Actions:**
- Add OpenAPI/Swagger specification
- Generate API documentation
- Add API versioning strategy

---

#### 10. **Security Hardening**

**Priority:** 🟢 Optional (but recommended)

**Actions:**
- Add rate limiting
- Implement CSRF protection
- Add security headers
- Regular security audits
- Dependency vulnerability scanning

---

## Appendix A: Reference Links

### Official Documentation

- **Next.js:** https://nextjs.org/docs
- **Prisma:** https://www.prisma.io/docs
- **Docker:** https://docs.docker.com
- **SQLite:** https://www.sqlite.org/docs
- **TypeScript:** https://www.typescriptlang.org/docs
- **TailwindCSS:** https://tailwindcss.com/docs
- **shadcn/ui:** https://ui.shadcn.com/docs
- **Zod:** https://zod.dev
- **Jest:** https://jestjs.io/docs

### Vercel-Specific Documentation

- **Vercel Functions:** https://vercel.com/docs/functions
- **Vercel Serverless File System:** https://vercel.com/docs/functions/serverless-functions
- **Vercel Storage:** https://vercel.com/docs/storage
- **Vercel Marketplace:** https://vercel.com/marketplace
- **Neon + Vercel Integration:** https://neon.tech/docs/guides/vercel
- **Vercel Postgres (Discontinued):** https://vercel.com/docs/storage/vercel-postgres

### Deployment References

- **Next.js Deployment:** https://nextjs.org/docs/deployment
- **Next.js 15 Upgrade:** https://nextjs.org/docs/app/building-your-application/upgrading/version-15
- **Prisma Deployment:** https://www.prisma.io/docs/guides/deployment
- **Prisma Serverless Best Practices:** https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections
- **Prisma Supported Databases:** https://www.prisma.io/docs/orm/overview/databases
- **Docker Best Practices:** https://docs.docker.com/develop/develop-images/dockerfile_best-practices
- **nginx Documentation:** https://nginx.org/en/docs

---

## Appendix B: Vercel Deployment Analysis

### Can This Project Be Deployed on Vercel?

**Short Answer:** ❌ **Not in its current state.** The project requires significant modifications to be compatible with Vercel's serverless infrastructure.

---

### Critical Limitations

#### 1. **SQLite Database — BLOCKING ISSUE** 🔴

**Problem:** Vercel's serverless functions run on an **ephemeral, read-only file system**. SQLite stores data in a file (`prod.db`), which is fundamentally incompatible with this architecture.

**Technical Details:**

| Aspect | Current Project | Vercel Requirement |
|--------|-----------------|-------------------|
| **Database Type** | SQLite (file-based) | PostgreSQL, MySQL, or external DB |
| **File System** | Read/write expected | Read-only except `/tmp` |
| **Data Persistence** | Local file | External database service |
| **Connection Type** | Direct file access | Network connection required |

**Vercel's File System:**
- `/tmp` directory is writable (500MB limit)
- All other paths are read-only
- Files in `/tmp` are deleted after function execution
- **Not suitable for persistent data storage**

**Official Vercel Documentation:**
> "Serverless Functions have a read-only filesystem. You can only write to the `/tmp` directory, which is ephemeral."
> — https://vercel.com/docs/functions/serverless-functions

**Why SQLite Cannot Work on Vercel:**
1. The database file (`prod.db`) cannot be written to on a read-only file system
2. SQLite's write-ahead log (`prod.db-journal`) cannot be created
3. The `/tmp` directory is ephemeral - data is deleted after function execution
4. Multiple serverless instances cannot share a single SQLite file

---

#### 2. **Vercel Postgres Discontinued — IMPORTANT UPDATE** 🔴

**Update (December 2024):** Vercel Postgres has been **discontinued**. All existing Vercel Postgres databases were automatically migrated to **Neon**.

**Verified:** March 9, 2026

**For New Projects:**
- Use the **Vercel Marketplace** to connect external PostgreSQL databases
- Recommended: **Neon Serverless Postgres** (direct integration via Marketplace)
- Alternative: Supabase, PlanetScale, or other Marketplace providers

**Neon Integration Options:**

| Option | Billing | Preview Branching | Best For |
|--------|---------|-------------------|----------|
| **Vercel-Managed Neon** | Through Vercel | ✅ Automatic | New users, unified billing |
| **Neon-Managed** | Through Neon | ✅ Git-based | Existing Neon users |
| **Manual Connection** | Through Neon | ❌ Manual | Custom CI/CD control |

**Source:** https://vercel.com/docs/storage/vercel-postgres | https://neon.tech/docs/guides/vercel

---

#### 3. **Prisma + SQLite Incompatibility** 🔴

**Problem:** Prisma with SQLite requires file write access for:
- Database file (`prod.db`)
- Write-ahead log (`prod.db-journal`)
- Prisma query engine binaries

**Vercel + Prisma Requirements:**
- Must use PostgreSQL, MySQL, or serverless databases
- Database must be accessible via network connection string
- Prisma recommends Neon, PlanetScale, or AWS Aurora Serverless for serverless

**Required Schema Change:**
```prisma
// Current (incompatible with Vercel)
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

// Required for Vercel
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

**Note on SQLite Enums:**
The current schema uses String fields with `@default` values for enums because SQLite doesn't support native enums. PostgreSQL supports native enums, so you may want to update the schema to use PostgreSQL enum types for better type safety.

---

#### 4. **basePath Configuration** ⚠️

**Current Configuration:**
```javascript
// next.config.js
basePath: process.env.NEXT_PUBLIC_BASE_PATH || '/prompt-database',
```

**Vercel Deployment Issue:**
- Vercel deploys to root domain by default (`https://your-project.vercel.app`)
- Current config assumes `/prompt-database` subfolder
- Changing `basePath` requires **rebuilding the application**

**Required Change:**
```javascript
// For Vercel root deployment
basePath: '',
assetPrefix: '',
```

Or use Vercel's `trailingSlash` configuration if needed.

---

#### 5. **Server-Side API Routes** ⚠️

**Current Implementation:**
- API routes use Next.js App Router (`app/api/*/route.ts`)
- Direct Prisma client calls in route handlers
- No connection pooling configured

**Vercel Serverless Considerations:**
- Cold starts affect database connections
- Connection limits on serverless databases
- Need to implement connection caching/reuse

**Required Optimization:**
```typescript
// lib/prisma.ts - Needs serverless optimization
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    datasources: {
      db: {
        url: process.env.DATABASE_URL,
      },
    },
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

---

### Required Modifications for Vercel Deployment

#### Phase 1: Database Migration (Critical)

**Step 1: Choose a Vercel-Compatible Database**

**⚠️ Note:** Vercel Postgres was discontinued in December 2024. Use Neon or other Marketplace providers.

| Option | Service | Cost | Best For |
|--------|---------|------|----------|
| **Neon** | neon.tech | Free tier (0.5 GB) | Serverless PostgreSQL, Vercel integration |
| **Supabase** | supabase.com | Free tier (0.5 GB) | PostgreSQL + real-time features |
| **PlanetScale** | planetscale.com | Free tier (5 GB) | MySQL-compatible, serverless branching |
| **AWS Aurora Serverless** | aws.amazon.com | Pay-per-use | Enterprise, AWS integration |

**Recommended:** Neon Serverless Postgres via Vercel Marketplace (simplest integration, automatic credential injection)

**Step 2: Update Prisma Schema**
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
  // For Vercel serverless, use only "native" binary target
  binaryTargets = ["native"]
}
```

**Note:** The current schema uses `binaryTargets` for multi-architecture Docker support. For Vercel serverless, you only need `["native"]` since Vercel handles the build environment.

**Step 3: Update Environment Variables**
```bash
# .env for Vercel
DATABASE_URL="postgresql://user:password@host:5432/dbname?schema=public"
NODE_ENV=production
NEXT_PUBLIC_BASE_PATH=""
```

**Step 4: Run Migrations**
```bash
npx prisma migrate deploy
npx prisma generate
```

---

#### Phase 2: Configuration Updates

**Step 1: Update `next.config.js`**

**Current Configuration (for Docker/subfolder deployment):**
```javascript
const nextConfig = {
  reactStrictMode: true,
  output: 'standalone',  // For Docker minimal image
  basePath: process.env.NEXT_PUBLIC_BASE_PATH || '/prompt-database',
  assetPrefix: process.env.NEXT_PUBLIC_BASE_PATH || '/prompt-database',
}
```

**For Vercel (root domain deployment):**
```javascript
const nextConfig = {
  reactStrictMode: true,
  // Remove 'output: standalone' - Vercel handles this automatically
  basePath: '',  // Empty for root deployment
  assetPrefix: '',
}
```

**⚠️ Important:** The `basePath` is baked into the build at compile time. Changing it requires a rebuild.

**Next.js 15 Breaking Changes Awareness:**

If you plan to upgrade to Next.js 15, be aware of these breaking changes:
- `cookies()`, `headers()`, `draftMode()` are now **async**
- `params` and `searchParams` are now **async** in page components
- `fetch()` requests are **not cached by default** (must use `cache: 'force-cache'`)
- Route Handler GET methods are **not cached by default**

Source: https://nextjs.org/docs/app/building-your-application/upgrading/version-15

**Step 2: Create `vercel.json` (Optional)**
```json
{
  "framework": "nextjs",
  "regions": ["iad1"],
  "env": {
    "NEXT_PUBLIC_BASE_PATH": ""
  }
}
```

**Step 3: Update `.gitignore`**
```gitignore
# Add Vercel-specific ignores
.vercel
```

---

#### Phase 3: Vercel Project Setup

**Step 1: Install Vercel CLI**
```bash
npm install -g vercel
```

**Step 2: Login and Link**
```bash
vercel login
vercel link
```

**Step 3: Configure Environment Variables in Vercel Dashboard**
```
DATABASE_URL = postgresql://...
NODE_ENV = production
NEXT_PUBLIC_BASE_PATH = (empty or remove)
```

**Step 4: Deploy**
```bash
vercel --prod
```

---

### Vercel-Specific Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| **Serverless timeout** | 10s (Hobby), 60s (Pro) | Optimize queries, add timeouts |
| **Cold starts** | 1-3s initial load | Use Neon, connection pooling |
| **Function memory** | 1024MB - 3008MB | Monitor memory usage |
| **Build timeout** | 120s | Optimize build process |
| **API rate limits** | Varies by plan | Implement rate limiting |
| **No WebSocket** | Cannot use real-time features | Use external services (Pusher, etc.) |

---

### Alternative: Vercel + External Database Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Vercel Platform                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Next.js Application (Serverless Functions)       │  │
│  │  - Frontend (React/Next.js)                       │  │
│  │  - API Routes (App Router)                        │  │
│  └─────────────────────┬─────────────────────────────┘  │
└────────────────────────┼────────────────────────────────┘
                         │ HTTPS (Connection String)
                         ▼
┌─────────────────────────────────────────────────────────┐
│              External Database Service                   │
│  Options: Neon, Supabase, PlanetScale, AWS Aurora       │
│  - PostgreSQL or MySQL                                   │
│  - Connection pooling included                           │
│  - Automatic backups                                     │
└─────────────────────────────────────────────────────────┘
```

---

### Migration Effort Estimate

| Task | Complexity | Time Estimate |
|------|------------|---------------|
| Database schema migration | Medium | 2-4 hours |
| Data migration (if existing data) | Medium | 1-2 hours |
| Configuration updates | Low | 30 minutes |
| Testing and validation | Medium | 2-4 hours |
| Deployment and verification | Low | 1 hour |
| **Total** | | **6-12 hours** |

---

### Recommendation

**For Vercel Deployment:**

1. ✅ **Migrate to PostgreSQL** (Neon recommended via Vercel Marketplace)
2. ✅ **Update `next.config.js`** for root deployment
3. ✅ **Create `vercel.json`** configuration
4. ✅ **Set up environment variables** in Vercel dashboard
5. ✅ **Test thoroughly** in preview deployment before production

**⚠️ Important:** Vercel Postgres was discontinued in December 2024. Use Neon or other Marketplace providers.

**For Simpler Deployment (No Changes Required):**

- Continue using current Docker-based deployment
- Deploy to VPS, Railway, Render, Fly.io, or similar
- These platforms support SQLite and file-based storage

---

## Appendix C: Document Revision Summary

### What Was Updated (March 9, 2026)

This document was reviewed and updated against official documentation. The following changes were made:

#### ✅ Validated Information (Still Current)

| Topic | Status | Notes |
|-------|--------|-------|
| SQLite limitations | ✅ Valid | SQLite still not suitable for high-concurrency |
| Docker deployment | ✅ Valid | Multi-stage build still best practice |
| Prisma schema structure | ✅ Valid | Schema design is sound |
| Next.js App Router pattern | ✅ Valid | Current best practice |
| Deployment scripts analysis | ✅ Valid | Script functionality correctly described |
| Environment variable requirements | ✅ Valid | Required variables correctly identified |

#### ⚠️ Updated Information

| Topic | Change | Reason |
|-------|--------|--------|
| Vercel Postgres | **Discontinued** | Migrated to Neon in December 2024 |
| Vercel deployment compatibility | **Requires migration** | SQLite → PostgreSQL required |
| Next.js version awareness | **Added v15 notes** | Next.js 15 has breaking changes |
| Database recommendations | **Updated** | Neon now recommended over Vercel Postgres |
| File system limitations | **Clarified** | Added explicit Vercel read-only FS documentation |

#### ❌ Obsolete Information Removed/Corrected

| Original Claim | Correction |
|----------------|------------|
| "Vercel Postgres" as deployment option | Removed; replaced with Neon/Marketplace providers |
| "Prisma requires dataProxy or accelerate for serverless" | Removed; not required with proper connection pooling |
| Vercel file system URL | Corrected to current documentation URL |

---

## Appendix D: Executive Summary - Vercel Deployment Guide

### Overview

This project **cannot be deployed to Vercel in its current state** due to SQLite database incompatibility. The following phases outline the required steps for Vercel deployment.

---

### Phase 1: Preparation

**Goal:** Set up required accounts and resources

| Task | User Action | Dependencies | Resources Needed |
|------|-------------|--------------|------------------|
| 1.1 Create Vercel account | Sign up at vercel.com | None | Email address |
| 1.2 Create Neon account | Sign up at neon.tech | None | Email address |
| 1.3 Install Vercel CLI | `npm install -g vercel` | Node.js 20+ | NPM access |
| 1.4 Backup existing data | Copy `prod.db` if data exists | None | Backup storage |

**Deliverables:**
- Vercel account created
- Neon account created
- Vercel CLI installed
- Database backup (if applicable)

---

### Phase 2: Database Migration

**Goal:** Migrate from SQLite to PostgreSQL

| Task | User Action | Dependencies | Resources Needed |
|------|-------------|--------------|------------------|
| 2.1 Create Neon database | Use Neon dashboard or Vercel Marketplace | Phase 1 | Neon account |
| 2.2 Update Prisma schema | Change `provider = "sqlite"` to `"postgresql"` | None | Text editor |
| 2.3 Update binaryTargets | Change to `["native"]` only | None | Text editor |
| 2.4 Generate Prisma Client | `npx prisma generate` | Task 2.2 | Node.js |
| 2.5 Run migrations | `npx prisma migrate deploy` | Task 2.1, 2.4 | Database connection |
| 2.6 Seed database (optional) | `npx prisma db seed` | Task 2.5 | Seed script |

**Deliverables:**
- PostgreSQL database running on Neon
- Updated Prisma schema
- Migrated database structure

**⚠️ Critical Notes:**
- Vercel Postgres is discontinued; use Neon via Marketplace
- SQLite enums (String with @default) should be converted to PostgreSQL native enums (optional)
- Test migration with sample data before production

---

### Phase 3: Configuration Updates

**Goal:** Update project configuration for Vercel

| Task | User Action | Dependencies | Resources Needed |
|------|-------------|--------------|------------------|
| 3.1 Update next.config.js | Remove `output: 'standalone'`, set `basePath: ''` | None | Text editor |
| 3.2 Create vercel.json | Add optional configuration | None | Text editor |
| 3.3 Update .gitignore | Add `.vercel` | None | Text editor |
| 3.4 Update .env.example | Add PostgreSQL example | None | Text editor |
| 3.5 Test locally | `npm run dev` with PostgreSQL | Phase 2 | Local PostgreSQL or Neon |

**Configuration Changes:**

```javascript
// next.config.js - FOR VERCEL
const nextConfig = {
  reactStrictMode: true,
  basePath: '',
  assetPrefix: '',
}
```

```bash
# .env.example - FOR VERCEL
DATABASE_URL="postgresql://user:password@host:5432/dbname?schema=public"
NODE_ENV=production
NEXT_PUBLIC_BASE_PATH=""
```

**Deliverables:**
- Updated configuration files
- Working local development with PostgreSQL

---

### Phase 4: Vercel Deployment

**Goal:** Deploy to Vercel platform

| Task | User Action | Dependencies | Resources Needed |
|------|-------------|--------------|------------------|
| 4.1 Link project | `vercel link` | Phase 3 | Vercel account |
| 4.2 Add environment variables | Configure in Vercel dashboard | Phase 2 | Neon connection string |
| 4.3 Deploy to preview | `vercel` | Task 4.1, 4.2 | Git repository |
| 4.4 Test preview deployment | Verify all features | Task 4.3 | Browser |
| 4.5 Deploy to production | `vercel --prod` | Task 4.4 | Successful preview |
| 4.6 Configure custom domain (optional) | Add in Vercel dashboard | None | Domain name |

**Environment Variables for Vercel:**

| Variable | Value | Notes |
|----------|-------|-------|
| `DATABASE_URL` | Neon connection string | From Neon dashboard |
| `NODE_ENV` | `production` | Runtime mode |
| `NEXT_PUBLIC_BASE_PATH` | (empty) | For root deployment |

**Deliverables:**
- Application deployed on Vercel
- Production URL: `https://your-project.vercel.app`

---

### Phase 5: Verification

**Goal:** Ensure deployment is working correctly

| Task | User Action | Dependencies | Success Criteria |
|------|-------------|--------------|------------------|
| 5.1 Test CRUD operations | Create, read, update, delete prompts | Phase 4 | All operations succeed |
| 5.2 Test categories/tags | Verify relationships | Phase 4 | No database errors |
| 5.3 Test export/import | JSON backup functionality | Phase 4 | Files download/upload |
| 5.4 Check Vercel logs | Monitor for errors | Phase 4 | No critical errors |
| 5.5 Set up monitoring | Vercel Analytics (optional) | None | Metrics visible |

**Deliverables:**
- Verified working deployment
- Monitoring configured (optional)

---

### Migration Timeline

| Phase | Estimated Time | Complexity |
|-------|----------------|------------|
| Phase 1: Preparation | 30 minutes | Low |
| Phase 2: Database Migration | 2-4 hours | Medium |
| Phase 3: Configuration | 1 hour | Low |
| Phase 4: Deployment | 1 hour | Low |
| Phase 5: Verification | 1-2 hours | Medium |
| **Total** | **5-9 hours** | |

---

### Alternative: Simpler Deployment Options

If Vercel deployment seems too complex, consider these alternatives that support SQLite:

| Platform | SQLite Support | Setup Complexity | Best For |
|----------|----------------|------------------|----------|
| **Railway** | ✅ Yes | Low | Quick deployment |
| **Render** | ✅ Yes | Low | Simple web apps |
| **Fly.io** | ✅ Yes | Medium | Global distribution |
| **VPS (Ubuntu)** | ✅ Yes | Medium | Full control |
| **Current Docker** | ✅ Yes | Low | Already configured |

These platforms allow file-based databases and require **no schema changes**.

---

**Document End**
