# JPLazy

Personal Japanese learning web app — JLPT N3 → N2 → N1 — bilingual JP↔VN, curriculum-driven daily lesson, AI tutor sidebar, SRS practice, all-Cloudflare free tier.

## Status

**Pre-implementation.** Plan đã review xong qua 3 vòng:
- `/office-hours` — product framing
- `/plan-eng-review` — architecture, code quality, tests, performance (8 issues + 4 outside voice tensions, all resolved)
- `/plan-design-review` — visual identity, interaction states, mobile/a11y (score 4/10 → 9/10)

22 decisions logged. Sẵn sàng implement.

Đọc design doc đầy đủ: [docs/design.md](docs/design.md).

## Quick reference

- **Stack:** Next.js 15 (App Router) + Cloudflare Workers (OpenNext adapter) + Workers AI (Llama 3.3 70B + Whisper + MeloTTS-JP) + D1 (SQLite-on-edge) + Cache API
- **Cost:** $0/tháng (full Cloudflare free tier, 10K neurons/day)
- **Primary device:** Android. iOS support secondary.
- **Bilingual:** JP↔VN native (không qua tiếng Anh trung gian)
- **Reminder:** PWA Web Push 8h sáng VN qua Cron Trigger
- **Auth:** HTTP Basic Auth (single user, password trong wrangler secret)
- **Framing:** "Bunpro replacement cho VN learners" — content quality bar cao, không chỉ AI draft

## MVP timeline

3 weekends sau Day-0 spikes:

- **Day-0** (~2-3 giờ): 5 spikes verify trước commit (MeloTTS-JP, quiz quality, sắc thái VN, streaming OpenNext, Web Push edge runtime)
- **Weekend 1:** Foundation + curriculum (Tuần 1-3 N2) + daily lesson view bilingual + TTS + neurons tracker
- **Weekend 2:** AI helper sidebar streaming + quiz generator (Zod validated) + SM-2 SRS
- **Weekend 3:** PWA Push + Basic Auth + deploy + tests + `/api/export`

Sau MVP: 1 tuần test thật, rồi mới quyết định bồi feature (full N2 dataset / STT shadowing / listening lab / reading lab / IT business track).

## Development

Setup chi tiết trong design doc. Tóm tắt:

```bash
npm create cloudflare@latest JPLazy -- --framework=next
wrangler d1 create jplazy-db
# bind D1 + Workers AI vào wrangler.toml
wrangler dev    # local dev
wrangler deploy # production
```

## Visual identity

- **Fonts:** Inter (Latin) + Noto Sans JP (Japanese)
- **Palette:** ink #1a1a1a, paper #fafaf7, asagi #3b6f9c, coral #e85a4f, sand #f0e8d8, muted #8a8a85
- **Dark mode:** default theo system preference, override trong Settings

Visual identity yaml đầy đủ trong [docs/design.md § Design Spec](docs/design.md).

## License

Personal project, không monetize, không serve user khác. No license declared.
