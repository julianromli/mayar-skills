# Recipe: Next.js (App Router)

Pola, helper `lib/mayar.ts`, dan logika webhook: `_pattern.md`. File ini hanya wiring Next.js.

## Env

`.env.local` + pastikan `.gitignore` mencakup `.env*`:

```bash
MAYAR_API_KEY=paste_key_sandbox_di_sini
MAYAR_ENV=sandbox
APP_URL=http://localhost:3000
```

## Checkout — `app/api/checkout/route.ts`

```ts
import { NextResponse } from "next/server";
import { createPaymentLink } from "@/lib/mayar"; // dari _pattern.md

export async function POST() {
  const product = await createPaymentLink({
    name: "Akses Pro 1 Bulan",
    description: "Upgrade ke Pro",
    amount: 150000,
    redirectUrl: `${process.env.APP_URL}/thanks`,
  });
  return NextResponse.json({ url: product.link });
}
```

## Tombol beli (client component)

```tsx
"use client";

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

## Webhook — `app/api/webhooks/mayar/route.ts`

```ts
import { NextResponse } from "next/server";
import { getTransaction } from "@/lib/mayar";

// TODO: ganti verify-by-fetch dengan signature verification saat Mayar merilis HMAC webhook.
// Produksi: ganti Set dengan tabel DB processedTransactions.
const processed = new Set<string>();

async function fulfill(customerEmail: string, productId: string) {
  // PROVISIONING: beri akses / kirim download / upgrade akun. Idempotent.
  console.log("fulfill", customerEmail, productId);
}

export async function POST(req: Request) {
  const payload = await req.json();
  const tx = payload.data ?? {};
  if ((payload.event ?? payload.type) !== "payment.received" || !tx.id) {
    return NextResponse.json({ ok: true });
  }

  const detail = await getTransaction(tx.id); // bukti dari API
  const paid = ["paid", "success", "settled"].includes(String(detail.status).toLowerCase());
  if (!paid) return NextResponse.json({ ok: true });

  if (processed.has(detail.id)) return NextResponse.json({ ok: true });
  processed.add(detail.id);

  await fulfill(detail.customer.email, detail.paymentLink.id);
  return NextResponse.json({ ok: true });
}
```

## Test

```bash
npx -y mayar@latest --sandbox webhook register https://<domain-atau-tunnel>/api/webhooks/mayar
```

Belum punya domain: tunnel (`cloudflared tunnel --url http://localhost:3000` / ngrok) atau polling `getTransaction` tiap 5-10 detik dari route/cron server.

## Checklist

- [ ] Transaksi sandbox `paid`, `fulfill` tereksekusi
- [ ] Webhook duplikat tidak double-fulfill
- [ ] Go-live: `MAYAR_ENV=production` + key production + webhook URL production
