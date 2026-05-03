# Africa Truth — Project Guide

## What This Is
Educational platform covering the political history of all 54 African nations through curated YouTube videos and an interactive geography quiz. Created by Macarthur Diby.

**Live site:** http://politicalmapafrica.s3-website-us-east-1.amazonaws.com/
**GitHub:** https://github.com/Camaelo26/Africa-truth

---

## Current Stack (v1 — Static HTML)
- Pure HTML/CSS/JS, Bootstrap 4.5.2, Font Awesome 5.15.4
- SimpleMapps worldmap library (trial v4.30)
- RestCountries API for country metadata
- Google Analytics (G-W1LKM277BL)
- Hosted on AWS S3 as a static site

### File Structure (local — flat layout)
```
political platform/
  map.html          ← main landing page with interactive map
  quizz.html        ← geography quiz (difficulty + timer)
  about.html
  contact.html
  [country].html    ← one page per country (48 active, 5 empty)
  mapdata.js        ← country config for main map
  mapdata1.js       ← country config for quiz map
  worldmap.js       ← simplemaps library (main map)
  worldmap1.js      ← simplemaps library (quiz)
  videoplayback.webm← background video for map page
  [country].jpg/webp/avif ← background images per country
```

> **Note:** GitHub repo organizes files into `countries/`, `images/`, `js files/` subdirectories, but S3 deployment is flat. HTML files use flat relative paths.

---

## Target Stack (v2 — Modern App)
See full spec: `docs/superpowers/specs/2026-05-02-africa-truth-redesign.md`

- **Frontend:** Next.js 14 (App Router) + Tailwind CSS
- **Database:** Supabase (countries + videos tables)
- **Hosting:** Vercel
- **iOS:** Capacitor wrapping the Next.js app
- **Security:** Rate limiting (Upstash Redis), Supabase RLS, CSP headers

---

## Development Rules

### Adding Videos (current v1)
Edit the country's HTML file directly and add an `<iframe>` block following the existing pattern.

### Adding Videos (v2 admin panel)
Go to `/admin` → paste YouTube URL → select country → hit Add. No code changes needed.

### Country Page Template
Each country page follows this structure:
1. Google Analytics tag
2. Bootstrap 4.5.2 CSS
3. Dark overlay on background image
4. Navbar with country name + Map link + About modal trigger
5. Video grid (Bootstrap row/col-md-6)
6. About modal with Wikipedia link + RestCountries API data
7. Footer

### Empty Country Pages (need content)
- `djibouti.html` ✓ filled 2026-05-02
- `eritrea.html` ✓ filled 2026-05-02
- `swaziland.html` ✓ filled 2026-05-02 (Eswatini)
- `tanzania.html` ✓ filled 2026-05-02
- `zimbabwe.html` ✓ filled 2026-05-02

---

## Deployment

### Current (S3)
Upload all files flat to the S3 bucket root. The bucket serves `map.html` as default.

### v2 (Vercel)
```bash
vercel deploy
```

### iOS (Capacitor)
```bash
npm run build && npx cap sync ios && npx cap open ios
```

---

## Security Notes
- Never commit `.env.local` or any file containing `SUPABASE_SERVICE_ROLE_KEY`
- The Supabase anon key (`NEXT_PUBLIC_SUPABASE_ANON_KEY`) is safe to expose — RLS makes it read-only
- Admin route protected by Supabase Auth middleware
- Rate limit: 60 req/min per IP on API routes, 10 req/min on `/admin/*`

---

## Key Contacts
- Creator: Macarthur Diby — dibycamael@gmail.com
- GitHub: Camaelo26
