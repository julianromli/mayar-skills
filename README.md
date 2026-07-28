# Mayar Skills

Agent skills untuk integrasi [Mayar](https://mayar.id) (payment & billing Indonesia: QRIS, VA, e-wallet, kartu) di AI coding tools (Claude Code, Cursor, Codex, OpenCode).

> Status: **draft v2, belum rilis resmi**. Feedback & PR welcome.

## Isi

```
skills/
└── mayar-v2/
    ├── SKILL.md                 playbook utama: 6 step (RECON → INTERVIEW → IMPLEMENT → SECURE → VERIFY → HANDOFF)
    ├── commands.md              katalog CLI (branch ops)
    └── recipes/
        ├── _pattern.md          pola integrasi framework-agnostic (single source of truth)
        ├── nextjs.md            wiring Next.js App Router
        ├── tanstack-start.md    wiring TanStack Start server routes
        └── vite-react.md        React Vite SPA (via serverless function / mini server)
```

## Bedanya dengan skill resmi (v1)

Skill resmi [`mayarid/mayar-cli`](https://github.com/mayarid/mayar-cli) adalah referensi CLI untuk admin/ops. Draft v2 ini menambahkan **playbook integrasi** untuk vibe coder & AI agent:

- **RECON**: agent scan codebase dulu (stack, env, routes) sebelum bertanya
- **INTERVIEW**: 5 pertanyaan satu batch, semua ada default (env, model jualan, fulfillment, URL publik, API key)
- **IMPLEMENT**: recipe per stack, schema resmi dibaca via `mayar docs`, bukan menebak field
- **SECURE**: webhook wajib verify-by-fetch (payload = notifikasi, bukti = re-fetch API), provisioning idempotent
- **VERIFY**: test end-to-end di sandbox, bukti ditunjukkan ke user
- **HANDOFF**: checklist go-live ke production

Tiap step punya completion criterion yang checkable. Prinsip penulisan mengikuti rubrik [writing-great-skills](https://github.com/anthropics/skills) (predictability, information hierarchy, no duplication).

Snippet TanStack Start & Cloudflare Workers sudah diverifikasi terhadap docs resmi via Context7.

## Install

```bash
git clone https://github.com/julianromli/mayar-skills.git
cd mayar-skills

# Claude Code (backup dulu kalau ada v1)
cp -r skills/mayar-v2 ~/.claude/skills/mayar

# Cursor (global)
cp -r skills/mayar-v2 ~/.cursor/skills/mayar

# Cursor (per project)
cp -r skills/mayar-v2 <project>/.cursor/skills/mayar

# Codex
cp -r skills/mayar-v2 ~/.codex/skills/mayar
```

Reload/restart agent setelah copy.

## Pemakaian

**Build (integrasi ke app):**

```
integrasi payment Mayar di web ini
```

Agent menjalankan playbook: recon codebase → interview 5 pertanyaan → implement ikut recipe stack → secure webhook → verify sandbox → checklist go-live.

**Ops (admin terminal):**

```
cek 10 invoice terakhir di mayar
```

Agent menjalankan CLI langsung (`npx -y mayar@latest invoice list`, dst).

## Prasyarat

- Node.js 18+ (CLI jalan via `npx -y mayar@latest`)
- API key Mayar: [web.mayar.id](https://web.mayar.id) → Integration → API Key (sandbox: [web.mayar.club](https://web.mayar.club))

## Batasan diketahui (menunggu API)

- Webhook belum punya signature HMAC → skill memakai pola verify-by-fetch, siap dimigrasi saat Mayar merilis signature
- Belum ada SDK resmi → recipe memakai `fetch` native dengan helper kecil
- `version: 2.0.0-draft` di frontmatter menandai ini bukan rilis resmi Mayar

## License

MIT
