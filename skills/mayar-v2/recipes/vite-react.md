# Recipe: React + Vite (SPA murni)

Vite SPA tidak punya server runtime. Dua hal mustahil di SPA murni: menyimpan `MAYAR_API_KEY` (bundle JS bisa dibaca siapa saja) dan menerima webhook (butuh endpoint publik). Jadi butuh **satu backend kecil**; kabar baiknya cukup 2 route.

Pola & helper: `_pattern.md`. File ini opsi backend + wiring SPA.

## Pilih satu opsi backend (INTERVIEW Step 1)

| Opsi | Kapan dipilih |
|---|---|
| **A. Serverless function** (Cloudflare Workers / Vercel / Netlify) | Default. Gratis, deploy cepat, URL publik langsung ada untuk webhook |
| **B. Mini server** (Hono/Express) | User sudah punya VPS/server sendiri |
| **C. Tanpa kode sama sekali** | User cuma butuh jualan cepat: buat link via CLI (ops), tempel URL-nya di SPA. Bukan integrasi penuh (tanpa provisioning otomatis) |

## Opsi A: Cloudflare Worker (contoh)

Catatan Workers (terverifikasi docs resmi): secret/env dibaca dari **parameter `env`** di handler, bukan `process.env` (kecuali pakai flag `nodejs_compat`). Karena helper `_pattern.md` membaca `process.env`, tambahkan adaptor kecil ini di Worker:

```ts
// mayar-init.ts — adaptor env Workers
let KEY = "";
export function initMayar(env: { MAYAR_API_KEY: string; MAYAR_ENV?: string }) {
  KEY = env.MAYAR_API_KEY;
}
export function apiKey() {
  return KEY;
}
```

Lalu di helper, ganti `process.env.MAYAR_API_KEY` menjadi `apiKey()`. `MAYAR_ENV` tetap dibaca dari `env.MAYAR_ENV` saat init.

`src/index.ts` di project Worker terpisah (satu folder kecil, `wrangler deploy`):

```ts
import { initMayar } from "./mayar-init";
import { createPaymentLink, getTransaction } from "./mayar"; // helper _pattern.md + adaptor di atas

interface Env {
  MAYAR_API_KEY: string;
  MAYAR_ENV?: string;
}

const processed = new Set<string>(); // produksi: KV/D1, bukan memory

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    initMayar(env);
    const url = new URL(request.url);

    if (url.pathname === "/api/checkout" && request.method === "POST") {
      const product = await createPaymentLink({
        name: "Akses Pro 1 Bulan",
        description: "Upgrade ke Pro",
        amount: 150000,
        redirectUrl: "https://<domain-spa>/thanks", // route SPA biasa
      });
      return Response.json({ url: product.link });
    }

    if (url.pathname === "/api/webhooks/mayar" && request.method === "POST") {
      const payload: any = await request.json();
      const tx = payload.data ?? {};
      if ((payload.event ?? payload.type) !== "payment.received" || !tx.id) {
        return Response.json({ ok: true });
      }
      const detail = await getTransaction(tx.id); // bukti dari API
      const paid = ["paid", "success", "settled"].includes(String(detail.status).toLowerCase());
      if (!paid) return Response.json({ ok: true });
      if (processed.has(detail.id)) return Response.json({ ok: true });
      processed.add(detail.id);
      // fulfill(tx.customerEmail, tx.productId)
      return Response.json({ ok: true });
    }

    return new Response("not found", { status: 404 });
  },
};
```

Env di Worker: `npx wrangler secret put MAYAR_API_KEY` + var `MAYAR_ENV` di wrangler config. Sertakan CORS header kalau SPA dan Worker beda domain (`Access-Control-Allow-Origin` untuk origin SPA).

## Wiring SPA (berlaku semua opsi)

```tsx
export function BuyButton() {
  return (
    <button
      onClick={async () => {
        const r = await fetch("https://<worker-atau-server>/api/checkout", { method: "POST" });
        const { url } = await r.json();
        window.location.href = url;
      }}
    >
      Beli Sekarang
    </button>
  );
}
```

`redirectUrl` mengarah ke route SPA (contoh `/thanks` via React Router). Halaman thanks bersifat informatif; status bayar sebenarnya tetap dari webhook/re-fetch, bukan dari URL.

## Test

```bash
npx -y mayar@latest --sandbox webhook register https://<worker>/api/webhooks/mayar
```

## Checklist

- [ ] Tidak ada `MAYAR_API_KEY` di bundle SPA (cek: `grep -r MAYAR dist/` kosong)
- [ ] Transaksi sandbox `paid`, provisioning jalan
- [ ] Webhook duplikat tidak double-fulfill
- [ ] Go-live: key production + webhook URL production
