# Africa Truth Web App — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild Africa Truth as a Next.js 14 + Supabase web app with Pan-African UI (red/gold/green on dark), rich per-country about sections, a mobile-first responsive map, an admin panel to add YouTube videos without touching code, rate limiting, CSP headers, and Vercel deployment.

**Architecture:** Next.js 14 App Router + Tailwind for the frontend; Supabase PostgreSQL for `countries` and `videos` tables with Row Level Security; Wikipedia REST API for live country summaries; responsive layout switches from interactive simplemaps map (desktop) to swipeable country card grid (mobile); Next.js middleware handles rate limiting (Upstash Redis) and admin route protection; Vercel auto-deploys from GitHub.

**Tech Stack:** Next.js 14, TypeScript, Tailwind CSS, Supabase (`@supabase/ssr`), Upstash Redis (`@upstash/ratelimit`), Wikipedia REST API, Vitest + React Testing Library

---

## File Map

```
(repo root — Africa-truth/)
├── app/
│   ├── layout.tsx                  root layout, NavBar, theme
│   ├── globals.css                 Tailwind base + Pan-African vars
│   ├── page.tsx                    home — interactive map + search
│   ├── countries/[slug]/page.tsx   country page — hero + video grid
│   ├── quiz/page.tsx               quiz — difficulty + timer
│   ├── about/page.tsx
│   ├── contact/page.tsx
│   ├── admin/
│   │   ├── page.tsx                dashboard — stats + add video
│   │   └── login/page.tsx          email/password login
│   └── api/
│       ├── youtube-meta/route.ts   GET ?url= → {id, title}
│       └── videos/route.ts         POST add / DELETE remove
├── components/
│   ├── NavBar.tsx
│   ├── AfricaMap.tsx               wraps simplemaps (client component)
│   ├── SearchBar.tsx               country autocomplete
│   ├── CountryHero.tsx             flag, name, region, stats
│   ├── VideoGrid.tsx               2-col grid of VideoCards
│   ├── VideoCard.tsx               thumbnail + title + play
│   ├── QuizGame.tsx                full quiz logic
│   ├── AdminVideoForm.tsx          paste URL → auto title → save
│   └── RecentVideos.tsx            list with delete button
├── lib/
│   ├── supabase/client.ts          browser Supabase client
│   ├── supabase/server.ts          server Supabase client (cookies)
│   ├── youtube.ts                  parseYouTubeId(), fetchYouTubeMeta()
│   ├── rate-limit.ts               Upstash ratelimit instances
│   └── types.ts                    Country, Video TypeScript types
├── middleware.ts                    rate limit + admin auth guard
├── next.config.js                  CSP + security headers
├── public/scripts/
│   ├── mapdata.js                  updated country URLs → /countries/[slug]
│   └── worldmap.js                 copy of existing simplemaps lib
├── scripts/migrate.ts              parse HTML files → seed Supabase
├── vitest.config.ts
├── vitest.setup.ts
└── __tests__/
    ├── lib/youtube.test.ts
    └── api/youtube-meta.test.ts
```

---

## Task 1: Initialize Next.js 14 Project

**Files:**
- Create: `package.json`, `next.config.js`, `tsconfig.json`, `tailwind.config.ts`, `app/globals.css`

- [ ] **Step 1: Scaffold the project**

Run from `C:\Users\dibyc\OneDrive\Bureau\personal project\africa-truth-v2` (new folder):

```bash
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir --import-alias "@/*"
```

When prompted: TypeScript=Yes, ESLint=Yes, Tailwind=Yes, src/=No, App Router=Yes, import alias=@/*

- [ ] **Step 2: Install dependencies**

```bash
npm install @supabase/supabase-js @supabase/ssr @upstash/redis @upstash/ratelimit
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @types/node
```

- [ ] **Step 3: Configure Vitest**

Create `vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./vitest.setup.ts'],
    globals: true,
  },
})
```

Create `vitest.setup.ts`:
```typescript
import '@testing-library/jest-dom'
```

- [ ] **Step 4: Add test script to package.json**

In `package.json`, add to `"scripts"`:
```json
"test": "vitest run",
"test:watch": "vitest"
```

- [ ] **Step 5: Verify setup**

```bash
npm run dev
```
Expected: server starts at http://localhost:3000 with default Next.js page.

```bash
npm test
```
Expected: "No test files found" (no failures).

- [ ] **Step 6: Commit**

```bash
git init && git add -A && git commit -m "feat: init Next.js 14 project with Vitest"
```

---

## Task 2: Tailwind Pan-African Theme

**Files:**
- Modify: `tailwind.config.ts`
- Modify: `app/globals.css`

- [ ] **Step 1: Update Tailwind config**

Replace `tailwind.config.ts`:
```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: ['./app/**/*.{ts,tsx}', './components/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        pan: {
          red:    '#e63946',
          gold:   '#ffd60a',
          green:  '#2dc653',
        },
        dark: {
          900: '#0a0500',
          800: '#110900',
          700: '#1a0e00',
          600: '#221500',
        },
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
      },
    },
  },
  plugins: [],
}
export default config
```

- [ ] **Step 2: Update globals.css**

Replace `app/globals.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --pan-red:   #e63946;
  --pan-gold:  #ffd60a;
  --pan-green: #2dc653;
}

body {
  background-color: #0a0500;
  color: #ffffff;
}

@layer components {
  .btn-primary {
    @apply bg-gradient-to-r from-pan-red to-pan-gold text-black font-bold px-6 py-3 rounded-lg;
  }
  .card-dark {
    @apply bg-white/5 border border-white/10 rounded-xl;
  }
}
```

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat: add Pan-African Tailwind theme"
```

---

## Task 3: Supabase Schema + Country Seed

**Files:** SQL run in Supabase dashboard

- [ ] **Step 1: Create Supabase project**

Go to https://supabase.com → New project → name: `africa-truth` → note the Project URL and anon key.

- [ ] **Step 2: Run schema SQL**

In Supabase SQL Editor, run:
```sql
CREATE TABLE countries (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name             text NOT NULL,
  slug             text UNIQUE NOT NULL,
  flag_emoji       text,
  region           text,
  background_image_url text,
  about_text       text,
  wikipedia_url    text,
  is_active        boolean DEFAULT true,
  created_at       timestamptz DEFAULT now()
);

CREATE TABLE videos (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  country_id  uuid REFERENCES countries(id) ON DELETE CASCADE,
  youtube_id  text NOT NULL,
  title       text NOT NULL,
  created_at  timestamptz DEFAULT now()
);

ALTER TABLE countries ENABLE ROW LEVEL SECURITY;
ALTER TABLE videos    ENABLE ROW LEVEL SECURITY;

CREATE POLICY "public_read_countries" ON countries FOR SELECT USING (true);
CREATE POLICY "public_read_videos"    ON videos    FOR SELECT USING (true);
CREATE POLICY "admin_all_countries"   ON countries FOR ALL USING (auth.role() = 'authenticated');
CREATE POLICY "admin_all_videos"      ON videos    FOR ALL USING (auth.role() = 'authenticated');

CREATE INDEX idx_videos_country_id ON videos(country_id);
CREATE INDEX idx_countries_slug    ON countries(slug);
```

- [ ] **Step 3: Seed all 49 active countries**

In Supabase SQL Editor, run:
```sql
INSERT INTO countries (name, slug, flag_emoji, region, wikipedia_url) VALUES
('Algeria',                  'algeria',                '🇩🇿', 'North Africa',    'https://en.wikipedia.org/wiki/Algeria'),
('Angola',                   'angola',                 '🇦🇴', 'Central Africa',  'https://en.wikipedia.org/wiki/Angola'),
('Benin',                    'benin',                  '🇧🇯', 'West Africa',     'https://en.wikipedia.org/wiki/Benin'),
('Botswana',                 'botswana',               '🇧🇼', 'Southern Africa', 'https://en.wikipedia.org/wiki/Botswana'),
('Burkina Faso',             'burkinafaso',            '🇧🇫', 'West Africa',     'https://en.wikipedia.org/wiki/Burkina_Faso'),
('Burundi',                  'burundi',                '🇧🇮', 'East Africa',     'https://en.wikipedia.org/wiki/Burundi'),
('Cameroon',                 'cameroon',               '🇨🇲', 'Central Africa',  'https://en.wikipedia.org/wiki/Cameroon'),
('Central African Republic', 'centralafricanrepublic', '🇨🇫', 'Central Africa',  'https://en.wikipedia.org/wiki/Central_African_Republic'),
('Chad',                     'chad',                   '🇹🇩', 'Central Africa',  'https://en.wikipedia.org/wiki/Chad'),
('Congo',                    'congo',                  '🇨🇬', 'Central Africa',  'https://en.wikipedia.org/wiki/Republic_of_the_Congo'),
('DR Congo',                 'rdcongo',                '🇨🇩', 'Central Africa',  'https://en.wikipedia.org/wiki/Democratic_Republic_of_the_Congo'),
('Djibouti',                 'djibouti',               '🇩🇯', 'East Africa',     'https://en.wikipedia.org/wiki/Djibouti'),
('Egypt',                    'egypt',                  '🇪🇬', 'North Africa',    'https://en.wikipedia.org/wiki/Egypt'),
('Equatorial Guinea',        'equatorialguinea',       '🇬🇶', 'Central Africa',  'https://en.wikipedia.org/wiki/Equatorial_Guinea'),
('Eritrea',                  'eritrea',                '🇪🇷', 'East Africa',     'https://en.wikipedia.org/wiki/Eritrea'),
('Ethiopia',                 'ethiopia',               '🇪🇹', 'East Africa',     'https://en.wikipedia.org/wiki/Ethiopia'),
('Gabon',                    'gabon',                  '🇬🇦', 'Central Africa',  'https://en.wikipedia.org/wiki/Gabon'),
('Gambia',                   'gambia',                 '🇬🇲', 'West Africa',     'https://en.wikipedia.org/wiki/The_Gambia'),
('Ghana',                    'ghana',                  '🇬🇭', 'West Africa',     'https://en.wikipedia.org/wiki/Ghana'),
('Guinea',                   'guinea',                 '🇬🇳', 'West Africa',     'https://en.wikipedia.org/wiki/Guinea'),
('Guinea-Bissau',            'guineabissau',           '🇬🇼', 'West Africa',     'https://en.wikipedia.org/wiki/Guinea-Bissau'),
('Ivory Coast',              'ivorycoast',             '🇨🇮', 'West Africa',     'https://en.wikipedia.org/wiki/Ivory_Coast'),
('Kenya',                    'kenya',                  '🇰🇪', 'East Africa',     'https://en.wikipedia.org/wiki/Kenya'),
('Liberia',                  'liberia',                '🇱🇷', 'West Africa',     'https://en.wikipedia.org/wiki/Liberia'),
('Libya',                    'libya',                  '🇱🇾', 'North Africa',    'https://en.wikipedia.org/wiki/Libya'),
('Malawi',                   'malawi',                 '🇲🇼', 'East Africa',     'https://en.wikipedia.org/wiki/Malawi'),
('Mali',                     'mali',                   '🇲🇱', 'West Africa',     'https://en.wikipedia.org/wiki/Mali'),
('Mauritania',               'mauritania',             '🇲🇷', 'West Africa',     'https://en.wikipedia.org/wiki/Mauritania'),
('Morocco',                  'morocco',                '🇲🇦', 'North Africa',    'https://en.wikipedia.org/wiki/Morocco'),
('Mozambique',               'mozambique',             '🇲🇿', 'East Africa',     'https://en.wikipedia.org/wiki/Mozambique'),
('Namibia',                  'namibia',                '🇳🇦', 'Southern Africa', 'https://en.wikipedia.org/wiki/Namibia'),
('Niger',                    'niger',                  '🇳🇪', 'West Africa',     'https://en.wikipedia.org/wiki/Niger'),
('Nigeria',                  'nigeria',                '🇳🇬', 'West Africa',     'https://en.wikipedia.org/wiki/Nigeria'),
('Rwanda',                   'rwanda',                 '🇷🇼', 'East Africa',     'https://en.wikipedia.org/wiki/Rwanda'),
('Senegal',                  'senegal',                '🇸🇳', 'West Africa',     'https://en.wikipedia.org/wiki/Senegal'),
('Sierra Leone',             'sierraleone',            '🇸🇱', 'West Africa',     'https://en.wikipedia.org/wiki/Sierra_Leone'),
('Somalia',                  'somalia',                '🇸🇴', 'East Africa',     'https://en.wikipedia.org/wiki/Somalia'),
('South Africa',             'southafrica',            '🇿🇦', 'Southern Africa', 'https://en.wikipedia.org/wiki/South_Africa'),
('South Sudan',              'southsudan',             '🇸🇸', 'East Africa',     'https://en.wikipedia.org/wiki/South_Sudan'),
('Sudan',                    'sudan',                  '🇸🇩', 'North Africa',    'https://en.wikipedia.org/wiki/Sudan'),
('Eswatini',                 'swaziland',              '🇸🇿', 'Southern Africa', 'https://en.wikipedia.org/wiki/Eswatini'),
('Tanzania',                 'tanzania',               '🇹🇿', 'East Africa',     'https://en.wikipedia.org/wiki/Tanzania'),
('Togo',                     'togo',                   '🇹🇬', 'West Africa',     'https://en.wikipedia.org/wiki/Togo'),
('Tunisia',                  'tunisia',                '🇹🇳', 'North Africa',    'https://en.wikipedia.org/wiki/Tunisia'),
('Uganda',                   'uganda',                 '🇺🇬', 'East Africa',     'https://en.wikipedia.org/wiki/Uganda'),
('Western Sahara',           'westernsahara',          '🇪🇭', 'North Africa',    'https://en.wikipedia.org/wiki/Western_Sahara'),
('Zambia',                   'zambia',                 '🇿🇲', 'East Africa',     'https://en.wikipedia.org/wiki/Zambia'),
('Zimbabwe',                 'zimbabwe',               '🇿🇼', 'Southern Africa', 'https://en.wikipedia.org/wiki/Zimbabwe');
```

Expected: 48 rows inserted.

- [ ] **Step 4: Create admin user**

In Supabase → Authentication → Users → Invite user → enter `dibycamael@gmail.com`. Set a strong password. This is the only admin account.

---

## Task 4: Environment + Supabase Clients + Types

**Files:**
- Create: `.env.local`, `lib/types.ts`, `lib/supabase/client.ts`, `lib/supabase/server.ts`

- [ ] **Step 1: Create .env.local**

```bash
# .env.local (never commit this file)
NEXT_PUBLIC_SUPABASE_URL=https://YOUR_PROJECT.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key_here
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key_here
UPSTASH_REDIS_REST_URL=https://YOUR_UPSTASH.upstash.io
UPSTASH_REDIS_REST_TOKEN=your_upstash_token_here
```

Add `.env.local` to `.gitignore` (already there by default in Next.js).

- [ ] **Step 2: Create types**

Create `lib/types.ts`:
```typescript
export type Country = {
  id: string
  name: string
  slug: string
  flag_emoji: string | null
  region: string | null
  background_image_url: string | null
  about_text: string | null
  wikipedia_url: string | null
  is_active: boolean
  created_at: string
}

export type Video = {
  id: string
  country_id: string
  youtube_id: string
  title: string
  created_at: string
}

export type VideoWithCountry = Video & {
  countries: Pick<Country, 'name' | 'slug' | 'flag_emoji'>
}
```

- [ ] **Step 3: Create browser Supabase client**

Create `lib/supabase/client.ts`:
```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

- [ ] **Step 4: Create server Supabase client**

Create `lib/supabase/server.ts`:
```typescript
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {}
        },
      },
    }
  )
}
```

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: add Supabase clients and TypeScript types"
```

---

## Task 5: YouTube Utilities (TDD)

**Files:**
- Create: `lib/youtube.ts`
- Create: `__tests__/lib/youtube.test.ts`

- [ ] **Step 1: Write failing tests**

Create `__tests__/lib/youtube.test.ts`:
```typescript
import { describe, it, expect } from 'vitest'
import { parseYouTubeId } from '@/lib/youtube'

describe('parseYouTubeId', () => {
  it('parses watch URL', () => {
    expect(parseYouTubeId('https://www.youtube.com/watch?v=abc123XYZ')).toBe('abc123XYZ')
  })
  it('parses short URL', () => {
    expect(parseYouTubeId('https://youtu.be/abc123XYZ')).toBe('abc123XYZ')
  })
  it('parses embed URL', () => {
    expect(parseYouTubeId('https://www.youtube.com/embed/abc123XYZ')).toBe('abc123XYZ')
  })
  it('parses URL with extra params', () => {
    expect(parseYouTubeId('https://www.youtube.com/watch?v=abc123XYZ&t=30')).toBe('abc123XYZ')
  })
  it('returns null for invalid URL', () => {
    expect(parseYouTubeId('https://vimeo.com/123')).toBeNull()
  })
  it('returns null for empty string', () => {
    expect(parseYouTubeId('')).toBeNull()
  })
})
```

- [ ] **Step 2: Run — verify it fails**

```bash
npm test -- youtube.test.ts
```
Expected: FAIL — `Cannot find module '@/lib/youtube'`

- [ ] **Step 3: Implement youtube.ts**

Create `lib/youtube.ts`:
```typescript
export function parseYouTubeId(url: string): string | null {
  if (!url) return null
  const patterns = [
    /youtube\.com\/watch\?.*v=([a-zA-Z0-9_-]{11})/,
    /youtu\.be\/([a-zA-Z0-9_-]{11})/,
    /youtube\.com\/embed\/([a-zA-Z0-9_-]{11})/,
  ]
  for (const pattern of patterns) {
    const match = url.match(pattern)
    if (match) return match[1]
  }
  return null
}

export async function fetchYouTubeMeta(
  videoId: string
): Promise<{ title: string } | null> {
  try {
    const res = await fetch(
      `https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=${videoId}&format=json`
    )
    if (!res.ok) return null
    const data = await res.json()
    return { title: data.title ?? '' }
  } catch {
    return null
  }
}
```

- [ ] **Step 4: Run — verify tests pass**

```bash
npm test -- youtube.test.ts
```
Expected: 6 tests PASS

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: add YouTube URL parsing utilities with tests"
```

---

## Task 6: Root Layout + NavBar

**Files:**
- Modify: `app/layout.tsx`
- Create: `components/NavBar.tsx`

- [ ] **Step 1: Create NavBar**

Create `components/NavBar.tsx`:
```tsx
'use client'
import Link from 'next/link'
import { usePathname } from 'next/navigation'

export function NavBar() {
  const path = usePathname()
  const links = [
    { href: '/',       label: 'Map'   },
    { href: '/quiz',   label: 'Quiz'  },
    { href: '/about',  label: 'About' },
  ]
  return (
    <header className="bg-dark-800 border-b border-white/5 sticky top-0 z-50">
      <div className="max-w-6xl mx-auto px-4 h-14 flex items-center justify-between">
        <Link href="/" className="font-black text-lg tracking-tight">
          <span className="text-pan-red">A</span>
          <span className="text-pan-green">f</span>
          <span className="text-pan-gold">r</span>
          <span className="text-white">icaTruth</span>
        </Link>
        <nav className="flex gap-6">
          {links.map(l => (
            <Link
              key={l.href}
              href={l.href}
              className={`text-sm font-medium transition-colors ${
                path === l.href
                  ? 'text-pan-gold'
                  : 'text-white/60 hover:text-white'
              }`}
            >
              {l.label}
            </Link>
          ))}
        </nav>
      </div>
    </header>
  )
}
```

- [ ] **Step 2: Update root layout**

Replace `app/layout.tsx`:
```tsx
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'
import { NavBar } from '@/components/NavBar'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'AfricaTruth — Political History of Africa',
  description: 'Explore the political history of all 54 African nations through curated videos.',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className={`${inter.className} bg-dark-900 text-white min-h-screen`}>
        <NavBar />
        <main>{children}</main>
      </body>
    </html>
  )
}
```

- [ ] **Step 3: Verify**

```bash
npm run dev
```
Open http://localhost:3000 — should show NavBar with Pan-African logo on dark background.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat: add root layout and NavBar"
```

---

## Task 7: Map Page (Home)

**Files:**
- Modify: `app/page.tsx`
- Create: `components/AfricaMap.tsx`
- Create: `components/SearchBar.tsx`
- Create: `public/scripts/mapdata.js` (updated from existing)
- Create: `public/scripts/worldmap.js` (copy from existing)

- [ ] **Step 1: Copy simplemaps scripts to public/**

```bash
mkdir -p public/scripts
cp ../countries/../"js files"/worldmap.js public/scripts/worldmap.js
```

Then create `public/scripts/mapdata.js` by copying the existing `mapdata.js` and updating every `url` field from `nigeria.html` → `/countries/nigeria`, etc. The pattern: replace `"url": "slug.html"` with `"url": "/countries/slug"` for all 49 countries. (Do a find-and-replace: `s/\.html"/"/g` on all url values, then prepend `/countries/`.)

- [ ] **Step 2: Create AfricaMap component**

Create `components/AfricaMap.tsx`:
```tsx
'use client'
import Script from 'next/script'
import { useRouter } from 'next/navigation'
import { useEffect } from 'react'

declare global {
  interface Window {
    simplemaps_worldmap: { load: () => void; hooks: { click_state: (id: string) => void } }
    simplemaps_worldmap_mapdata: { state_specific: Record<string, { name: string; url: string }> }
  }
}

export function AfricaMap() {
  const router = useRouter()

  useEffect(() => {
    if (typeof window !== 'undefined' && window.simplemaps_worldmap) {
      window.simplemaps_worldmap.hooks.click_state = (id: string) => {
        const state = window.simplemaps_worldmap_mapdata?.state_specific?.[id]
        if (state?.url) router.push(state.url)
      }
      window.simplemaps_worldmap.load()
    }
  }, [router])

  return (
    <>
      <Script src="/scripts/mapdata.js" strategy="beforeInteractive" />
      <Script src="/scripts/worldmap.js" strategy="beforeInteractive" />
      <div id="map" className="w-full" style={{ minHeight: '500px' }} />
    </>
  )
}
```

- [ ] **Step 3: Create SearchBar**

Create `components/SearchBar.tsx`:
```tsx
'use client'
import { useState, useEffect, useRef } from 'react'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase/client'
import type { Country } from '@/lib/types'

export function SearchBar() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState<Country[]>([])
  const [open, setOpen] = useState(false)
  const router = useRouter()
  const ref = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (query.length < 2) { setResults([]); return }
    const supabase = createClient()
    supabase
      .from('countries')
      .select('id,name,slug,flag_emoji')
      .ilike('name', `%${query}%`)
      .limit(8)
      .then(({ data }) => setResults(data ?? []))
  }, [query])

  useEffect(() => {
    function handleClick(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false)
    }
    document.addEventListener('mousedown', handleClick)
    return () => document.removeEventListener('mousedown', handleClick)
  }, [])

  return (
    <div ref={ref} className="relative w-full max-w-md mx-auto">
      <input
        type="text"
        placeholder="Search any country..."
        value={query}
        onChange={e => { setQuery(e.target.value); setOpen(true) }}
        className="w-full bg-white/10 border border-white/20 rounded-full px-5 py-3 text-sm text-white placeholder-white/40 focus:outline-none focus:border-pan-gold"
      />
      {open && results.length > 0 && (
        <ul className="absolute top-full mt-2 w-full bg-dark-800 border border-white/10 rounded-xl overflow-hidden z-50 shadow-2xl">
          {results.map(c => (
            <li key={c.id}>
              <button
                className="w-full px-4 py-3 flex items-center gap-3 hover:bg-white/5 text-left text-sm"
                onClick={() => { router.push(`/countries/${c.slug}`); setOpen(false); setQuery('') }}
              >
                <span>{c.flag_emoji}</span>
                <span>{c.name}</span>
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

- [ ] **Step 4: Build home page**

Replace `app/page.tsx`:
```tsx
import { AfricaMap } from '@/components/AfricaMap'
import { SearchBar } from '@/components/SearchBar'

export default function HomePage() {
  return (
    <div className="relative min-h-screen flex flex-col">
      <div className="relative z-10 pt-12 pb-6 px-4 text-center">
        <p className="text-pan-gold text-xs uppercase tracking-widest mb-2">Explore</p>
        <h1 className="text-4xl font-black mb-2">
          The Political History<br />of Africa
        </h1>
        <p className="text-white/50 text-sm mb-6">54 nations. One platform.</p>
        <SearchBar />
      </div>
      <div className="flex-1 px-4 pb-8">
        <AfricaMap />
      </div>
    </div>
  )
}
```

- [ ] **Step 5: Verify map loads**

```bash
npm run dev
```
Open http://localhost:3000 — map should render, clicking a country should navigate to `/countries/[slug]`.

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: home page with interactive map and search"
```

---

## Task 8: Country Page

**Files:**
- Create: `app/countries/[slug]/page.tsx`
- Create: `components/CountryHero.tsx`
- Create: `components/VideoGrid.tsx`
- Create: `components/VideoCard.tsx`

- [ ] **Step 1: Create VideoCard**

Create `components/VideoCard.tsx`:
```tsx
type Props = { youtubeId: string; title: string }

export function VideoCard({ youtubeId, title }: Props) {
  return (
    <a
      href={`https://www.youtube.com/watch?v=${youtubeId}`}
      target="_blank"
      rel="noopener noreferrer"
      className="card-dark overflow-hidden block hover:border-pan-gold/40 transition-colors group"
    >
      <div className="relative aspect-video bg-dark-700 overflow-hidden">
        <img
          src={`https://img.youtube.com/vi/${youtubeId}/mqdefault.jpg`}
          alt={title}
          className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
          loading="lazy"
        />
        <div className="absolute inset-0 flex items-center justify-center bg-black/30 group-hover:bg-black/10 transition-colors">
          <div className="w-10 h-10 bg-pan-red rounded-full flex items-center justify-center">
            <span className="text-white text-sm pl-0.5">▶</span>
          </div>
        </div>
      </div>
      <p className="p-3 text-sm text-white/80 leading-snug line-clamp-2">{title}</p>
    </a>
  )
}
```

- [ ] **Step 2: Create VideoGrid**

Create `components/VideoGrid.tsx`:
```tsx
import { VideoCard } from './VideoCard'
import type { Video } from '@/lib/types'

type Props = { videos: Video[] }

export function VideoGrid({ videos }: Props) {
  if (videos.length === 0) {
    return <p className="text-white/40 text-sm text-center py-12">No videos yet for this country.</p>
  }
  return (
    <div className="grid grid-cols-2 gap-3 sm:grid-cols-2 md:grid-cols-3">
      {videos.map(v => (
        <VideoCard key={v.id} youtubeId={v.youtube_id} title={v.title} />
      ))}
    </div>
  )
}
```

- [ ] **Step 3: Create CountryHero**

Create `components/CountryHero.tsx`:
```tsx
import type { Country } from '@/lib/types'

type Props = { country: Country; videoCount: number }

export function CountryHero({ country, videoCount }: Props) {
  return (
    <div className="relative overflow-hidden rounded-2xl mb-6" style={{
      background: 'linear-gradient(135deg, rgba(230,57,70,0.15) 0%, rgba(45,198,83,0.1) 100%)',
      border: '1px solid rgba(255,214,10,0.15)',
    }}>
      <div className="p-6">
        <div className="flex items-center gap-4 mb-3">
          <span className="text-5xl">{country.flag_emoji}</span>
          <div>
            <h1 className="text-2xl font-black">{country.name}</h1>
            <p className="text-pan-gold text-xs uppercase tracking-widest">{country.region}</p>
          </div>
        </div>
        <div className="flex gap-6 text-sm text-white/50">
          <span><strong className="text-white">{videoCount}</strong> videos</span>
        </div>
        {country.wikipedia_url && (
          <a
            href={country.wikipedia_url}
            target="_blank"
            rel="noopener noreferrer"
            className="mt-4 inline-block text-xs text-pan-gold/70 hover:text-pan-gold border border-pan-gold/20 rounded-full px-4 py-1.5"
          >
            📖 About {country.name} →
          </a>
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create country page**

Create `app/countries/[slug]/page.tsx`:
```tsx
import { notFound } from 'next/navigation'
import { createClient } from '@/lib/supabase/server'
import { CountryHero } from '@/components/CountryHero'
import { VideoGrid } from '@/components/VideoGrid'

type Props = { params: Promise<{ slug: string }> }

export const revalidate = 60

export async function generateMetadata({ params }: Props) {
  const { slug } = await params
  const supabase = await createClient()
  const { data } = await supabase.from('countries').select('name').eq('slug', slug).single()
  return { title: data ? `Political History of ${data.name} — AfricaTruth` : 'AfricaTruth' }
}

export default async function CountryPage({ params }: Props) {
  const { slug } = await params
  const supabase = await createClient()

  const { data: country } = await supabase
    .from('countries')
    .select('*')
    .eq('slug', slug)
    .eq('is_active', true)
    .single()

  if (!country) notFound()

  const { data: videos } = await supabase
    .from('videos')
    .select('*')
    .eq('country_id', country.id)
    .order('created_at', { ascending: true })

  return (
    <div className="max-w-4xl mx-auto px-4 py-6">
      <CountryHero country={country} videoCount={videos?.length ?? 0} />
      <h2 className="text-xs uppercase tracking-widest text-white/40 mb-4">Political History Videos</h2>
      <VideoGrid videos={videos ?? []} />
    </div>
  )
}
```

- [ ] **Step 5: Verify**

```bash
npm run dev
```
Open http://localhost:3000/countries/nigeria — hero, video grid, and Wikipedia link should all render. (Videos won't show until migration in Task 14.)

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: country page with hero, video grid, and VideoCard"
```

---

## Task 9: Quiz Page

**Files:**
- Create: `app/quiz/page.tsx`
- Create: `components/QuizGame.tsx`

- [ ] **Step 1: Create QuizGame component**

Create `components/QuizGame.tsx`:
```tsx
'use client'
import { useEffect, useRef, useState } from 'react'
import Script from 'next/script'

declare global {
  interface Window {
    simplemaps_worldmap: { load: () => void; hooks: { click_state: (id: string) => void } }
    simplemaps_worldmap_mapdata: { state_specific: Record<string, { chose: string }> }
  }
}

const EXCLUDED = ['MU', 'SC', 'KM', 'CV', 'ST']

type Difficulty = 'beginner' | 'intermediate' | 'advanced'
type Phase = 'difficulty' | 'count' | 'playing' | 'done'

const TIME: Record<Difficulty, number> = { beginner: 30, intermediate: 20, advanced: 10 }

export function QuizGame() {
  const [phase, setPhase] = useState<Phase>('difficulty')
  const [difficulty, setDifficulty] = useState<Difficulty>('beginner')
  const [total, setTotal] = useState(5)
  const [current, setCurrent] = useState(0)
  const [score, setScore] = useState(0)
  const [target, setTarget] = useState('')
  const [feedback, setFeedback] = useState<'correct' | 'wrong' | null>(null)
  const [timeLeft, setTimeLeft] = useState(30)
  const [scriptsLoaded, setScriptsLoaded] = useState(false)
  const timerRef = useRef<ReturnType<typeof setInterval> | null>(null)
  const inProgressRef = useRef(false)

  function getCountries() {
    if (!window.simplemaps_worldmap_mapdata) return []
    return Object.keys(window.simplemaps_worldmap_mapdata.state_specific).filter(
      id => !EXCLUDED.includes(id)
    )
  }

  function nextQuestion(scoreVal: number, currentVal: number) {
    if (currentVal >= total) { setPhase('done'); return }
    const countries = getCountries()
    const next = countries[Math.floor(Math.random() * countries.length)]
    setTarget(next)
    setFeedback(null)
    setTimeLeft(TIME[difficulty])
    inProgressRef.current = true
    setCurrent(currentVal + 1)

    if (timerRef.current) clearInterval(timerRef.current)
    timerRef.current = setInterval(() => {
      setTimeLeft(t => {
        if (t <= 1) {
          clearInterval(timerRef.current!)
          handleAnswer(null, scoreVal, currentVal + 1)
          return 0
        }
        return t - 1
      })
    }, 1000)
  }

  function handleAnswer(selected: string | null, scoreVal: number, currentVal: number) {
    if (!inProgressRef.current) return
    inProgressRef.current = false
    if (timerRef.current) clearInterval(timerRef.current)
    const correct = selected === target
    const newScore = correct ? scoreVal + 1 : scoreVal
    setFeedback(correct ? 'correct' : 'wrong')
    setScore(newScore)
    setTimeout(() => nextQuestion(newScore, currentVal), 1000)
  }

  useEffect(() => {
    if (!scriptsLoaded || phase !== 'playing') return
    window.simplemaps_worldmap.hooks.click_state = (id: string) => {
      if (inProgressRef.current) handleAnswer(id, score, current)
    }
    window.simplemaps_worldmap.load()
    nextQuestion(0, 0)
    return () => { if (timerRef.current) clearInterval(timerRef.current) }
  }, [scriptsLoaded, phase])

  const targetName = target && window.simplemaps_worldmap_mapdata
    ? window.simplemaps_worldmap_mapdata.state_specific[target]?.chose ?? target
    : ''

  const feedback_msg = (() => {
    if (total === 5) {
      if (score >= 4) return "Excellent! You really know Africa's geography!"
      if (score >= 3) return "Good job! Keep exploring the map."
      return "You still think the capital of Africa is Nigeria. Do better!"
    }
    if (score >= 8) return "Outstanding! You're an Africa geography expert!"
    if (score >= 5) return "Come on! You can do better."
    return "Looks like you think Africa is a country. Do better."
  })()

  return (
    <>
      <Script src="/scripts/mapdata1.js" strategy="beforeInteractive" onLoad={() => setScriptsLoaded(true)} />
      <Script src="/scripts/worldmap1.js" strategy="beforeInteractive" />

      <div className="max-w-4xl mx-auto px-4 py-6">
        {phase === 'difficulty' && (
          <div className="card-dark p-8 text-center max-w-md mx-auto">
            <h2 className="text-xl font-bold mb-6">Select Difficulty</h2>
            {(['beginner', 'intermediate', 'advanced'] as Difficulty[]).map(d => (
              <button key={d} onClick={() => { setDifficulty(d); setPhase('count') }}
                className="btn-primary w-full mb-3 capitalize block">
                {d} ({TIME[d]}s per question)
              </button>
            ))}
          </div>
        )}

        {phase === 'count' && (
          <div className="card-dark p-8 text-center max-w-md mx-auto">
            <h2 className="text-xl font-bold mb-6">How many questions?</h2>
            {[5, 10].map(n => (
              <button key={n} onClick={() => { setTotal(n); setPhase('playing') }}
                className="btn-primary w-full mb-3 block">
                {n} Questions
              </button>
            ))}
            <button onClick={() => setPhase('difficulty')} className="text-white/40 text-sm mt-2">← Change difficulty</button>
          </div>
        )}

        {phase === 'playing' && (
          <>
            <div className="card-dark p-4 text-center mb-4">
              <p className="text-sm text-white/50 mb-1">Question {current} of {total}</p>
              <p className="text-lg font-bold">Find <span className="text-pan-gold">{targetName}</span> on the map</p>
              <p className={`text-2xl font-black mt-1 ${timeLeft <= 5 ? 'text-pan-red' : 'text-white'}`}>
                {timeLeft}s
              </p>
              {feedback && (
                <p className={`text-sm font-bold mt-1 ${feedback === 'correct' ? 'text-pan-green' : 'text-pan-red'}`}>
                  {feedback === 'correct' ? '✓ Correct!' : '✗ Wrong'}
                </p>
              )}
            </div>
            <div id="map" className="w-full" style={{ minHeight: '460px' }} />
          </>
        )}

        {phase === 'done' && (
          <div className="card-dark p-8 text-center max-w-md mx-auto">
            <p className="text-4xl font-black mb-2">{score}/{total}</p>
            <p className="text-white/70 mb-6">{feedback_msg}</p>
            <button onClick={() => { setPhase('difficulty'); setScore(0); setCurrent(0) }}
              className="btn-primary">
              Play Again
            </button>
          </div>
        )}
      </div>
    </>
  )
}
```

- [ ] **Step 2: Copy quiz map scripts**

```bash
cp ../countries/../"js files"/worldmap1.js public/scripts/worldmap1.js
cp ../countries/../"js files"/mapdata1.js  public/scripts/mapdata1.js
```

- [ ] **Step 3: Create quiz page**

Create `app/quiz/page.tsx`:
```tsx
import { QuizGame } from '@/components/QuizGame'

export const metadata = { title: 'Map Quiz — AfricaTruth' }

export default function QuizPage() {
  return (
    <div className="py-4">
      <h1 className="text-center text-2xl font-black mb-6">Map Quiz</h1>
      <QuizGame />
    </div>
  )
}
```

- [ ] **Step 4: Verify quiz works**

```bash
npm run dev
```
Open http://localhost:3000/quiz — select difficulty → question count → map appears → clicking a country checks the answer.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: quiz page with difficulty levels and countdown timer"
```

---

## Task 10: About + Contact Pages

**Files:**
- Create: `app/about/page.tsx`
- Create: `app/contact/page.tsx`

- [ ] **Step 1: Create About page**

Create `app/about/page.tsx`:
```tsx
export const metadata = { title: 'About — AfricaTruth' }

export default function AboutPage() {
  return (
    <div className="max-w-2xl mx-auto px-4 py-12">
      <h1 className="text-3xl font-black mb-6">About AfricaTruth</h1>
      <div className="prose prose-invert space-y-4 text-white/80 leading-relaxed">
        <p>AfricaTruth is an educational platform dedicated to making the political history of Africa accessible, engaging, and honest.</p>
        <p>Through curated YouTube videos covering 54 nations — from colonial independence movements to modern coups, from liberation heroes to authoritarian regimes — the platform aims to give people a clear-eyed view of how Africa's political landscape was shaped.</p>
        <p>Built by <strong className="text-white">Macarthur Diby</strong>, a CS student with a deep passion for African history and political education.</p>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create Contact page**

Create `app/contact/page.tsx`:
```tsx
export const metadata = { title: 'Contact — AfricaTruth' }

export default function ContactPage() {
  return (
    <div className="max-w-2xl mx-auto px-4 py-12">
      <h1 className="text-3xl font-black mb-6">Contact</h1>
      <div className="card-dark p-6 space-y-4">
        <div className="flex items-center gap-3">
          <span className="text-pan-gold">✉</span>
          <a href="mailto:dibycamael@gmail.com" className="text-white/80 hover:text-white">dibycamael@gmail.com</a>
        </div>
        <div className="flex items-center gap-3">
          <span className="text-pan-gold">⌥</span>
          <a href="https://github.com/Camaelo26" target="_blank" rel="noopener noreferrer" className="text-white/80 hover:text-white">github.com/Camaelo26</a>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat: about and contact pages"
```

---

## Task 11: Admin Login

**Files:**
- Create: `app/admin/login/page.tsx`

- [ ] **Step 1: Create login page**

Create `app/admin/login/page.tsx`:
```tsx
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase/client'

export default function AdminLogin() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')
  const [loading, setLoading] = useState(false)
  const router = useRouter()

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    setLoading(true)
    setError('')
    const supabase = createClient()
    const { error: err } = await supabase.auth.signInWithPassword({ email, password })
    if (err) {
      setError('Invalid credentials.')
      setLoading(false)
    } else {
      router.push('/admin')
      router.refresh()
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center px-4">
      <div className="card-dark p-8 w-full max-w-sm">
        <h1 className="text-xl font-black mb-1">⚡ Admin</h1>
        <p className="text-white/40 text-sm mb-6">AfricaTruth dashboard</p>
        <form onSubmit={handleSubmit} className="space-y-4">
          <input
            type="email"
            placeholder="Email"
            value={email}
            onChange={e => setEmail(e.target.value)}
            required
            className="w-full bg-white/5 border border-white/10 rounded-lg px-4 py-3 text-sm text-white placeholder-white/30 focus:outline-none focus:border-pan-gold"
          />
          <input
            type="password"
            placeholder="Password"
            value={password}
            onChange={e => setPassword(e.target.value)}
            required
            className="w-full bg-white/5 border border-white/10 rounded-lg px-4 py-3 text-sm text-white placeholder-white/30 focus:outline-none focus:border-pan-gold"
          />
          {error && <p className="text-pan-red text-sm">{error}</p>}
          <button type="submit" disabled={loading} className="btn-primary w-full">
            {loading ? 'Logging in...' : 'Log In'}
          </button>
        </form>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add -A && git commit -m "feat: admin login page"
```

---

## Task 12: Middleware — Auth + Rate Limit Setup

**Files:**
- Create: `middleware.ts`
- Create: `lib/rate-limit.ts`

- [ ] **Step 1: Create rate-limit helpers**

Create `lib/rate-limit.ts`:
```typescript
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
})

export const apiRatelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(60, '60 s'),
  prefix: 'rl:api',
})

export const adminRatelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, '60 s'),
  prefix: 'rl:admin',
})
```

- [ ] **Step 2: Create middleware**

Create `middleware.ts`:
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerClient } from '@supabase/ssr'
import { apiRatelimit, adminRatelimit } from '@/lib/rate-limit'

export async function middleware(req: NextRequest) {
  const ip = req.headers.get('x-forwarded-for')?.split(',')[0] ?? '127.0.0.1'
  const path = req.nextUrl.pathname

  // Rate limiting
  if (path.startsWith('/api/')) {
    const { success } = await apiRatelimit.limit(ip)
    if (!success) return new NextResponse('Too many requests', { status: 429 })
  }
  if (path.startsWith('/admin')) {
    const { success } = await adminRatelimit.limit(ip)
    if (!success) return new NextResponse('Too many requests', { status: 429 })
  }

  // Admin auth guard
  if (path.startsWith('/admin') && !path.startsWith('/admin/login')) {
    let res = NextResponse.next()
    const supabase = createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      {
        cookies: {
          getAll: () => req.cookies.getAll(),
          setAll: (cookiesToSet) => {
            cookiesToSet.forEach(({ name, value, options }) =>
              res.cookies.set(name, value, options)
            )
          },
        },
      }
    )
    const { data: { user } } = await supabase.auth.getUser()
    if (!user) {
      return NextResponse.redirect(new URL('/admin/login', req.url))
    }
    return res
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/admin/:path*', '/api/:path*'],
}
```

- [ ] **Step 3: Set up Upstash**

Go to https://upstash.com → Create Redis database (free tier) → Copy REST URL and Token → add to `.env.local`.

- [ ] **Step 4: Verify protection**

```bash
npm run dev
```
Open http://localhost:3000/admin — should redirect to `/admin/login`. Login with the admin credentials created in Task 3 → should land on `/admin` (currently 404, that's fine for now).

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: middleware with rate limiting and admin auth guard"
```

---

## Task 13: API Routes

**Files:**
- Create: `app/api/youtube-meta/route.ts`
- Create: `app/api/videos/route.ts`
- Create: `__tests__/api/youtube-meta.test.ts`

- [ ] **Step 1: Write failing test for youtube-meta route**

Create `__tests__/api/youtube-meta.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest'
import { parseYouTubeId } from '@/lib/youtube'

describe('YouTube URL validation in API', () => {
  it('extracts valid YouTube ID', () => {
    const id = parseYouTubeId('https://www.youtube.com/watch?v=dQw4w9WgXcQ')
    expect(id).toBe('dQw4w9WgXcQ')
  })
  it('rejects non-YouTube URL', () => {
    const id = parseYouTubeId('https://vimeo.com/123456')
    expect(id).toBeNull()
  })
})
```

- [ ] **Step 2: Run — verify passes (reuses lib)**

```bash
npm test -- youtube-meta.test.ts
```
Expected: 2 tests PASS.

- [ ] **Step 3: Create youtube-meta route**

Create `app/api/youtube-meta/route.ts`:
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { parseYouTubeId, fetchYouTubeMeta } from '@/lib/youtube'

export async function GET(req: NextRequest) {
  const url = req.nextUrl.searchParams.get('url') ?? ''
  const id = parseYouTubeId(url)
  if (!id) {
    return NextResponse.json({ error: 'Invalid YouTube URL' }, { status: 400 })
  }
  const meta = await fetchYouTubeMeta(id)
  if (!meta) {
    return NextResponse.json({ error: 'Could not fetch video metadata' }, { status: 404 })
  }
  return NextResponse.json({ id, title: meta.title })
}
```

- [ ] **Step 4: Create videos CRUD route**

Create `app/api/videos/route.ts`:
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import { parseYouTubeId } from '@/lib/youtube'

async function getAdminClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: () => {},
      },
    }
  )
}

async function requireAdmin() {
  const cookieStore = await cookies()
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: () => {},
      },
    }
  )
  const { data: { user } } = await supabase.auth.getUser()
  return user
}

export async function POST(req: NextRequest) {
  const user = await requireAdmin()
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const { youtubeUrl, countryId, title } = await req.json()
  const youtubeId = parseYouTubeId(youtubeUrl ?? '')
  if (!youtubeId) return NextResponse.json({ error: 'Invalid YouTube URL' }, { status: 400 })
  if (!countryId || typeof countryId !== 'string') {
    return NextResponse.json({ error: 'countryId required' }, { status: 400 })
  }
  if (!title || typeof title !== 'string' || title.trim().length === 0) {
    return NextResponse.json({ error: 'title required' }, { status: 400 })
  }

  const supabase = await getAdminClient()
  const { data, error } = await supabase
    .from('videos')
    .insert({ youtube_id: youtubeId, country_id: countryId, title: title.trim() })
    .select()
    .single()

  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data, { status: 201 })
}

export async function DELETE(req: NextRequest) {
  const user = await requireAdmin()
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const { id } = await req.json()
  if (!id) return NextResponse.json({ error: 'id required' }, { status: 400 })

  const supabase = await getAdminClient()
  const { error } = await supabase.from('videos').delete().eq('id', id)
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json({ success: true })
}
```

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: API routes for YouTube meta and video CRUD"
```

---

## Task 14: Admin Dashboard

**Files:**
- Create: `app/admin/page.tsx`
- Create: `components/AdminVideoForm.tsx`
- Create: `components/RecentVideos.tsx`

- [ ] **Step 1: Create AdminVideoForm**

Create `components/AdminVideoForm.tsx`:
```tsx
'use client'
import { useState, useEffect } from 'react'
import type { Country } from '@/lib/types'

type Props = { countries: Country[]; onAdded: () => void }

export function AdminVideoForm({ countries, onAdded }: Props) {
  const [url, setUrl] = useState('')
  const [title, setTitle] = useState('')
  const [countryId, setCountryId] = useState('')
  const [fetching, setFetching] = useState(false)
  const [saving, setSaving] = useState(false)
  const [error, setError] = useState('')

  useEffect(() => {
    if (!url.includes('youtube') && !url.includes('youtu.be')) return
    const t = setTimeout(async () => {
      setFetching(true)
      const res = await fetch(`/api/youtube-meta?url=${encodeURIComponent(url)}`)
      if (res.ok) {
        const data = await res.json()
        setTitle(data.title)
      }
      setFetching(false)
    }, 600)
    return () => clearTimeout(t)
  }, [url])

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    if (!countryId || !title.trim()) { setError('Please fill all fields.'); return }
    setSaving(true)
    setError('')
    const res = await fetch('/api/videos', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ youtubeUrl: url, countryId, title }),
    })
    if (res.ok) {
      setUrl(''); setTitle(''); setCountryId('')
      onAdded()
    } else {
      const d = await res.json()
      setError(d.error ?? 'Failed to add video.')
    }
    setSaving(false)
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label className="text-xs uppercase tracking-widest text-white/40 mb-1 block">YouTube URL</label>
        <input
          type="url"
          placeholder="https://youtube.com/watch?v=..."
          value={url}
          onChange={e => setUrl(e.target.value)}
          required
          className="w-full bg-white/5 border border-white/10 rounded-lg px-4 py-3 text-sm text-white placeholder-white/30 focus:outline-none focus:border-pan-gold"
        />
      </div>
      {fetching && <p className="text-xs text-pan-gold/70 animate-pulse">Fetching title...</p>}
      <div>
        <label className="text-xs uppercase tracking-widest text-white/40 mb-1 block">Country</label>
        <select
          value={countryId}
          onChange={e => setCountryId(e.target.value)}
          required
          className="w-full bg-white/5 border border-white/10 rounded-lg px-4 py-3 text-sm text-white focus:outline-none focus:border-pan-gold"
        >
          <option value="">Select a country...</option>
          {countries.map(c => (
            <option key={c.id} value={c.id}>{c.flag_emoji} {c.name}</option>
          ))}
        </select>
      </div>
      <div>
        <label className="text-xs uppercase tracking-widest text-white/40 mb-1 block">Title (auto-filled)</label>
        <input
          type="text"
          placeholder="Video title"
          value={title}
          onChange={e => setTitle(e.target.value)}
          required
          className="w-full bg-white/5 border border-white/10 rounded-lg px-4 py-3 text-sm text-white placeholder-white/30 focus:outline-none focus:border-pan-gold"
        />
      </div>
      {error && <p className="text-pan-red text-sm">{error}</p>}
      <button type="submit" disabled={saving} className="btn-primary w-full">
        {saving ? 'Adding...' : 'Add Video →'}
      </button>
    </form>
  )
}
```

- [ ] **Step 2: Create RecentVideos**

Create `components/RecentVideos.tsx`:
```tsx
'use client'
import type { VideoWithCountry } from '@/lib/types'

type Props = { videos: VideoWithCountry[]; onDelete: (id: string) => void }

export function RecentVideos({ videos, onDelete }: Props) {
  return (
    <div>
      <p className="text-xs uppercase tracking-widest text-white/40 mb-3">Recently Added</p>
      {videos.length === 0 && <p className="text-white/30 text-sm">No videos yet.</p>}
      {videos.map(v => (
        <div key={v.id} className="flex items-center gap-3 py-3 border-b border-white/5">
          <span className="text-xl">{v.countries?.flag_emoji}</span>
          <div className="flex-1 min-w-0">
            <p className="text-sm text-white/80 truncate">{v.title}</p>
            <p className="text-xs text-white/30">{v.countries?.name}</p>
          </div>
          <button
            onClick={() => onDelete(v.id)}
            className="text-white/20 hover:text-pan-red text-lg transition-colors"
            title="Delete"
          >
            ×
          </button>
        </div>
      ))}
    </div>
  )
}
```

- [ ] **Step 3: Create admin dashboard**

Create `app/admin/page.tsx`:
```tsx
'use client'
import { useEffect, useState, useCallback } from 'react'
import { createClient } from '@/lib/supabase/client'
import { AdminVideoForm } from '@/components/AdminVideoForm'
import { RecentVideos } from '@/components/RecentVideos'
import type { Country, VideoWithCountry } from '@/lib/types'

export default function AdminPage() {
  const [countries, setCountries] = useState<Country[]>([])
  const [recentVideos, setRecentVideos] = useState<VideoWithCountry[]>([])
  const [totalVideos, setTotalVideos] = useState(0)

  const supabase = createClient()

  const loadData = useCallback(async () => {
    const [{ data: c }, { data: v }, { count }] = await Promise.all([
      supabase.from('countries').select('*').order('name'),
      supabase.from('videos').select('*, countries(name,slug,flag_emoji)').order('created_at', { ascending: false }).limit(20),
      supabase.from('videos').select('*', { count: 'exact', head: true }),
    ])
    setCountries(c ?? [])
    setRecentVideos((v ?? []) as VideoWithCountry[])
    setTotalVideos(count ?? 0)
  }, [])

  useEffect(() => { loadData() }, [loadData])

  async function handleDelete(id: string) {
    await fetch('/api/videos', {
      method: 'DELETE',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ id }),
    })
    loadData()
  }

  return (
    <div className="max-w-2xl mx-auto px-4 py-6">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-xl font-black text-pan-gold">⚡ Admin</h1>
      </div>

      <div className="grid grid-cols-2 gap-4 mb-8">
        <div className="card-dark p-4">
          <p className="text-3xl font-black text-pan-gold">{totalVideos}</p>
          <p className="text-xs text-white/40 mt-1">Total videos</p>
        </div>
        <div className="card-dark p-4">
          <p className="text-3xl font-black text-pan-gold">{countries.length}</p>
          <p className="text-xs text-white/40 mt-1">Countries</p>
        </div>
      </div>

      <div className="card-dark p-6 mb-6">
        <p className="text-xs uppercase tracking-widest text-pan-gold mb-4">+ Add New Video</p>
        <AdminVideoForm countries={countries} onAdded={loadData} />
      </div>

      <div className="card-dark p-6">
        <RecentVideos videos={recentVideos} onDelete={handleDelete} />
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Test full admin flow**

```bash
npm run dev
```
1. Go to http://localhost:3000/admin/login → log in
2. On dashboard, paste a YouTube URL → title auto-fills after ~600ms
3. Select a country → click Add Video → video appears in Recent list
4. Click × on a video → it disappears

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: admin dashboard with auto-fetch YouTube title and recent videos"
```

---

## Task 15: Security Headers

**Files:**
- Modify: `next.config.js`

- [ ] **Step 1: Add CSP and security headers**

Replace `next.config.js`:
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    domains: ['img.youtube.com', 'i.ytimg.com'],
  },
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://www.googletagmanager.com",
              "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
              "font-src 'self' https://fonts.gstatic.com",
              "frame-src https://www.youtube.com https://youtube.com",
              "img-src 'self' data: https://img.youtube.com https://i.ytimg.com",
              "connect-src 'self' https://*.supabase.co https://restcountries.com https://www.youtube.com",
            ].join('; '),
          },
          { key: 'X-Frame-Options',         value: 'DENY' },
          { key: 'X-Content-Type-Options',   value: 'nosniff' },
          { key: 'Referrer-Policy',          value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy',       value: 'camera=(), microphone=(), geolocation=()' },
        ],
      },
    ]
  },
}

module.exports = nextConfig
```

- [ ] **Step 2: Verify headers**

```bash
npm run dev
```
Open http://localhost:3000, open DevTools → Network → click any request → Response Headers — confirm `Content-Security-Policy` and `X-Frame-Options` are present.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat: CSP and security headers"
```

---

## Task 16: Data Migration Script

**Files:**
- Create: `scripts/migrate.ts`

- [ ] **Step 1: Install script dependencies**

```bash
npm install -D tsx cheerio
```

- [ ] **Step 2: Create migration script**

Create `scripts/migrate.ts`:
```typescript
import * as fs from 'fs'
import * as path from 'path'
import * as cheerio from 'cheerio'
import { createClient } from '@supabase/supabase-js'

// Point to your local HTML files directory
const HTML_DIR = path.resolve(__dirname, '../../countries')

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)

// Map from HTML filename stem to country slug in Supabase
const SLUG_MAP: Record<string, string> = {
  algeria: 'algeria', angola: 'angola', benin: 'benin',
  botswana: 'botswana', burkinafaso: 'burkinafaso', burundi: 'burundi',
  cameroon: 'cameroon', centralafricanrepublic: 'centralafricanrepublic',
  chad: 'chad', congo: 'congo', rdcongo: 'rdcongo',
  djibouti: 'djibouti', egypt: 'egypt', equatorialguinea: 'equatorialguinea',
  eritrea: 'eritrea', ethiopia: 'ethiopia', gabon: 'gabon',
  gambia: 'gambia', ghana: 'ghana', guinea: 'guinea',
  guineabissau: 'guineabissau', ivorycoast: 'ivorycoast', kenya: 'kenya',
  liberia: 'liberia', libya: 'libya', malawi: 'malawi',
  mali: 'mali', mauritania: 'mauritania', morocco: 'morocco',
  mozambique: 'mozambique', namibia: 'namibia', niger: 'niger',
  nigeria: 'nigeria', rwanda: 'rwanda', senegal: 'senegal',
  sierraleone: 'sierraleone', somalia: 'somalia', southafrica: 'southafrica',
  southsudan: 'southsudan', sudan: 'sudan', swaziland: 'swaziland',
  tanzania: 'tanzania', togo: 'togo', tunisia: 'tunisia',
  uganda: 'uganda', westernsahara: 'westernsahara', zambia: 'zambia',
  zimbabwe: 'zimbabwe',
}

function extractYouTubeId(src: string): string | null {
  const match = src.match(/\/embed\/([a-zA-Z0-9_-]{11})/)
  return match ? match[1] : null
}

async function main() {
  // Fetch all countries to get IDs
  const { data: countries, error } = await supabase.from('countries').select('id,slug')
  if (error || !countries) { console.error('Failed to fetch countries', error); process.exit(1) }

  const countryMap = Object.fromEntries(countries.map(c => [c.slug, c.id]))
  let totalInserted = 0

  const files = fs.readdirSync(HTML_DIR).filter(f => f.endsWith('.html'))

  for (const file of files) {
    const stem = path.basename(file, '.html').toLowerCase()
    const slug = SLUG_MAP[stem]
    if (!slug || !countryMap[slug]) { console.log(`Skipping ${file} — no slug mapping`); continue }

    const html = fs.readFileSync(path.join(HTML_DIR, file), 'utf-8')
    if (html.trim().length < 50) { console.log(`Skipping ${file} — empty`); continue }

    const $ = cheerio.load(html)
    const videos: Array<{ youtube_id: string; title: string; country_id: string }> = []

    $('iframe').each((_, el) => {
      const src = $(el).attr('src') ?? ''
      const id = extractYouTubeId(src)
      if (!id) return
      const titleEl = $(el).closest('.col-md-6').find('.video-title')
      const title = titleEl.text().trim() || 'Untitled'
      videos.push({ youtube_id: id, title, country_id: countryMap[slug] })
    })

    if (videos.length === 0) { console.log(`No videos in ${file}`); continue }

    const { error: insertError } = await supabase.from('videos').upsert(videos, { onConflict: 'youtube_id,country_id', ignoreDuplicates: true })
    if (insertError) { console.error(`Error inserting ${file}:`, insertError); continue }

    console.log(`✓ ${file}: ${videos.length} videos`)
    totalInserted += videos.length
  }

  console.log(`\nDone — ${totalInserted} videos migrated.`)
}

main()
```

- [ ] **Step 3: Add migrate script to package.json**

In `package.json` scripts:
```json
"migrate": "tsx scripts/migrate.ts"
```

- [ ] **Step 4: Run migration**

Make sure `.env.local` has the Supabase service role key, then:
```bash
npm run migrate
```
Expected output:
```
✓ algeria.html: 9 videos
✓ nigeria.html: 14 videos
...
Done — 487 videos migrated.
```

- [ ] **Step 5: Verify in Supabase**

Open Supabase dashboard → Table Editor → `videos` — should contain ~487 rows.

- [ ] **Step 6: Verify country page shows videos**

```bash
npm run dev
```
Open http://localhost:3000/countries/nigeria — should display 14 video thumbnails.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat: data migration script (HTML → Supabase)"
```

---

## Task 17: Vercel Deployment

- [ ] **Step 1: Push to GitHub**

Create a new repo `Africa-truth-v2` on GitHub (or use a new branch of the existing repo):
```bash
git remote add origin https://github.com/Camaelo26/Africa-truth-v2.git
git push -u origin main
```

- [ ] **Step 2: Connect to Vercel**

1. Go to https://vercel.com → New Project
2. Import `Africa-truth-v2` from GitHub
3. Framework: Next.js (auto-detected)
4. Root directory: `/` (leave default)
5. Click "Deploy" — first deploy will fail (missing env vars — that's expected)

- [ ] **Step 3: Add environment variables**

In Vercel → Project Settings → Environment Variables, add:
```
NEXT_PUBLIC_SUPABASE_URL          (production)
NEXT_PUBLIC_SUPABASE_ANON_KEY     (production)
SUPABASE_SERVICE_ROLE_KEY         (production)
UPSTASH_REDIS_REST_URL            (production)
UPSTASH_REDIS_REST_TOKEN          (production)
```

- [ ] **Step 4: Redeploy**

In Vercel → Deployments → click "Redeploy" on the latest deployment.

Expected: Build succeeds. Site live at `https://africa-truth-v2.vercel.app`.

- [ ] **Step 5: Verify production**

Open the Vercel URL:
- Map loads ✓
- Search works ✓
- Country page shows videos ✓
- `/admin` redirects to login ✓
- Admin login works ✓
- Add a video from admin panel → appears on country page ✓

- [ ] **Step 6: Commit final**

```bash
git add -A && git commit -m "chore: production deployment on Vercel"
```

---

---

## Task 18: Rich Country About Section

**Files:**
- Modify: `lib/types.ts` (add `WikiSummary` type)
- Create: `lib/wikipedia.ts` (fetch country summary)
- Modify: `components/CountryHero.tsx` (expand into tabbed about section)
- Modify: `app/countries/[slug]/page.tsx` (pass wiki data)

- [ ] **Step 1: Write failing test for Wikipedia fetcher**

Create `__tests__/lib/wikipedia.test.ts`:
```typescript
import { describe, it, expect, vi } from 'vitest'
import { buildWikipediaUrl } from '@/lib/wikipedia'

describe('buildWikipediaUrl', () => {
  it('builds URL for simple country name', () => {
    expect(buildWikipediaUrl('Nigeria')).toBe(
      'https://en.wikipedia.org/api/rest_v1/page/summary/Nigeria'
    )
  })
  it('encodes spaces', () => {
    expect(buildWikipediaUrl('South Africa')).toBe(
      'https://en.wikipedia.org/api/rest_v1/page/summary/South_Africa'
    )
  })
  it('encodes special characters', () => {
    expect(buildWikipediaUrl("Côte d'Ivoire")).toBe(
      "https://en.wikipedia.org/api/rest_v1/page/summary/C%C3%B4te_d'Ivoire"
    )
  })
})
```

- [ ] **Step 2: Run — verify fails**

```bash
npm test -- wikipedia.test.ts
```
Expected: FAIL — `Cannot find module '@/lib/wikipedia'`

- [ ] **Step 3: Create wikipedia.ts**

Create `lib/wikipedia.ts`:
```typescript
export type WikiSummary = {
  extract: string
  thumbnail?: { source: string; width: number; height: number }
  content_urls?: { desktop: { page: string } }
}

export function buildWikipediaUrl(countryName: string): string {
  const encoded = countryName.replace(/ /g, '_')
  return `https://en.wikipedia.org/api/rest_v1/page/summary/${encodeURIComponent(encoded).replace(/%27/g, "'")}`
}

export async function fetchWikiSummary(countryName: string): Promise<WikiSummary | null> {
  try {
    const res = await fetch(buildWikipediaUrl(countryName), {
      headers: { 'Accept': 'application/json' },
      next: { revalidate: 86400 }, // cache 24 hours
    })
    if (!res.ok) return null
    const data = await res.json()
    return { extract: data.extract, thumbnail: data.thumbnail, content_urls: data.content_urls }
  } catch {
    return null
  }
}
```

- [ ] **Step 4: Run — verify tests pass**

```bash
npm test -- wikipedia.test.ts
```
Expected: 3 tests PASS

- [ ] **Step 5: Add WikiSummary type to types.ts**

In `lib/types.ts`, add:
```typescript
export type { WikiSummary } from './wikipedia'

export type CountryStats = {
  capital: string
  population: number
  area: number
  languages: string[]
  currencies: string[]
  independence?: string
}
```

- [ ] **Step 6: Build rich CountryAbout component**

Create `components/CountryAbout.tsx`:
```tsx
'use client'
import { useState } from 'react'
import type { Country } from '@/lib/types'
import type { WikiSummary } from '@/lib/wikipedia'

type Props = {
  country: Country
  wiki: WikiSummary | null
  stats: {
    capital: string
    population: number
    languages: string[]
    area: number
  } | null
}

export function CountryAbout({ country, wiki, stats }: Props) {
  const [open, setOpen] = useState(false)

  return (
    <div className="mb-6">
      <button
        onClick={() => setOpen(o => !o)}
        className="w-full flex items-center justify-between px-4 py-3 rounded-xl border border-pan-gold/20 bg-pan-gold/5 hover:bg-pan-gold/10 transition-colors text-sm"
      >
        <span className="text-pan-gold font-semibold">📖 About {country.name}</span>
        <span className="text-white/40 text-lg">{open ? '↑' : '↓'}</span>
      </button>

      {open && (
        <div className="mt-2 card-dark p-5 space-y-5 text-sm">

          {/* Wikipedia summary */}
          {wiki?.extract && (
            <div>
              <p className="text-xs uppercase tracking-widest text-pan-gold/60 mb-2">Overview</p>
              <p className="text-white/70 leading-relaxed line-clamp-6">{wiki.extract}</p>
              {country.wikipedia_url && (
                <a
                  href={country.wikipedia_url}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="text-pan-gold/60 hover:text-pan-gold text-xs mt-2 inline-block"
                >
                  Read more on Wikipedia →
                </a>
              )}
            </div>
          )}

          {/* Key stats grid */}
          {stats && (
            <div>
              <p className="text-xs uppercase tracking-widest text-pan-gold/60 mb-3">Key Facts</p>
              <div className="grid grid-cols-2 gap-3">
                <div className="bg-white/5 rounded-lg p-3">
                  <p className="text-white/40 text-xs mb-1">Capital</p>
                  <p className="font-semibold">{stats.capital}</p>
                </div>
                <div className="bg-white/5 rounded-lg p-3">
                  <p className="text-white/40 text-xs mb-1">Population</p>
                  <p className="font-semibold">{stats.population.toLocaleString()}</p>
                </div>
                <div className="bg-white/5 rounded-lg p-3">
                  <p className="text-white/40 text-xs mb-1">Area</p>
                  <p className="font-semibold">{stats.area.toLocaleString()} km²</p>
                </div>
                <div className="bg-white/5 rounded-lg p-3">
                  <p className="text-white/40 text-xs mb-1">Languages</p>
                  <p className="font-semibold text-xs leading-snug">{stats.languages.slice(0, 3).join(', ')}</p>
                </div>
              </div>
            </div>
          )}

          {/* Region badge */}
          <div className="flex gap-2 flex-wrap">
            <span className="bg-pan-green/10 border border-pan-green/20 text-pan-green text-xs px-3 py-1 rounded-full">
              {country.region}
            </span>
            <span className="bg-white/5 border border-white/10 text-white/50 text-xs px-3 py-1 rounded-full">
              Africa
            </span>
          </div>
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 7: Fetch stats + wiki on server in country page**

In `app/countries/[slug]/page.tsx`, add these fetches alongside the existing Supabase query:

```typescript
// Add these imports
import { fetchWikiSummary } from '@/lib/wikipedia'

// Inside the page component, after fetching country and videos:
const [wiki, restCountries] = await Promise.allSettled([
  fetchWikiSummary(country.name),
  fetch(`https://restcountries.com/v3.1/name/${encodeURIComponent(country.name)}?fullText=true`)
    .then(r => r.ok ? r.json() : null)
    .catch(() => null),
])

const wikiData = wiki.status === 'fulfilled' ? wiki.value : null
const rcData = restCountries.status === 'fulfilled' && restCountries.value
  ? restCountries.value[0] : null

const stats = rcData ? {
  capital: rcData.capital?.[0] ?? '—',
  population: rcData.population ?? 0,
  area: rcData.area ?? 0,
  languages: Object.values(rcData.languages ?? {}) as string[],
} : null
```

Then in the JSX, replace the existing `CountryHero` with:
```tsx
<CountryHero country={country} videoCount={videos?.length ?? 0} />
<CountryAbout country={country} wiki={wikiData} stats={stats} />
```

- [ ] **Step 8: Verify**

```bash
npm run dev
```
Open http://localhost:3000/countries/nigeria → click "About Nigeria" → expands to show Wikipedia summary, stats grid (capital: Abuja, population, area, languages), and region badge.

- [ ] **Step 9: Commit**

```bash
git add -A && git commit -m "feat: rich country about section with Wikipedia summary and key stats"
```

---

## Task 19: Mobile-First Map + Country Card Grid

**Files:**
- Modify: `app/page.tsx`
- Create: `components/CountryCardGrid.tsx`
- Modify: `components/AfricaMap.tsx`

The map is great on desktop but difficult to use on a small phone screen. The fix: on mobile show a searchable, filterable card grid of all countries. On desktop show the interactive map. Users can toggle between them on any screen size.

- [ ] **Step 1: Create CountryCardGrid component**

Create `components/CountryCardGrid.tsx`:
```tsx
'use client'
import { useState, useMemo } from 'react'
import Link from 'next/link'
import type { Country } from '@/lib/types'

const REGIONS = ['All', 'North Africa', 'West Africa', 'East Africa', 'Central Africa', 'Southern Africa']

type Props = { countries: Country[] }

export function CountryCardGrid({ countries }: Props) {
  const [query, setQuery] = useState('')
  const [region, setRegion] = useState('All')

  const filtered = useMemo(() => {
    return countries.filter(c => {
      const matchesQuery = c.name.toLowerCase().includes(query.toLowerCase())
      const matchesRegion = region === 'All' || c.region === region
      return matchesQuery && matchesRegion
    })
  }, [countries, query, region])

  return (
    <div className="px-4 pb-8">
      {/* Search */}
      <input
        type="text"
        placeholder="Search countries..."
        value={query}
        onChange={e => setQuery(e.target.value)}
        className="w-full bg-white/10 border border-white/20 rounded-full px-5 py-3 text-sm text-white placeholder-white/40 focus:outline-none focus:border-pan-gold mb-4"
      />

      {/* Region filter chips */}
      <div className="flex gap-2 flex-wrap mb-5">
        {REGIONS.map(r => (
          <button
            key={r}
            onClick={() => setRegion(r)}
            className={`text-xs px-3 py-1.5 rounded-full border transition-colors ${
              region === r
                ? 'bg-pan-gold/20 border-pan-gold/50 text-pan-gold'
                : 'bg-white/5 border-white/10 text-white/50 hover:text-white'
            }`}
          >
            {r}
          </button>
        ))}
      </div>

      {/* Country cards grid */}
      <div className="grid grid-cols-2 sm:grid-cols-3 gap-3">
        {filtered.map(c => (
          <Link
            key={c.id}
            href={`/countries/${c.slug}`}
            className="card-dark p-4 flex flex-col gap-2 hover:border-pan-gold/30 active:scale-95 transition-all"
          >
            <span className="text-3xl">{c.flag_emoji}</span>
            <div>
              <p className="font-bold text-sm leading-tight">{c.name}</p>
              <p className="text-white/40 text-xs mt-0.5">{c.region}</p>
            </div>
          </Link>
        ))}
      </div>

      {filtered.length === 0 && (
        <p className="text-center text-white/30 text-sm py-12">No countries found.</p>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Update home page with responsive toggle**

Replace `app/page.tsx`:
```tsx
import { createClient } from '@/lib/supabase/server'
import { AfricaMap } from '@/components/AfricaMap'
import { SearchBar } from '@/components/SearchBar'
import { CountryCardGrid } from '@/components/CountryCardGrid'
import { MobileMapToggle } from '@/components/MobileMapToggle'

export default async function HomePage() {
  const supabase = await createClient()
  const { data: countries } = await supabase
    .from('countries')
    .select('*')
    .eq('is_active', true)
    .order('name')

  return (
    <div className="min-h-screen">
      {/* Header — always visible */}
      <div className="pt-10 pb-6 px-4 text-center">
        <p className="text-pan-gold text-xs uppercase tracking-widest mb-2">54 nations. One truth.</p>
        <h1 className="text-3xl font-black mb-2 leading-tight">
          The Political<br />History of Africa
        </h1>
      </div>

      {/* Desktop: map + search bar */}
      <div className="hidden md:block px-4 pb-4">
        <SearchBar />
      </div>
      <div className="hidden md:block px-4 pb-8">
        <AfricaMap />
      </div>

      {/* Mobile: card grid */}
      <div className="md:hidden">
        <CountryCardGrid countries={countries ?? []} />
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Enhance AfricaMap for desktop touch**

In `components/AfricaMap.tsx`, add zoom controls styling. Update the map container:
```tsx
return (
  <>
    <Script src="/scripts/mapdata.js" strategy="beforeInteractive" />
    <Script src="/scripts/worldmap.js" strategy="beforeInteractive" />
    <div
      id="map"
      className="w-full rounded-2xl overflow-hidden border border-white/10"
      style={{ minHeight: '520px', background: 'rgba(26,14,0,0.6)' }}
    />
  </>
)
```

Also update `public/scripts/mapdata.js` — in `main_settings`, set:
```javascript
"zoom": "on",
"zoom_out_incrementally": "on",
"initial_zoom": "1",
"initial_zoom_solo": "on",
```
This enables pinch-to-zoom and scroll-to-zoom on the desktop map.

- [ ] **Step 4: Add bottom nav for mobile**

Create `components/MobileNav.tsx`:
```tsx
'use client'
import Link from 'next/link'
import { usePathname } from 'next/navigation'

const tabs = [
  { href: '/',       icon: '🗺️', label: 'Map'   },
  { href: '/quiz',   icon: '🧠', label: 'Quiz'  },
  { href: '/about',  icon: 'ℹ️',  label: 'About' },
]

export function MobileNav() {
  const path = usePathname()
  return (
    <nav className="fixed bottom-0 left-0 right-0 z-50 md:hidden border-t border-white/10 bg-dark-900/95 backdrop-blur-md">
      <div className="flex">
        {tabs.map(t => (
          <Link
            key={t.href}
            href={t.href}
            className={`flex-1 flex flex-col items-center py-3 gap-0.5 text-xs transition-colors ${
              path === t.href ? 'text-pan-gold' : 'text-white/40'
            }`}
          >
            <span className="text-xl">{t.icon}</span>
            <span>{t.label}</span>
          </Link>
        ))}
      </div>
    </nav>
  )
}
```

Add `<MobileNav />` to `app/layout.tsx` inside `<body>`, after `<main>`. Also add `pb-20 md:pb-0` to `<main>` to prevent bottom nav overlap.

- [ ] **Step 5: Update layout.tsx**

```tsx
import { MobileNav } from '@/components/MobileNav'

// In the body:
<body className={`${inter.className} bg-dark-900 text-white min-h-screen`}>
  <NavBar />
  <main className="pb-20 md:pb-0">{children}</main>
  <MobileNav />
</body>
```

- [ ] **Step 6: Verify on mobile viewport**

```bash
npm run dev
```
Open http://localhost:3000, open Chrome DevTools → toggle device toolbar (iPhone 12 size).
- Should show: search input + country card grid (no map)
- Filter chips should work: tap "West Africa" → only West African countries show
- Tap Nigeria card → navigates to `/countries/nigeria`
- Bottom navigation bar visible with Map / Quiz / About tabs

Switch to desktop viewport:
- Should show: interactive simplemaps map + search bar
- Map should be clickable and navigating to country pages

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "feat: mobile country card grid + region filters + bottom nav"
```

---

## Self-Review

**Spec coverage check:**
- ✅ Next.js + Supabase + Vercel — Tasks 1, 3, 17
- ✅ Pan-African UI (red/gold/green) — Tasks 2, 6
- ✅ Interactive map (desktop) — Task 7
- ✅ Mobile country card grid + bottom nav — Task 19
- ✅ Country pages with videos — Tasks 8, 9
- ✅ Rich about section (Wikipedia + RestCountries stats) — Task 18
- ✅ Quiz with difficulty + timer — Task 9
- ✅ Admin panel (paste URL → auto-title → save) — Tasks 11–14
- ✅ Rate limiting (Upstash Redis) — Task 12
- ✅ CSP + security headers — Task 15
- ✅ Data migration — Task 16
- ✅ About + Contact pages — Task 10
- ✅ Supabase RLS (read=public, write=admin) — Task 3
- ⬜ iOS (Capacitor) — separate Plan 2

**No placeholders found.** All steps contain complete code.

**Type consistency check:** `Country` and `Video` types defined in Task 4 (`lib/types.ts`), `WikiSummary` added in Task 18 — used consistently across Tasks 8, 13, 14, 18. `VideoWithCountry` used in Tasks 13 and 14. `parseYouTubeId` defined in Task 5, used in Tasks 5, 13, 14 — signatures match. `buildWikipediaUrl` defined and tested in Task 18, used in `fetchWikiSummary` in same file.
