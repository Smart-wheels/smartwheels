# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

SmartWheels is a Dutch-language marketing site for a wheel-restoration business (velgenservice: poedercoaten, CNC afdraaien, reparatie). It's a Next.js 16 App Router site with a Sanity CMS for the projects portfolio and a Resend-backed quote-request (offerte) form. UI copy and Sanity field names are in Dutch — keep new user-facing strings in Dutch.

## Commands

```bash
npm run dev            # Next.js dev server (http://localhost:3000)
npm run build          # production build
npm run start          # serve the production build
npm run lint           # eslint (next/core-web-vitals + next/typescript)
npm test               # jest
npm run test:watch     # jest watch mode
npm run test:coverage  # jest with coverage
npx jest tests/app/components/navigation.test.tsx   # run a single test file
npx jest -t "should render the logo"                # run tests matching a name
npx sanity <command>   # Sanity CLI (deploy, dataset, etc.); needs the NEXT_PUBLIC_SANITY_* env vars
```

The Sanity Studio is embedded at `/studio` (route `app/studio/[[...tool]]/page.tsx`) — no separate Studio process is needed.

## Environment variables

- `RESEND_API_KEY` — required for the offerte API route to send email; the route returns a 500 if missing.
- `NEXT_PUBLIC_SANITY_PROJECT_ID` (defaults to `ckmy7d6l`), `NEXT_PUBLIC_SANITY_DATASET` (defaults to `production`), `NEXT_PUBLIC_SANITY_API_VERSION` (defaults to `2024-11-01`) — read in `sanity/env.ts`, which throws if the resolved value is undefined.

`.env*` files are gitignored.

## Architecture

**Pages (Server Components) + `*-client.tsx` (Client Components).** Each route's `page.tsx` is a Server Component that exports SEO `metadata`, fetches Sanity data, and renders static markup. Interactive parts (forms, animated counters, filters) live in sibling `'use client'` files — `home-client.tsx`, `offerte/offerte-client.tsx`, `projecten/content.tsx`, `diensten/content.tsx`. When adding interactivity, follow this split rather than turning a whole page into a client component.

**Sanity content flow.** There is a single schema type, `project` (`sanity/schemaTypes/project.ts`), with before/after images, a services string array, and a numeric `order`. Pages fetch via GROQ using the plain client in `sanity/lib/client.ts` (`client.fetch(query)`), e.g. `app/projecten/page.tsx` (ordered by `order`) and `app/page.tsx` (recent 3 by `_createdAt`). The projects page uses `export const revalidate = 60` for ISR. Images render through `next/image`; remote `cdn.sanity.io` is allowlisted in `next.config.ts`. For Sanity image transforms use `@sanity/image-url`'s `urlFor(...)` (see `app/page.tsx`). A `sanity/lib/live.ts` (`sanityFetch` / `SanityLive`) exists but pages currently use the plain `client.fetch`.

**Offerte form → email.** `offerte-client.tsx` builds a `FormData` (text fields + up to 5 photo `File`s under the `photos` key) and POSTs to `app/api/send-offerte/route.ts`. The route validates name/email/phone, reads `photos` files into Buffer attachments, and sends two emails via Resend: an admin notification to `smartwheels1@outlook.com` and a customer confirmation, both `from: noreply@smart-wheels.nl`. Email HTML is built as inline-styled template strings in the route.

## Conventions

- **Path alias:** `@/*` maps to the repo root (e.g. `@/components/ui/button`, `@/sanity/lib/client`, `@/lib/utils`).
- **UI components:** shadcn/ui (new-york style) in `components/ui/`, configured via `components.json`. Use the `cn()` helper from `lib/utils.ts` (clsx + tailwind-merge) for conditional classes. Icons from `lucide-react`.
- **Styling:** Tailwind CSS v4 (config-less, via `@tailwindcss/postcss`); global styles and custom keyframe animations (`animate-fade-in-up`, `animate-float`, `animate-shake`, etc.) are defined in `app/globals.css`. The site's palette is dark zinc backgrounds with orange-600 accents.
- **Tests:** Jest + React Testing Library, jsdom environment, config via `next/jest` (`jest.config.js`). Tests live under `tests/` mirroring the source path. Mock `next/link` in component tests (see `tests/app/components/navigation.test.tsx`).
