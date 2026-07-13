# Technical Requirements Document (TRD)
**Project:** Curated Multi-Author Blog Platform[cite: 1]  
**Version:** 2.0 (Production-Hardened)  
**Document Type:** System & Engineering Specification[cite: 1]  
**Stack:** Next.js 15, React 19, TypeScript, Clerk, Prisma, Neon PostgreSQL, Cloudinary, Tailwind CSS, Upstash Redis[cite: 1]  

---

## Section 1: Foundation & System Architecture

### 1.1 Purpose & Architectural Objectives
This document establishes the authoritative technical blueprint for building, deploying, and scaling a curated multi-author blog platform[cite: 1]. The architecture prioritizes server-first rendering, robust data integrity, sub-second content discovery, and strict role-based access control (RBAC)[cite: 1].

#### Key Engineering Objectives
* **Zero-Locking Scalability:** Sustain 1,000+ concurrent active users without database connection pool exhaustion or query degradation[cite: 1].
* **Sub-Second Read Performance:** Serve static and cached article reads from global edge networks with initial page loads under 2.0 seconds[cite: 1].
* **Strict Access Boundaries:** Enforce absolute isolation between unauthenticated guests, registered readers, approved authors, and system administrators at both the edge routing and database layers[cite: 1].
* **Maintainable Monolith:** Co-locate frontend presentation and backend API logic within a single Next.js App Router repository to maximize type safety and streamline deployment pipelines[cite: 1].

### 1.2 High-Level System Architecture & Request Flow
The platform utilizes a modern serverless edge architecture. Static assets and cached reads are terminated at the Content Delivery Network (CDN) edge, while dynamic mutations and authenticated workflows route through serverless compute functions to a managed PostgreSQL database[cite: 1].

```
                  ┌────────────────────┐
                  │    User Browser    │
                  └─────────┬──────────┘
                            │
                            ▼
                  ┌────────────────────┐
                  │   Cloudflare CDN   │ (SSL Termination, DDoS Protection, Static Caching)
                  └─────────┬──────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼ (Cache Miss / Dynamic)        ▼ (Cache Hit)
┌───────────────────────┐         ┌──────────────────┐
│  Vercel Edge Network  │         │  Edge Cache Read │
└───────────┬───────────┘         └──────────────────┘
            │
            ▼
┌───────────────────────┐         ┌─────────────────────────────────┐
│ Next.js 15 App Router │◄───────►│  Clerk Auth (Edge Middleware)   │
└───────────┬───────────┘         └─────────────────────────────────┘
            │
            ├───────────────────────────────────────┐
            ▼ (Data Queries / Mutations)            ▼ (Media Uploads / Delivery)
┌───────────────────────┐                 ┌──────────────────┐
│  Prisma ORM Client    │                 │ Cloudinary API   │
└───────────┬───────────┘                 └──────────────────┘
            │
            ├──────────────────────┐
            ▼                      ▼
┌───────────────────────┐ ┌──────────────────┐
│   Neon PostgreSQL     │ │  Upstash Redis   │ (Rate Limiting, Async Queues)
└───────────────────────┘ └──────────────────┘
```

### 1.3 Directory Structure & Component Architecture
To enforce separation of concerns, the repository follows a feature-scoped modular hierarchy[cite: 1]:

```
app/
├── (public)/              # Statically cached public reading routes
│   ├── blog/[slug]/       # Article detail page (ISR + On-Demand Revalidation)
│   ├── categories/[slug]/ # Category filtering
│   ├── author/[username]/ # Public author profile
│   └── page.tsx           # Homepage
├── (authenticated)/       # Routes requiring Clerk session validation
│   ├── dashboard/         # Author content management system
│   ├── admin/             # System administrator control panel
│   └── settings/          # User profile configuration
├── api/                   # Backend Route Handlers
│   ├── blogs/             # Article CRUD and publishing mutations
│   ├── comments/          # Threaded comment management
│   └── webhooks/clerk/    # Cryptographic profile synchronization
├── layout.tsx             # Root application layout
└── middleware.ts          # Edge authentication and rate limiting
components/
├── ui/                    # Reusable shadcn/ui atomic components
├── blog/                  # Blog cards, reading progress, TOC
├── editor/                # Rich text editor and preview canvas
└── layout/                # Navigation, sidebars, footers
lib/
├── auth/                  # Clerk helper wrappers and RBAC guards
├── db/                    # Singleton Prisma client instance
├── validation/            # Zod request payload schemas
└── utils/                 # Formatting, slugification, and error handlers
prisma/
├── schema.prisma          # Relational database schema
└── migrations/            # Versioned SQL database changes
```

### 1.4 Non-Functional Requirements & SLAs
Engineering execution must meet the following strict Service Level Agreements across all production environments[cite: 1]:

* **Core Web Vitals Thresholds:**
  * Largest Contentful Paint (LCP): < 2.5 seconds[cite: 1]
  * Interaction to Next Paint (INP): < 200 ms[cite: 1]
  * Cumulative Layout Shift (CLS): < 0.1[cite: 1]
  * Lighthouse Performance, Accessibility, Best Practices, and SEO: >= 95[cite: 1]
* **System Capacity:**
  * Database must natively support 100,000+ registered user profiles[cite: 1].
  * Search and listing indexes must scale beyond 10,000+ published articles without query degradation[cite: 1].
  * Route Handlers must support 1,000+ concurrent active users[cite: 1].

### 1.5 Engineering & Coding Standards
* **Strict Static Typing:** TypeScript strict mode enabled; explicit return types required on all API handlers and utility functions; zero implicit `any` allowed[cite: 1].
* **Component Paradigm:** React Server Components (RSC) used by default to minimize client bundle size[cite: 1]. Client Components (`'use client'`) restricted strictly to interactive UI elements (modals, forms, rich text editor)[cite: 1].
* **Validation Layer:** All incoming API request bodies, query parameters, and form submissions must be validated using Zod before executing business logic[cite: 1].
* **Clean Architecture:** Zero inline SQL or direct database calls within UI components[cite: 1]. All data access must route through typed Prisma service layers or Next.js Route Handlers[cite: 1].

---

## Section 2: Backend Architecture, Authentication & Database

### 2.1 Monolithic Route Handler Architecture
Backend logic is structured into RESTful Next.js Route Handlers (`app/api/*`)[cite: 1]. Each handler executes a standardized execution pipeline:
1. **Edge Middleware Check:** Validate IP rate limits and IP blocklists.
2. **Authentication & RBAC:** Verify Clerk session token and extract user role from database context[cite: 1].
3. **Payload Validation:** Parse and validate request JSON against a Zod schema[cite: 1].
4. **Business Logic & Mutation:** Execute atomic Prisma database transactions[cite: 1].
5. **Cache Invalidation:** Trigger Next.js on-demand revalidation tags where applicable.
6. **Standardized JSON Response:** Return typed success or structured error payloads[cite: 1].

### 2.2 Authentication & Authorization (RBAC)
Clerk manages credential storage, session tokens, multi-factor authentication, and OAuth flows (Google, GitHub)[cite: 1]. Local user state is mirrored in the PostgreSQL database to support relational joins with articles and comments[cite: 1].

#### Permission Matrix
| Capability | Guest[cite: 1] | User[cite: 1] | Author[cite: 1] | Admin[cite: 1] |
| :--- | :--- | :--- | :--- | :--- |
| **Read Published Articles** | Yes[cite: 1] | Yes[cite: 1] | Yes[cite: 1] | Yes[cite: 1] |
| **Like / Bookmark Articles**| No | Yes[cite: 1] | Yes[cite: 1] | Yes[cite: 1] |
| **Submit Comments** | No | Yes[cite: 1] | Yes[cite: 1] | Yes[cite: 1] |
| **Create Article Drafts** | No | No | Yes[cite: 1] | Yes[cite: 1] |
| **Publish / Edit Own Articles**| No | No | Yes[cite: 1] | Yes[cite: 1] |
| **Manage Categories & Tags**| No | No | No | Yes[cite: 1] |
| **Promote / Revoke Authors**| No | No | No | Yes[cite: 1] |
| **Moderate Global Content** | No | No | No | Yes[cite: 1] |

#### Author Approval Workflow
1. A new visitor registers via Clerk; the database synchronization webhook creates a user record with `role = USER`[cite: 1].
2. The user applies for author access; an administrator reviews the profile via `/admin/users`[cite: 1].
3. The administrator promotes the account; the database updates to `role = AUTHOR`[cite: 1].
4. Next.js Middleware recognizes the updated role claim, granting access to `/dashboard` and publishing endpoints[cite: 1].

### 2.3 Hardened Database Schema
The relational schema is structured in Prisma to enforce data integrity, cascading deletions, and high-speed B-tree indexing across critical query paths[cite: 1].

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum Role {
  USER
  AUTHOR
  ADMIN
}

enum BlogStatus {
  DRAFT
  PENDING_REVIEW
  PUBLISHED
  ARCHIVED
}

model User {
  id          String     @id @default(uuid()) @db.Uuid
  clerkId     String     @unique
  username    String     @unique
  displayName String
  email       String     @unique
  avatar      String
  bio         String?
  role        Role       @default(USER)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  blogs       Blog[]
  comments    Comment[]
  likes       Like[]
  bookmarks   Bookmark[]
  followers   Follower[] @relation("UserFollowers")
  following   Follower[] @relation("UserFollowing")

  @@index([role])
}

model Category {
  id          String   @id @default(uuid()) @db.Uuid
  name        String
  slug        String   @unique
  description String?
  blogs       Blog[]
}

model Tag {
  id    String    @id @default(uuid()) @db.Uuid
  name  String    @unique
  slug  String    @unique
  blogs BlogTag[]
}

model Blog {
  id           String     @id @default(uuid()) @db.Uuid
  title        String
  slug         String     @unique
  excerpt      String?
  content      Json       // Stored as rich text AST; never queried directly for text search
  coverImage   String
  status       BlogStatus @default(DRAFT)
  publishedAt  DateTime?
  readingTime  Int
  viewCount    Int        @default(0)
  likeCount    Int        @default(0)
  commentCount Int        @default(0)
  
  authorId     String     @db.Uuid
  author       User       @relation(fields: [authorId], references: [id], onDelete: Cascade)
  
  categoryId   String     @db.Uuid
  category     Category   @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt

  tags         BlogTag[]
  comments     Comment[]
  likes        Like[]
  bookmarks    Bookmark[]

  // Mandatory B-Tree Indexes for High-Traffic Query Paths
  @@index([authorId])
  @@index([categoryId])
  @@index([status, publishedAt(sort: Desc)])
}

model BlogTag {
  blogId  String @db.Uuid
  tagId   String @db.Uuid
  blog    Blog   @relation(fields: [blogId], references: [id], onDelete: Cascade)
  tag     Tag    @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([blogId, tagId])
  @@index([tagId])
}

model Comment {
  id        String    @id @default(uuid()) @db.Uuid
  content   String
  blogId    String    @db.Uuid
  blog      Blog      @relation(fields: [blogId], references: [id], onDelete: Cascade)
  userId    String    @db.Uuid
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  parentId  String?   @db.Uuid
  parent    Comment?  @relation("CommentReplies", fields: [parentId], references: [id], onDelete: Cascade)
  replies   Comment[] @relation("CommentReplies")
  createdAt DateTime  @default(now())

  @@index([blogId])
  @@index([parentId])
}

model Like {
  blogId  String @db.Uuid
  userId  String @db.Uuid
  blog    Blog   @relation(fields: [blogId], references: [id], onDelete: Cascade)
  user    User   @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@id([blogId, userId])
  @@index([userId])
}

model Bookmark {
  blogId    String   @db.Uuid
  userId    String   @db.Uuid
  blog      Blog     @relation(fields: [blogId], references: [id], onDelete: Cascade)
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())

  @@id([blogId, userId])
  @@index([userId])
}

model Follower {
  followerId String @db.Uuid
  authorId   String @db.Uuid
  follower   User   @relation("UserFollowing", fields: [followerId], references: [id], onDelete: Cascade)
  author     User   @relation("UserFollowers", fields: [authorId], references: [id], onDelete: Cascade)

  @@id([followerId, authorId])
  @@index([authorId])
}
```

### 2.4 High-Speed Full-Text Search (GIN Indexing)
To achieve sub-200ms search responses without runtime string formatting or expensive JSON column scanning[cite: 1], execute the following SQL migration against PostgreSQL immediately following Prisma schema deployment:

```sql
-- Create an auto-updating generated tsvector column combining weighted title and excerpt
ALTER TABLE "Blog" ADD COLUMN search_vector tsvector 
GENERATED ALWAYS AS (
  setweight(to_tsvector('english', coalesce(title, '')), 'A') || 
  setweight(to_tsvector('english', coalesce(excerpt, '')), 'B')
) STORED;

-- Apply a Generalized Inverted Index (GIN) for high-speed lexical matching
CREATE INDEX blog_search_vector_idx ON "Blog" USING GIN (search_vector);
```

### 2.5 REST API Specifications & Error Handling
All API endpoints exchange JSON payloads and adhere to strict HTTP status code semantics[cite: 1].

#### Core API Contracts
| Endpoint | Method | Authentication | Description |
| :--- | :--- | :--- | :--- |
| `/api/blogs`[cite: 1] | GET | Public | List published articles with pagination, category, and tag filters[cite: 1]. |
| `/api/blogs`[cite: 1] | POST | Author / Admin | Create a new article draft or publish immediately[cite: 1]. |
| `/api/blogs/[id]`[cite: 1] | PUT | Author / Admin | Update article content, metadata, or status[cite: 1]. |
| `/api/blogs/[id]/like`[cite: 1]| POST / DELETE | Registered User | Toggle like status for an article; atomically updates `likeCount`[cite: 1]. |
| `/api/comments`[cite: 1] | POST | Registered User | Submit a top-level comment or threaded reply[cite: 1]. |
| `/api/admin/users/[id]/role`[cite: 1]| PATCH | Admin | Promote or demote user roles (`USER`, `AUTHOR`, `ADMIN`)[cite: 1]. |

#### Standardized Error Payload Structure
When validation or business logic fails, APIs return a consistent JSON schema[cite: 1]:
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request payload failed validation checks.",
    "details": [
      {
        "field": "title",
        "message": "Title must be between 10 and 120 characters."
      }
    ]
  }
}
```

### 2.6 Data Integrity Protocols

#### Slug Collision Resolution
Because `Blog.slug` enforces a database unique constraint[cite: 1], automated slugification from article titles must handle duplicates cleanly without throwing HTTP 409 Conflict errors[cite: 1]:
* **Protocol:** When generating a slug, the backend checks database existence. If a match is found, the system appends a hyphen followed by a 6-character alphanumeric string generated via `nanoid` (e.g., `modern-architecture-a8k2x9`) before executing the Prisma write payload.

#### Comment Nesting Depth Control
To prevent recursive Common Table Expression (CTE) queries from locking database memory during complex discussions:
* **Protocol:** The `/api/comments` Zod schema enforces a maximum reply depth of 2 levels[cite: 1]. If a user attempts to reply to a level-2 comment, the backend forces the `parentId` to reference the level-1 parent comment, flattening deeper threads automatically.

---

## Section 3: Frontend Architecture, UI/UX & Feature Specifications

### 3.1 Rendering Strategy & Revalidation Architecture
The platform combines Incremental Static Regeneration (ISR) with On-Demand Revalidation to balance sub-second page delivery with instant content freshness[cite: 1].

| Page Route | Rendering Strategy | Revalidation Mechanism |
| :--- | :--- | :--- |
| `/` (Homepage)[cite: 1] | Statically Rendered at Build | On-Demand Revalidation via cache tag `blog-listing`[cite: 1]. |
| `/blog/[slug]`[cite: 1] | ISR (Incremental Static Regeneration) | Statically cached; invalidated instantly via `revalidatePath` on edit[cite: 1]. |
| `/categories/[slug]`[cite: 1]| Statically Rendered at Build | Revalidated when articles are published or assigned to categories[cite: 1]. |
| `/search`[cite: 1] | Dynamic Client-Side (TanStack Query) | Real-time querying against `/api/blogs?q=` with debounce[cite: 1]. |
| `/dashboard/*`[cite: 1] | Server-Side Rendered (Dynamic) | Strictly dynamic; requires real-time Clerk session validation[cite: 1]. |
| `/admin/*`[cite: 1] | Server-Side Rendered (Dynamic) | Strictly dynamic; requires Admin role verification[cite: 1]. |

### 3.2 Design System & UI Component Hierarchy
Built on Tailwind CSS and shadcn/ui[cite: 1], the visual hierarchy relies on strict tokenization[cite: 1]:
* **Typography:** Geist Sans for primary headings; Inter for body copy; JetBrains Mono for code blocks[cite: 1].
* **Spacing Scale:** Strict 8px baseline grid (`8px`, `16px`, `24px`, `32px`, `48px`, `64px`)[cite: 1].
* **Theme Support:** Native Light, Dark, and System preferences managed via `next-themes` with zero layout shift or hydration flash[cite: 1].

### 3.3 Key Page & Feature Specifications

#### Article Detail Page (`/blog/[slug]`)
* **Sticky Table of Contents (TOC):** Automatically generated by parsing H2 and H3 tags from the rich text AST; highlights the active section during scroll via Intersection Observer[cite: 1].
* **Reading Progress Indicator:** A fixed 2px accent bar at the top of the viewport tracking vertical scroll completion[cite: 1].
* **Optimized Code Blocks:** Code snippets render with syntax highlighting, line numbers, and a one-click "Copy Code" clipboard button[cite: 1].

#### Rich Text Editor (Author Dashboard)
* **Editor Engine:** Headless rich text editor supporting structured JSON AST output[cite: 1].
* **Capabilities:** Headings, blockquotes, code blocks, bullet/numbered lists, markdown shortcuts, embedded YouTube videos, and HTML tables[cite: 1].
* **Autosave Workflow:** Form state syncs automatically to localStorage every 5 seconds and pushes draft updates to `/api/blogs/[id]` every 30 seconds[cite: 1].

### 3.4 Client-Side State Management & XSS Safeguards
* **Server State:** Managed exclusively via TanStack Query; handles pagination, infinite scrolling, and background polling for interactive comment threads[cite: 1].
* **Form State:** Controlled via React Hook Form paired with Zod validation resolvers[cite: 1].
* **Mandatory XSS Hydration Guard:** Because article content is stored as JSON AST and may contain embedded HTML or external links[cite: 1], React's default escaping is insufficient[cite: 1]. All rich text payloads must pass through a strict `DOMPurify` sanitization pipeline before client-side DOM rendering.

### 3.5 SEO & Core Web Vitals Strategy
* **Metadata Injection:** Every public route dynamically generates Next.js Metadata objects containing canonical URLs, Open Graph tags, Twitter Cards, and dynamic `<title>` elements[cite: 1].
* **Structured Data:** Inject JSON-LD schemas (`Article`, `Person`, `Organization`, `BreadcrumbList`) into the `<head>` of blog detail pages to maximize rich snippet indexing[cite: 1].
* **Image Optimization:** All cover imagery and author avatars must route through the Next.js `<Image/>` component, leveraging Cloudinary for automatic AVIF/WEBP format conversion and responsive `srcset` generation[cite: 1].

---

## Section 4: Infrastructure, DevOps, Security & Production Readiness

### 4.1 Cloud Infrastructure & Edge Routing
The platform operates on a distributed serverless infrastructure designed for zero maintenance overhead and infinite horizontal scaling[cite: 1]:
* **Edge Layer:** Cloudflare handles DNS, SSL termination, DDoS mitigation, and edge caching for static assets[cite: 1].
* **Application Hosting:** Vercel deploys the Next.js App Router, executing Route Handlers within isolated serverless functions and serving static pages from global edge nodes[cite: 1].
* **Database Infrastructure:** Neon PostgreSQL provides a managed serverless database with automated compute scaling and connection pooling[cite: 1].
* **Media Delivery:** Cloudinary acts as the dedicated media storage and image optimization CDN[cite: 1].

### 4.2 Edge Rate Limiting & API Governance
Upstash Redis is integrated into Next.js Edge Middleware (`middleware.ts`) to enforce sliding-window rate limits, protecting the database from scraping and brute-force attacks[cite: 1].

| Endpoint Path | Protected Methods | Time Window | Maximum Requests | Violation Action |
| :--- | :--- | :--- | :--- | :--- |
| `/api/blogs/[id]/like`[cite: 1]| POST, DELETE[cite: 1] | 60 Seconds | 10 per IP | Return HTTP 429 Too Many Requests |
| `/api/comments`[cite: 1] | POST[cite: 1] | 60 Seconds | 5 per User ID | Return HTTP 429 Too Many Requests |
| `/api/blogs` (Search)[cite: 1] | GET[cite: 1] | 60 Seconds | 30 per IP | Return HTTP 429 Too Many Requests |
| `/api/bookmarks`[cite: 1] | POST, DELETE[cite: 1] | 60 Seconds | 15 per User ID| Return HTTP 429 Too Many Requests |

### 4.3 Cryptographic Webhook Synchronization
To maintain synchronization between Clerk's identity servers and the local PostgreSQL `User` table without exposing the system to privilege escalation:
* **Protocol:** The `/api/webhooks/clerk` endpoint uses the `svix` library to verify the `svix-id`, `svix-timestamp`, and `svix-signature` headers against the secret `CLERK_WEBHOOK_SECRET`.
* **Enforcement:** Any payload failing cryptographic validation is immediately rejected with HTTP 401 Unauthorized before database connections are initialized[cite: 1].

### 4.4 CI/CD Pipeline & Git Workflow
Development follows a strict Git-flow branching model (`main`, `develop`, `feature/*`, `hotfix/*`)[cite: 1]. Pull requests into `develop` or `main` trigger an automated GitHub Actions CI pipeline:
1. **Dependency Install:** Install frozen packages using `pnpm`.
2. **Linting & Prettier:** Verify formatting and ESLint rules[cite: 1].
3. **Static Type Check:** Execute `tsc --noEmit` to confirm zero TypeScript compilation errors[cite: 1].
4. **Automated Testing:** Run Vitest unit tests for Zod schemas and utility logic, followed by Playwright end-to-end tests covering authentication, publishing, and commenting workflows[cite: 1].
5. **Staging Deployment:** Merging to `develop` deploys an isolated preview build connected to a Neon database branching environment[cite: 1].
6. **Production Release:** Merging `develop` into `main` triggers a zero-downtime production release on Vercel[cite: 1].

### 4.5 Security Hardening & HTTP Headers
The application enforces defensive HTTP headers globally via `next.config.ts`[cite: 1]:
* `Content-Security-Policy`: Restrict script execution to trusted domains (Vercel, Clerk, Cloudflare)[cite: 1].
* `Strict-Transport-Security`: Enforce HTTPS connections with `max-age=31536000; includeSubDomains; preload`[cite: 1].
* `X-Frame-Options`: Set to `DENY` to prevent clickjacking attacks[cite: 1].
* `X-Content-Type-Options`: Set to `nosniff` to prevent MIME-sniffing vulnerabilities[cite: 1].
* `Referrer-Policy`: Set to `strict-origin-when-cross-origin`[cite: 1].

### 4.6 Disaster Recovery, Backups & Operational Runbooks
* **Database Resilience:** Neon PostgreSQL executes automated continuous daily backups with a 30-day retention schedule[cite: 1].
* **RPO / RTO SLAs:** In the event of a catastrophic database failure, system restoration procedures guarantee a Recovery Point Objective (RPO) of <= 24 hours and a Recovery Time Objective (RTO) of <= 4 hours via point-in-time recovery[cite: 1].
* **Incident Response Runbook:**
    1. Detect anomaly via Sentry exception alerts or Vercel Analytics latency spikes[cite: 1].
    2. Triage severity and engage engineering stakeholders[cite: 1].
    3. If a bad deployment is identified, execute an immediate one-click rollback in Vercel to the previous immutable Git commit[cite: 1].
    4. Verify database stability and edge cache integrity[cite: 1].

### 4.7 Production Readiness Checklist
Before signing off on production deployment, the engineering team must verify that every item on this operational checklist is complete[cite: 1]:
* [x] All TypeScript compilation passes in strict mode with zero implicit `any` types[cite: 1].
* [x] Zod validation schemas strictly enforce image upload limits (<= 10 MB) and restrict media types to JPG, PNG, WEBP, and AVIF[cite: 1].
* [x] Prisma database indexes (`authorId`, `categoryId`, `status/publishedAt`, and GIN vector search) are applied to the production Neon database[cite: 1].
* [x] Upstash Redis rate limiting is actively throttling requests in Next.js Middleware[cite: 1].
* [x] Clerk webhook endpoints cryptographically verify all incoming payloads using `svix`[cite: 1].
* [x] All UI text hydration paths pass through `DOMPurify` to eliminate XSS injection vulnerabilities[cite: 1].
* [x] Security HTTP headers are actively injected and validated via security audit tools[cite: 1].
* [x] Playwright E2E test suites pass across desktop and mobile viewports[cite: 1].
* [x] Lighthouse CI scores register >= 95 across Performance, Accessibility, Best Practices, and SEO on production preview URLs[cite: 1].
