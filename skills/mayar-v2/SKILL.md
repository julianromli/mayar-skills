---
name: mayar-v2
display_name: Mayar
version: "1.1.0"
description: >
  Mayar payments & billing API (Indonesia: QRIS, VA, e-wallet, kartu).
  BUILD branch: pakai saat user minta integrasi pembayaran ditulis ke dalam app mereka
  (payment link, invoice, membership/subscription, credit wallet, software license,
  webhook, checkout flow) — all-in-one: wiring ke seluruh flow monetisasi user, bukan
  hanya kode Mayar-nya saja.
  OPS branch: pakai saat user minta aksi admin/terminal di akun Mayar mereka
  (balance, list invoices/transactions/customers, buat produk, register webhook).
  Triggers: mayar, payment gateway indonesia, QRIS, billing, langganan, subscription,
  invoice, payment link.
env:
  MAYAR_API_KEY:
    description: Mayar API key. web.mayar.id → Integration → API Key.
    required: false
    secret: true
---

# Mayar

Dua mode. Putuskan dulu sebelum bertindak:

- **build** = menulis kode integrasi ke dalam app user — termasuk wiring ke flow monetisasi, CTA, dan halaman terkait. Bukan hanya kode Mayar-nya. Ikuti playbook 7 step di bawah, berurutan.
- **ops** = menjalankan CLI di terminal untuk admin/testing. Lihat `commands.md`, atau sumber live-nya: `npx -y mayar@latest docs <topic>` dan `npx -y mayar@latest <command> --help`.

CLI dipakai via `npx -y mayar@latest ...` (selalu latest, tanpa install). Di kode app: HTTP native (`fetch`/`axios`). CLI milik terminal.

---

## Setup (berlaku untuk build & ops)

1. Cek auth: `npx -y mayar@latest whoami --json`. `"valid": true` = lanjut.
2. Kalau invalid, tanya environment dulu, baru minta API key ke URL yang sesuai:
   - **Sandbox** (recommended untuk awal): ambil key di [web.mayar.club](https://web.mayar.club) → Integration → API Key
   - **Production**: ambil key di [web.mayar.id](https://web.mayar.id) → Integration → API Key

   Set key: `npx -y mayar@latest api-key <key>` atau set env `MAYAR_API_KEY`.

3. Pilih environment:
   - **build**: default sandbox (`api.mayar.club`, flag `--sandbox`). Pindah production tinggal ganti flag/env, bukan ganti kode.
   - **ops**: tanya user kalau tidak jelas dari konteks; default production.
   - Override manual: flag `--sandbox`/`--production`, atau `MAYAR_API_URL`.

Done when: `whoami` valid dan environment terpilih eksplisit.

---

## Playbook BUILD

### Step 0: RECON

Scan proyek sebelum bertanya apa pun: framework & bahasa (package.json, composer.json, requirements.txt), package manager, file `.env*` yang ada (jangan baca isi secret, cukup catat nama var), direktori routes/api, halaman/komponen yang sudah ada (pricing, product, dashboard, landing), dan kode pembayaran lama kalau ada.

Done when: kamu bisa menyebutkan stack, lokasi route handler, lokasi file env, dan halaman-halaman yang relevan dengan monetisasi — tanpa bertanya ke user.

### Step 1: INTERVIEW

Tanya **satu pertanyaan per pesan**. Tunggu jawaban user sebelum lanjut ke pertanyaan berikutnya. Jika fakta bisa ditemukan di codebase lewat RECON, gunakan itu — jangan tanya ulang. Sesuaikan pertanyaan lanjutan berdasarkan jawaban sebelumnya.

Setiap pertanyaan wajib menggunakan format pilihan ganda dengan jawaban recommended yang paling tepat (bukan paling mudah):

> **[Pertanyaan singkat]**
>
> A) Opsi — penjelasan singkat
> B) Opsi — penjelasan singkat
> C) Opsi — penjelasan singkat
>
> Recommended: **[opsi]** — [alasan singkat].

Urutan pertanyaan (skip yang sudah diketahui dari RECON):

1. **Environment** — sandbox dulu, atau langsung production?
2. **Model jualan** — apa yang dijual, dan bagaimana cara jualnya? (one-off, invoice, langganan, credit, lisensi)
3. **CTA "Beli"** — tombol beli atau checkout mau ada di mana di app? (halaman yang sudah ada, halaman baru, modal, dll.)
4. **Pricing page** — app perlu halaman pricing yang menampilkan plan/harga, atau sudah ada, atau tidak perlu?
5. **Post-payment UX** — setelah bayar sukses, apa yang terjadi di UI? (redirect ke halaman tertentu, modal ditutup, halaman diupdate, dll.)
6. **Fulfillment** — user mendapatkan akses ke apa setelah bayar? Ini menentukan kode provisioning yang ditulis.
7. **State di app** — bagaimana app mengetahui bahwa user sudah bayar / aktif? (field di DB, session, JWT claim, dll.)
8. **API key** — cek via `whoami`. Tanya hanya kalau invalid.

Done when: semua jawaban terdokumentasi di chat. User menjawab "terserah/default" pun sah — catat default yang dipakai.

**Jangan lakukan perubahan apa pun setelah INTERVIEW selesai. Lanjut ke PLAN.**

### Step 2: PLAN

Setelah semua jawaban dari INTERVIEW terkumpul, buat rencana implementasi yang konkret:

- File baru yang akan dibuat, beserta tujuannya
- File existing yang akan diubah, dan bagian mana yang berubah
- Env var baru yang diperlukan
- Urutan implementasi (dari mana mulai, dependensinya apa)

Rencana harus mencakup keseluruhan flow — bukan hanya endpoint Mayar, tapi juga CTA, pricing page (bila diperlukan), webhook handler, dan wiring ke state/DB app user.

Tawarkan setelah plan selesai:
> Plan ini mau disimpan ke codebase sebagai file `.md` (misal `docs/payment-plan.md`), atau cukup di chat saja?

Tutup dengan:
> Ada yang kurang jelas atau perlu direvisi dari plan ini sebelum gue mulai?

**Jangan mulai implementasi sampai user memberikan konfirmasi eksplisit bahwa plan sudah oke.**

Done when: user menyatakan plan sudah dipahami dan memberikan lampu hijau.

### Step 3: IMPLEMENT

- Sebelum menulis request apa pun, baca schema resminya: `npx -y mayar@latest docs <topic> --json` (contoh: `docs create-payment-link`, `docs create-invoice`). Jangan menebak nama field.
- Semua stack berbagi satu pola: `recipes/_pattern.md`. Kelayakan stack ditentukan oleh ada/tidaknya server runtime. SPA murni tanpa backend: wajib 1 function kecil, lihat `recipes/vite-react.md`.
- Stack punya recipe (`recipes/nextjs.md`, `recipes/tanstack-start.md`, dst)? Ikuti wiring-nya. Belum ada recipe? Pakai `_pattern.md` langsung.
- Implementasi harus mencakup seluruh yang ada di PLAN: endpoint Mayar, CTA button, pricing page (bila diperlukan), webhook handler, dan wiring ke state/DB app user. Jangan hanya kode Mayar-nya saja.
- API key masuk `.env` (bukan source code), hanya dipakai di kode server, dan `.gitignore` sudah mencakup file env. Cek `.gitignore` secara eksplisit.

Done when: semua file dari PLAN sudah diimplementasi, kode compile/runtime, key di env, gitignore terverifikasi.

### Step 4: SECURE

Aturan inti: **payload webhook = notifikasi, bukan bukti. Bukti ada di API.**

- Handler webhook wajib re-fetch status transaksi/invoice via GET API (pakai Bearer key) sebelum provisioning. Klaim `paid` dari payload saja tidak cukup.
- Provisioning harus idempotent: catat ID transaksi yang sudah diproses (tabel DB / store), kiriman webhook duplikat menghasilkan efek yang sama tanpa double-fulfill.
- Tambahkan komentar di handler: `// TODO: ganti verify-by-fetch dengan signature verification saat Mayar merilis HMAC webhook`.

Done when: handler melakukan re-fetch, dan kiriman duplikat tidak memicu provisioning ganda (tunjukkan dengan test atau reasoning kode).

### Step 5: VERIFY

Buktikan end-to-end di sandbox:

1. Buat payment link / invoice via API sandbox.
2. Selesaikan pembayaran test. Kalau perlu aksi manual user (klik bayar di hosted page), minta user melakukannya. Kalau perlu dev server lokal, minta izin user dulu sebelum menjalankannya.
3. Konfirmasi hasil: `npx -y mayar@latest --sandbox webhook history` dan/atau re-fetch status sampai `paid`.
4. Minimal bila simulasi bayar sandbox mentok: pembuatan sukses + webhook terkirim tercatat di history.

Done when: transaksi sandbox mencapai `paid` dan jalur provisioning app terbukti tereksekusi (atau terbukti via dry-run).

### Step 6: HANDOFF

Laporkan ke user: file apa yang dibuat/diubah, env var apa yang harus diisi, dan checklist go-live:

- [ ] Ganti API key production + environment production
- [ ] Register webhook URL production (`mayar webhook register <url>`)
- [ ] Test 1 transaksi kecil beneran
- [ ] Pantau `mayar webhook history` untuk delivery

Done when: user menyatakan checklist diterima/dipahami.

---

## Branch OPS

Jalankan command CLI langsung di terminal. Katalog lengkap: `commands.md`.
Sumber authoritative: `npx -y mayar@latest docs <query>` (cari docs API) dan `npx -y mayar@latest <command> --help`.

Untuk parsing programatik, tambahkan `--json`.

---

## Reference (inline, satu-satunya sumber)

**Base URL** — production `https://api.mayar.id/hl/v2`, sandbox `https://api.mayar.club/hl/v2`.

**Envelope V2** — `{ statusCode, messages, data }`. Sebagian write endpoint memakai `message` (singular). Parse defensif: `body.messages ?? body.message`. Tanggal: beberapa endpoint menerima ISO 8601 tapi mengembalikan epoch ms (`expiredAt`); normalisasi di sisi client.

**Error** — `{ statusCode, messages }` dengan detail minim (`Validation Error` tanpa field). Saat kena 400: cocokkan ulang body request dengan schema dari `mayar docs <topic> --json`, jangan menebak. `429` pada create = duplikat terdeteksi, tunggu ±1 menit lalu retry dengan payload identik.

**Payment link name unik** — Mayar tolak create dengan error `"already exist"` kalau nama sama pernah dipakai. Selalu append timestamp atau UUID: `name: \`${productName} #${Date.now()}\``. Error ini muncul sebagai HTTP 200 dengan `statusCode: 429` di body.

**Rate limit** — 50 req/menit per API key, header `Retry-After` disediakan. Jarakkan polling status ≥ 5 detik.
