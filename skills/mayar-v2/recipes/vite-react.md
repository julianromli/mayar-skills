# Recipe: React Vite (SPA + mini server)

Pola, helper `lib/mayar.ts`, dan logika webhook: `_pattern.md`. File ini hanya wiring untuk SPA React Vite.

SPA murni tidak punya server runtime — semua Mayar calls (create payment link, webhook handler) **wajib** melewati minimal 1 server endpoint kecil. Dua opsi:

| Opsi | Kapan dipilih |
|---|---|
| **A — Vite + Express/Hono mini server** | Project sudah ada backend terpisah, atau mau semua dalam satu repo |
| **B — Cloudflare Workers** | Deploy static ke CF Pages, handler di Workers |

---

## Opsi A: Vite + Hono mini server

### Struktur

```
src/          ← React SPA (Vite)
server/
  index.ts    ← Hono server (checkout + webhook)
  mayar.ts    ← helper dari _pattern.md
.env
```

### Env

```bash
MAYAR_API_KEY=paste_key_di_sini
MAYAR_ENV=production
APP_URL=http://localhost:3000
```

### Server — `server/index.ts`

```ts
import { Hono } from "hono";
import { serve } from "@hono/node-server";
import { createPaymentLink, getTransaction } from "./mayar";

const app = new Hono();

// Checkout
app.post("/api/checkout", async (c) => {
  const product = await createPaymentLink({
    name: "Akses Pro",
    description: "Upgrade ke Pro",
    amount: 150000,
    redirectUrl: `${process.env.APP_URL}/thanks`,
  });
  return c.json({ url: product.link });
});

// Webhook
// TODO: ganti verify-by-fetch dengan signature verification saat Mayar merilis HMAC webhook.
// Produksi: ganti Set dengan tabel DB processedTransactions.
const processed = new Set<string>();

interface WebhookPayload {
  event?: string;
  type?: string;
  data?: { id: string };
}

app.post("/api/webhooks/mayar", async (c) => {
  const payload = await c.req.json<WebhookPayload>();
  const tx = payload.data ?? {};
  if ((payload.event ?? payload.type) !== "payment.received" || !tx.id) {
    return c.json({ ok: true });
  }

  const detail = await getTransaction(tx.id); // bukti dari API
  const paid = ["paid", "success", "settled"].includes(String(detail.status).toLowerCase());
  if (!paid) return c.json({ ok: true });

  if (processed.has(detail.id)) return c.json({ ok: true });
  processed.add(detail.id);

  // PROVISIONING: idempotent — beri akses / kirim download / upgrade akun.
  console.log("fulfill", detail.customer.email, detail.paymentLink.id);
  return c.json({ ok: true });
});

serve({ fetch: app.fetch, port: 3001 });
```

### Tombol beli (React component)

```tsx
export function BuyButton() {
  return (
    <button
      onClick={async () => {
        const r = await fetch("/api/checkout", { method: "POST" });
        const { url } = await r.json();
        window.location.href = url;
      }}
    >
      Beli Sekarang
    </button>
  );
}
```

Vite dev server proxy ke Hono dengan tambahan di `vite.config.ts`:

```ts
server: {
  proxy: {
    "/api": "http://localhost:3001",
  },
},
```

---

## Opsi B: Cloudflare Workers

Di Workers, env dibaca dari parameter `env` handler — **bukan** `process.env`. Adaptasi helper:

```ts
// worker.ts
import { createPaymentLink, getTransaction } from "./mayar";

interface Env {
  MAYAR_API_KEY: string;
  MAYAR_ENV: string;
  APP_URL: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/api/checkout" && request.method === "POST") {
      // inject env ke helper sebelum call
      process.env.MAYAR_API_KEY = env.MAYAR_API_KEY;
      process.env.MAYAR_ENV = env.MAYAR_ENV;
      const product = await createPaymentLink({
        name: "Akses Pro",
        description: "Upgrade ke Pro",
        amount: 150000,
        redirectUrl: `${env.APP_URL}/thanks`,
      });
      return Response.json({ url: product.link });
    }

    return new Response("Not found", { status: 404 });
  },
};
```

Atau refaktor `mayarFetch` agar terima `apiKey` eksplisit (lebih clean untuk Workers):

```ts
export async function mayarFetch<T>(path: string, apiKey: string, env = "production", init?: RequestInit): Promise<T> {
  const BASE = env === "production"
    ? "https://api.mayar.id/hl/v2"
    : "https://api.mayar.club/hl/v2";
  // ... rest sama
}
```

---

## Test

```bash
# Opsi A
npx -y mayar@latest webhook register https://<tunnel>/api/webhooks/mayar

# Tunnel lokal (belum punya domain)
cloudflared tunnel --url http://localhost:3001
```

## Checklist

- [ ] Transaksi `paid`, `fulfill` tereksekusi
- [ ] Webhook duplikat tidak double-fulfill
- [ ] Go-live: `MAYAR_ENV=production` + key production + webhook URL production
- [ ] Workers: env di `wrangler.toml` `[vars]`, secret via `wrangler secret put MAYAR_API_KEY`
