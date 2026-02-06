# StreamHub MVP Implementation Plan

A comprehensive UGC video/music streaming platform with AI-powered recommendations, HLS streaming, creator monetization, and robust analytics.

---

## User Review Required

> [!IMPORTANT]
> **Monorepo vs Polyrepo**: This plan uses a monorepo structure with npm workspaces. Confirm this approach works for your deployment strategy.

> [!IMPORTANT]
> **Database Provider**: Plan assumes self-hosted PostgreSQL or a managed service (Supabase, Neon, Railway). Please confirm your preferred provider.

> [!WARNING]
> **API Keys Required**: You'll need to provision these before deployment:
> - Cloudflare R2 credentials
> - Stripe API keys (test mode for development)
> - Redis connection string
> - (Optional) OpenAI/Anthropic keys for AI features

---

## Proposed Changes

### Project Structure

```
streamhub/
├── apps/
│   ├── web/                    # Next.js 14+ frontend
│   └── api/                    # Node.js/Express backend
├── packages/
│   ├── ui/                     # Shared React component library
│   ├── config/                 # Shared ESLint, TypeScript configs
│   └── types/                  # Shared TypeScript interfaces
├── services/
│   └── transcoder/             # FFmpeg processing service
├── docker/
│   ├── docker-compose.yml
│   └── Dockerfile.*
└── package.json                # Workspace root
```

---

### Frontend (`apps/web`)

#### [NEW] [next.config.js](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/next.config.js)
Next.js 14 configuration with App Router, image optimization, and environment variables.

#### [NEW] [tailwind.config.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/tailwind.config.ts)
Custom design tokens matching the Component Spec: colors, typography, spacing, animations.

#### [NEW] [src/app/layout.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/layout.tsx)
Root layout with Navbar, Sidebar, and global providers (Auth, Theme, Query).

#### [NEW] [src/app/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/page.tsx)
Home feed with `VideoGrid` component, trending/recommended sections.

#### [NEW] [src/app/(auth)/login/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/(auth)/login/page.tsx)
Login page with email/password and OAuth buttons.

#### [NEW] [src/app/(auth)/register/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/(auth)/register/page.tsx)
Registration with form validation (React Hook Form + Zod).

#### [NEW] [src/app/watch/[id]/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/watch/[id]/page.tsx)
Video watch page with HLS player, comments, related videos.

#### [NEW] [src/app/upload/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/upload/page.tsx)
Multi-step upload wizard with drag-drop, progress tracking, metadata form.

#### [NEW] [src/app/channel/[username]/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/channel/[username]/page.tsx)
Creator channel page with banner, videos grid, about section.

#### [NEW] [src/app/studio/page.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/web/src/app/studio/page.tsx)
Creator dashboard with analytics, video management, settings.

---

### UI Component Library (`packages/ui`)

#### [NEW] [src/components/Button.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/Button.tsx)
Primary, secondary, ghost, danger variants with loading states.

#### [NEW] [src/components/Input.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/Input.tsx)
Text, password, search inputs with validation states.

#### [NEW] [src/components/Modal.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/Modal.tsx)
Accessible modal with focus trap and animations.

#### [NEW] [src/components/VideoCard.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/VideoCard.tsx)
Thumbnail with duration, title, channel info, view count.

#### [NEW] [src/components/VideoPlayer.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/VideoPlayer.tsx)
HLS.js player with custom dynamic adaptation.

#### [NEW] [src/components/VideoGrid.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/VideoGrid.tsx)
Responsive grid layout with skeleton loading states.

#### [NEW] [src/components/CommentSection.tsx](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/ui/src/components/CommentSection.tsx)
Threaded comments with reply, like, and moderation actions.

---

### Backend API (`apps/api`)

#### [NEW] [src/index.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/index.ts)
Express server entry point with middleware setup.

#### [NEW] [src/config/database.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/config/database.ts)
PostgreSQL connection with connection pooling (pg-pool).

#### [NEW] [src/config/redis.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/config/redis.ts)
Redis client for caching and session storage.

#### [NEW] [src/middleware/auth.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/middleware/auth.ts)
JWT verification middleware with Passport.js strategies.

#### [NEW] [src/routes/auth.routes.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/routes/auth.routes.ts)
`POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`

#### [NEW] [src/routes/users.routes.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/routes/users.routes.ts)
`GET /users/:id`, `PATCH /users/:id`, `GET /users/:id/videos`

#### [NEW] [src/routes/videos.routes.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/routes/videos.routes.ts)
`POST /videos`, `GET /videos/:id`, `PATCH /videos/:id`, `DELETE /videos/:id`

#### [NEW] [src/routes/comments.routes.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/routes/comments.routes.ts)
`POST /videos/:id/comments`, `GET /videos/:id/comments`, `DELETE /comments/:id`

#### [NEW] [src/routes/search.routes.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/routes/search.routes.ts)
`GET /search` with filters (type, duration, date, sort)

#### [NEW] [src/services/upload.service.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/services/upload.service.ts)
Multer + Cloudflare R2 integration for chunked uploads.

#### [NEW] [src/services/transcoding.service.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/src/services/transcoding.service.ts)
BullMQ job queue for FFmpeg transcoding to HLS.

---

### Database Migrations

#### [NEW] [migrations/001_initial_schema.sql](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/migrations/001_initial_schema.sql)
Core tables: `users`, `videos`, `comments`, `likes`, `subscriptions`, `playlists`, `watch_history`

#### [NEW] [migrations/002_indexes.sql](file:///c:/Users/Ashir/Documents/GitHub/Streamix/apps/api/migrations/002_indexes.sql)
Performance indexes on foreign keys and frequently queried columns.

---

### Shared Types (`packages/types`)

#### [NEW] [src/user.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/types/src/user.ts)
```typescript
interface User { id: string; username: string; email: string; ... }
```

#### [NEW] [src/video.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/types/src/video.ts)
```typescript
interface Video { id: string; title: string; hlsUrl: string; ... }
```

#### [NEW] [src/api.ts](file:///c:/Users/Ashir/Documents/GitHub/Streamix/packages/types/src/api.ts)
API response wrappers, pagination types, error schemas.

---

## Verification Plan

### Automated Tests

| Test Type | Command | Description |
|-----------|---------|-------------|
| **Unit Tests (Frontend)** | `npm run test -w apps/web` | Jest + React Testing Library for components |
| **Unit Tests (Backend)** | `npm run test -w apps/api` | Jest for services, utilities |
| **Integration Tests** | `npm run test:integration -w apps/api` | Supertest for API endpoints |
| **E2E Tests** | `npm run test:e2e` | Playwright for critical user flows |
| **Type Check** | `npm run typecheck` | TypeScript strict mode across workspaces |
| **Lint** | `npm run lint` | ESLint with StreamHub code style rules |

### Manual Verification

1. **User Registration Flow**
   - Navigate to `/register`
   - Fill form with valid data → Success redirect to home
   - Fill form with existing email → Error message displayed

2. **Video Upload Flow**
   - Login as any user → Navigate to `/upload`
   - Drag-drop a video file (< 100MB for testing)
   - Fill metadata (title, description, tags)
   - Submit → Progress bar completes → Video appears on channel

3. **Video Playback**
   - Navigate to any video's watch page
   - Verify HLS streaming works (multiple quality levels)
   - Test play/pause, seek, fullscreen, volume controls

4. **Search & Discovery**
   - Use search bar with a known video title → Results appear
   - Filter by duration/date → Results update accordingly

### Browser Testing (via Playwright)

```bash
# Run full E2E suite
npx playwright test

# Run specific test file
npx playwright test tests/e2e/upload.spec.ts

# Run with UI mode for debugging
npx playwright test --ui
```

---

## Tech Stack Summary

| Layer | Technology |
|-------|------------|
| **Frontend** | Next.js 14, React 18, TypeScript, Tailwind CSS, Zustand, TanStack Query |
| **Backend** | Node.js, Express.js, TypeScript, Passport.js, JWT |
| **Database** | PostgreSQL 16+, Redis |
| **Storage** | Cloudflare R2, Cloudflare CDN |
| **Processing** | FFmpeg, BullMQ |
| **Testing** | Jest, React Testing Library, Playwright, Supertest |
| **DevOps** | Docker, GitHub Actions, Vercel (frontend), Railway (backend) |

---

## Implementation Order

1. **Week 1**: Project setup, monorepo config, design system
2. **Week 2**: Auth system, user management, database schema
3. **Week 3**: Video upload, transcoding pipeline, HLS streaming
4. **Week 4**: Watch page, comments, likes, subscriptions
5. **Week 5**: Search, discovery, home feed algorithms
6. **Week 6**: Creator dashboard, analytics, polish & testing
