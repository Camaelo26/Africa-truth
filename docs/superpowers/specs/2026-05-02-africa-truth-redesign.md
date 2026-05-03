# Africa Truth — Redesign Spec
**Date:** 2026-05-02
**Author:** Macarthur Diby

---

## 1. Overview

Africa Truth is an educational platform covering the political history of all 54 African nations through curated YouTube videos and an interactive geography quiz. The current version is a static HTML/JS site hosted on AWS S3.

**Goal:** Rebuild as a modern Next.js web app backed by Supabase, wrap it as an iOS app via Capacitor, and add a simple admin panel so the creator can add YouTube videos to any country in 10 seconds without touching code.

---

## 2. Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Frontend | Next.js 14 (App Router) + Tailwind CSS | SSR for SEO, fast page loads, great DX |
| Database | Supabase (PostgreSQL) | Free tier, built-in auth, RLS, great dashboard |
| Hosting | Vercel | One-click deploys, edge network, free tier |
| iOS | Capacitor wrapping the Next.js app | Same codebase for web + iOS, no extra effort |
| UI Style | Pan-African Vibrant (red/gold/green) | Approved in design session |
| Auth | Supabase Auth (admin only) | No visitor accounts needed in v1 |
| Rate limiting | Next.js middleware (upstash/redis or in-memory) | Protect API routes from abuse |

---

## 3. Database Schema

```sql
-- Countries (seeded once, rarely updated)
CREATE TABLE countries (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name        text NOT NULL,
  slug        text UNIQUE NOT NULL,   -- e.g. "nigeria", "algeria"
  flag_emoji  text,                   -- e.g. "🇳🇬"
  region      text,                   -- "West Africa", "North Africa", etc.
  background_image_url text,          -- S3 or Supabase Storage URL
  about_text  text,
  wikipedia_url text,
  is_active   boolean DEFAULT true,
  created_at  timestamptz DEFAULT now()
);

-- Videos (the main content, managed via admin panel)
CREATE TABLE videos (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  country_id  uuid REFERENCES countries(id) ON DELETE CASCADE,
  youtube_id  text NOT NULL,          -- just the video ID, not full URL
  title       text NOT NULL,
  created_at  timestamptz DEFAULT now()
);

-- Row Level Security
-- Public: SELECT on both tables
-- Authenticated (admin): ALL on both tables
ALTER TABLE countries ENABLE ROW LEVEL SECURITY;
ALTER TABLE videos ENABLE ROW LEVEL SECURITY;

CREATE POLICY "public read countries" ON countries FOR SELECT USING (true);
CREATE POLICY "public read videos"    ON videos    FOR SELECT USING (true);
CREATE POLICY "admin all countries"   ON countries FOR ALL USING (auth.role() = 'authenticated');
CREATE POLICY "admin all videos"      ON videos    FOR ALL USING (auth.role() = 'authenticated');
```

---

## 4. Routes & Pages

| Route | Description | Auth |
|---|---|---|
| `/` | Home — interactive Africa map + search | Public |
| `/countries/[slug]` | Country page — videos grid, about, stats | Public |
| `/quiz` | Geography quiz with difficulty levels + timer | Public |
| `/about` | About the project | Public |
| `/contact` | Contact page | Public |
| `/admin` | Add/delete videos, view stats | Admin only |
| `/admin/login` | Admin login page | Public (redirects if already authed) |

---

## 5. Key Components

### Map (`/`)
- SimpleMapps worldmap (existing, kept as-is)
- Search bar with country autocomplete
- Click country → navigate to `/countries/[slug]`
- Background video (`videoplayback.webm`)
- Pan-African color scheme (red/gold/green accents on dark)

### Country Page (`/countries/[slug]`)
- Hero section: flag emoji + country name + region + population/capital (RestCountries API)
- "About" expandable panel with Wikipedia link
- Video grid: 2 columns, YouTube thumbnail (`img.youtube.com/vi/[id]/mqdefault.jpg`), title
- Videos fetched from Supabase on each page load (ISR, 60s revalidation)
- "Load more" pagination (8 videos per page)

### Admin Panel (`/admin`)
- Protected by Supabase Auth middleware
- Stats bar: total videos, total countries
- Quick-add form:
  1. Paste YouTube URL → auto-extract video ID → fetch title via YouTube oEmbed API (no key needed)
  2. Select country from dropdown
  3. Title is auto-filled but editable
  4. Hit "Add Video" → inserts to Supabase
- Recent additions list with delete button

### Quiz (`/quiz`)
- Difficulty selector: Beginner (30s), Intermediate (20s), Advanced (10s)
- Question count: 5 or 10
- Countdown timer
- Island nations excluded

---

## 6. Security Implementation

### Rate Limiting
```
middleware.ts:
- /api/* routes: 60 req/min per IP
- /admin/*: 10 req/min per IP (extra strict)
- Implementation: Upstash Redis (free tier) or in-memory Map with TTL
```

### Input Validation
- YouTube URL validator: must match `youtube.com/watch?v=` or `youtu.be/` pattern
- Video ID extracted server-side, never raw URL stored
- All Supabase queries use parameterized calls (no SQL injection possible)

### Headers (via `next.config.js`)
```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'unsafe-inline' googletagmanager.com;
  frame-src youtube.com www.youtube.com;
  img-src 'self' img.youtube.com i.ytimg.com data:;
  connect-src 'self' *.supabase.co restcountries.com;

X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
```

### Environment Variables
```
NEXT_PUBLIC_SUPABASE_URL=       (public, read-only access)
NEXT_PUBLIC_SUPABASE_ANON_KEY=  (public, RLS enforces read-only)
SUPABASE_SERVICE_ROLE_KEY=      (server-only, never in browser)
UPSTASH_REDIS_REST_URL=         (server-only)
UPSTASH_REDIS_REST_TOKEN=       (server-only)
```

### Supabase Auth
- Single admin account (email + password)
- Session stored in httpOnly cookie
- Middleware redirects unauthenticated requests to `/admin/login`
- Supabase blocks repeated failed logins automatically

### iOS (Capacitor)
- App Transport Security (ATS) enforced — all requests HTTPS
- No sensitive data stored locally on device
- Supabase anon key is read-only (safe in bundle)

---

## 7. iOS Deployment (Capacitor)

```bash
# After Next.js build
npm run build
npx cap sync ios
npx cap open ios   # opens Xcode
# → Archive → Submit to App Store
```

- Capacitor config: `capacitor.config.ts` points to production Vercel URL
- `@capacitor/status-bar`, `@capacitor/splash-screen` for native feel
- App icon: Pan-African themed (red/gold/green)

---

## 8. Migration Plan

### Data Migration
1. Parse all existing 48 country HTML pages → extract YouTube video IDs + titles
2. Seed `countries` table (54 countries, slugs, flags, regions)
3. Seed `videos` table from parsed data
4. Upload country background images to Supabase Storage

### Phased Rollout
| Phase | Work | Status |
|---|---|---|
| 0 | Fill 5 empty country pages (current HTML) | In progress |
| 1 | Set up Next.js project + Supabase + Vercel | Next |
| 2 | Build map page + country pages | After Phase 1 |
| 3 | Build admin panel | After Phase 2 |
| 4 | Security hardening (rate limiting, CSP headers) | After Phase 3 |
| 5 | Capacitor iOS build | After Phase 4 |
| 6 | App Store submission | After Phase 5 |

---

## 9. Out of Scope (v1)

- User accounts / bookmarks (can add in v2)
- Comments or ratings
- Push notifications
- Android (comes for free with Capacitor but not the priority)
- Video uploading (YouTube embeds only)
