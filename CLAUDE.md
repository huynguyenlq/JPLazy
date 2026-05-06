# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

**Pre-implementation.** Repo currently contains design doc only (`docs/design.md`). No code, no scaffold, no `package.json` yet.

The design has been through 3 review rounds (office-hours, plan-eng-review, plan-design-review) producing **22 logged decisions**. These decisions are authoritative — do not re-litigate them. If a decision needs to change, update the doc explicitly with rationale.

**Read [docs/design.md](docs/design.md) before any non-trivial work.** Especially the "Eng-Review Decisions Log" and "Design-Review Decisions Log" sections at the bottom — they capture *why* current architecture choices were made.

## Project context

JPLazy = personal Japanese learning web app. JLPT N3 → N2 → N1. Bilingual JP↔VN (no English intermediate). Curriculum-driven daily lessons with AI tutor sidebar, SM-2 SRS, PWA push reminder.

- **User:** Vietnamese IT/software dev, JLPT N3 from 7 years ago, prefers Vietnamese-language responses
- **Single user:** No multi-user model. Auth = HTTP Basic with manual constant-time password compare
- **Primary device: Android.** iOS secondary. Many UI/risk assumptions hinge on this
- **Cost target: $0/month** — Cloudflare free tier strict (10,000 neurons/day, resets 00:00 UTC)
- **Framing:** "Bunpro replacement for VN learners" — content quality bar high, AI draft alone is insufficient

## Stack (locked)

- Next.js 15 App Router on **Cloudflare Workers via OpenNext adapter** (NOT Cloudflare Pages — `next-on-pages` deprecated 2026)
- Workers AI: primary `@cf/meta/llama-3.3-70b-instruct-fp8-fast`, fallback `@cf/meta/llama-3.1-8b-instruct-fast`
- Whisper `@cf/openai/whisper-large-v3-turbo` for STT
- MeloTTS `@cf/myshell-ai/melotts` for TTS — Japanese support unconfirmed, requires Day-0 Spike 1; browser SpeechSynthesis is the documented fallback
- D1 (SQLite-on-edge) for persistence including curriculum
- Cache API (built-in, edge-cached) for TTS audio cache from MVP — NOT R2
- PWA Web Push via Cron Trigger `0 1 * * *` (01:00 UTC = 08:00 VN)

**Do not introduce:**
- `web-push` npm package — Node-only deps, breaks edge runtime. Use `@block65/webcrypto-web-push` or raw VAPID JWT + AES-GCM via Web Crypto API
- `crypto.subtle.timingSafeEqual` — does not exist in Web Crypto API. See `lib/auth.ts` plan for manual constant-time impl
- Vercel hosting, paid LLM APIs, multi-user auth, R2 audio cache (use Cache API instead)
- Bundled curriculum JSON — D1-backed from MVP to avoid 1MB Worker script limit

## Day-0 prerequisites (run before any Weekend 1 work)

5 spikes must verify before committing to MVP scope. See design doc § "Day 0 — Spikes":

1. **Spike 1:** MeloTTS-JP with `lang: "ja"` produces usable Japanese audio
2. **Spike 2:** Llama 3.3 70B generates valid fill-in-the-blank quiz with grammar context injected
3. **Spike 3:** Llama 3.3 70B explains 〜ものの vs 〜ながらも nuance acceptably in Vietnamese
4. **Spike 4:** `ReadableStream` token-by-token flush works through OpenNext + Workers App Router (not buffered)
5. **Spike 5:** Web Push protocol works on edge runtime (verify chosen lib or raw impl)

If any spike fails, the design doc has documented fallbacks. Do not silently work around a failed spike — surface to the user.

## Architecture pointers (post-scaffold)

When implementation begins, follow design doc exactly:

- `lib/ai.ts` — `callAI()` wrapper with timeout (30s AbortController), retry once on 5xx, model fallback on deprecated, friendly error messages. **All AI calls go through this** — no direct `env.AI.run()` in route handlers
- `lib/lessons.ts` — `getTodaysLessons(today)` runs date-based query (`scheduled_date <= today AND lesson_id NOT IN completed`). Caps at 5 to handle catch-up
- `lib/srs.ts` — SM-2 algorithm. Correct: `interval *= ease`, `ease = clamp(ease + 0.1, 1.3, 2.5)`. Incorrect: `interval = 1`, `ease -= 0.2`, `failures++`. Graduate at `interval > 30 days`. Leech at `failures >= 5`
- `lib/auth.ts` — manual `timingSafeEqual()` constant-time loop. Middleware matcher `['/((?!api/health|_next/static|manifest.json|sw.js).*)']`
- `lib/quiz.ts` — Zod discriminated union for `fill_blank | multiple_choice | sentence_rearrange`. Validate AI output before saving to `quiz_pool`. Retry once on malformed; fallback to hand-curated quiz from lesson JSON
- D1 schema includes `daily_neurons(date, used)` for quota tracking. UI status bar shows usage; soft cap at 80%, hard cap at 95% (refuse non-essential AI calls, keep chat helper available)
- Curriculum: source JSON in `curriculum/n2/week-XX.json`, migration script `scripts/import-curriculum.ts` UPSERTs into D1 `lessons` table

## UI / design constraints

- **Bilingual layout:** mobile (<768px) stack JP↑VN↓; desktop (≥768px) side-by-side 50/50. JP always primary visual (1.25rem ink), VN secondary (0.875rem muted)
- **Mobile AI sidebar:** bottom-sheet drag-up (shadcn `Drawer`). Floating action button "Hỏi AI" 56px bottom-right. NOT inline expand
- **Touch targets:** ≥ 44px on mobile (TTS audio buttons, quiz options)
- **Accessibility:** `<span lang="ja">` and `<span lang="vi">` for screen readers, keyboard nav with arrow keys + Enter for quiz, focus ring asagi 2px (not browser default), `V` keyboard shortcut toggles bilingual VN visibility
- **Dark mode:** default via `@media prefers-color-scheme`, override in Settings (Auto/Light/Dark), persist localStorage
- **Welcome back UX (gap ≥ 2 days):** warm framing "Đã mấy hôm không gặp! Có N bài đợi bạn" + 2 CTA "Catch up từng bài" / "Skip, bắt đầu hôm nay". NOT guilt-tripping
- **First-time empty state:** banner "Curriculum N2 12 tuần cần setup trước" + button "Setup" → triggers migration spinner ~5s
- **AI sidebar empty:** placeholder + 3 example chips. Streaming: typing cursor `▍` blink + "AI đang gõ..."

## Visual identity

Locked in `docs/design.md § Visual Identity`. Summary:

- **Fonts:** Inter (Latin) + Noto Sans JP / Yu Gothic (Japanese). 3 weights max: 400, 500, 700
- **Light palette:** ink `#1a1a1a`, paper `#fafaf7`, asagi `#3b6f9c`, coral `#e85a4f`, sand `#f0e8d8`, muted `#8a8a85`, warning `#c8941d`
- **Dark palette:** spec'd separately in design doc
- **Spacing scale:** `4 | 8 | 12 | 16 | 24 | 32 | 48 | 64` px — pick from this set, no arbitrary values
- **Composition:** single-column reading flow max-width 720px on desktop. No card mosaic. Cards earn existence

## Build / dev commands (post-scaffold, not yet valid)

```bash
npm create cloudflare@latest -- --framework=next   # Weekend 1 scaffold (OpenNext + Workers)
wrangler d1 create jplazy-db                        # create D1 database
wrangler d1 execute jplazy-db --file=schema.sql     # apply schema
wrangler dev                                         # local dev with D1 + AI bindings
wrangler deploy                                      # production deploy
vitest                                               # unit tests (lib/lessons, lib/srs, lib/auth, lib/ai, lib/quiz)
playwright test                                      # E2E smoke (daily lesson + quiz→SRS)
```

## Communication

User writes in Vietnamese — respond in Vietnamese. Bilingual content in app is a hard requirement, not optional.

Reviews completed: eng-review and design-review both CLEAR (see `docs/design.md § GSTACK REVIEW REPORT`). Do not re-run them unless the plan changes substantially. Status: ENG+DESIGN-REVIEWED — sẵn sàng implement.
