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

# Hermes
cp -r skills/mayar-v2 ~/.hermes/skills/mayar
```

Reload/restart agent setelah copy.

## Pemakaian

### Build (integrasi ke app)

Prompt umum, agent menjalankan playbook lengkap (recon → interview → implement → secure → verify → handoff):

```
integrasi payment Mayar di web ini
```

Variasi per model jualan (jawaban INTERVIEW menentukan endpoint yang dipakai):

| Contoh prompt | Model | Endpoint utama |
|---|---|---|
| `pasang pembayaran buat jual ebook saya` | One-off payment | payment link |
| `buatkan halaman checkout produk digital, habis bayar user dapat link download` | One-off + fulfillment | payment link + webhook provisioning |
| `integrasi invoice Mayar, saya freelancer mau tagih klien per project` | Invoice itemized | invoice create + `extraData` |
| `buat sistem langganan bulanan untuk konten premium saya` | Membership/subscription | membership register + invoice per term |
| `user beli credit, tiap panggil fitur AI credit-nya kepotong` | Credit usage-based | credit add/spend/balance |
| `jual lisensi software, user aktivasi pakai kode lisensi` | SaaS/software license | saas activate/verify |
| `buat QRIS on-demand di kasir untuk nominal berapapun` | QRIS dynamic | qrcode create |

Variasi per situasi:

| Contoh prompt | Yang terjadi |
|---|---|
| `integrasi payment Mayar, jawab default semua` | Agent jalan pakai semua default INTERVIEW (sandbox, payment link, dst) |
| `web saya TanStack Start, pasang payment Mayar` | Agent ikut `recipes/tanstack-start.md` |
| `project saya React Vite doang, bisa terima pembayaran?` | Agent jelaskan butuh 1 function kecil, ikut `recipes/vite-react.md` |
| `webhook pembayaran saya kok double terus, cek` | Agent audit handler: dedupe + verify-by-fetch (Step SECURE) |
| `siapkan go-live, sekarang masih sandbox` | Agent kerjakan checklist HANDOFF: ganti key production, register webhook production |

### Ops (admin terminal)

Agent menjalankan CLI langsung. Contoh:

| Contoh prompt | Command yang jalan |
|---|---|
| `saldo mayar saya berapa` | `mayar balance` |
| `cek 10 invoice terakhir` | `mayar invoice list --limit 10` |
| `transaksi hari ini gimana` | `mayar tx daily` |
| `ada berapa transaksi belum dibayar` | `mayar tx unpaid` |
| `buatkan produk membership tiers-nya 3` | `mayar product create --type membership` |
| `register webhook https://app.com/hooks/mayar` | `mayar webhook register <url>` |
| `webhook terakhir gagal mana, retry` | `mayar webhook history` + `mayar webhook retry <id>` |
| `cari customer email budi@gmail.com` | `mayar customer search budi@gmail.com` |
| `kirim magic link portal ke customer itu` | `mayar customer magic-link <email>` |
| `buat QRIS 50 ribu` | `mayar qrcode 50000` |
| `verifikasi kode lisensi ABC123 untuk produk X` | `mayar saas verify ABC123 <productId>` |
| `pindah ke sandbox dulu` | semua command jalan dengan `--sandbox` |

## Prasyarat

- Node.js 18+ (CLI jalan via `npx -y mayar@latest`)
- API key Mayar: [web.mayar.id](https://web.mayar.id) → Integration → API Key (sandbox: [web.mayar.club](https://web.mayar.club))

## Batasan diketahui (menunggu API)

- Webhook belum punya signature HMAC → skill memakai pola verify-by-fetch, siap dimigrasi saat Mayar merilis signature
- Belum ada SDK resmi → recipe memakai `fetch` native dengan helper kecil
- `version: 2.0.0-draft` di frontmatter menandai ini bukan rilis resmi Mayar

## License

MIT
