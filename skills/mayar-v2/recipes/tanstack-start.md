# Recipe: TanStack Start

Pola, helper `lib/mayar.ts`, dan logika webhook: `_pattern.md`. File ini hanya wiring TanStack Start.

Catatan: API server route Start beberapa kali berubah. Kalau snippet ini error compile, cek docs resmi TanStack Start (server routes / handlers), pola isinya tetap sama.

## Env

`.env` + pastikan `.gitignore` mencakupnya. Start membaca `process.env` di server route:

```bash
MAYAR_API_KEY=paste_key_sandbox_di_sini
MAYAR_ENV=sandbox
APP_URL=http://localhost:3000
```

## Checkout — `src/routes/api/checkout.ts`

```ts
import { createFileRoute } from "@tanstack/react-router";
```ts
export const Route = createFileRoute("/api/checkout")({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const body = (await request.json()) as CheckoutBody;
        const product = await createPaymentLink({
          name: "Akses Pro 1 Bulan",
          description: "Upgrade ke Pro",
          amount: 150000,
          redirectUrl: `${process.env.APP_URL}/thanks`,
        });
        return Response.json({ url: product.link });
      },
    },
  },
});
```

> **`paymentMethod` values (tidak terdokumentasi di public docs — verified production):**
> - QRIS: `"qris"` (lowercase)
> - VA BNI: `"va/bni"` — pola `"va/{bank_code_lowercase}"`
> - VA Mandiri: `"va/mandiri"`
> - VA BRI: `"va/bri"`, VA BSI: `"va/bsi"`, VA CIMB: `"va/cimb"`, VA Permata: `"va/permata"`
>
> `paymentDetail` tersedia langsung di create response — tidak perlu GET ulang.
> Sandbox tidak support `paymentMethod` spesifik (semua return 400). Test VA/QRIS harus pakai production key.

## Tombol beli (client)

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

## Webhook — `src/routes/api/webhooks/mayar.ts`

```ts
import { createFileRoute } from "@tanstack/react-router";
import { getTransaction } from "../../../lib/mayar";

interface WebhookPayload {
  event?: string;
  type?: string;
  data?: { id: string };
}

// TODO: ganti verify-by-fetch dengan signature verification saat Mayar merilis HMAC webhook.
// Dedupe via DB — Set tidak persist saat server restart, jangan dipakai di production.
// Contoh Drizzle:
//   await db.insert(processedTx).values({ txId: detail.id }).onConflictDoNothing()
//   const existing = await db.query.processedTx.findFirst({ where: eq(processedTx.txId, detail.id) })
//   if (existing) return Response.json({ ok: true })
// Ganti blok `processed.has / processed.add` di bawah dengan query DB sesuai ORM kamu.

async function fulfill(customerEmail: string, productId: string) {
  // PROVISIONING: idempotent.
  console.log("fulfill", customerEmail, productId);
}

export const Route = createFileRoute("/api/webhooks/mayar")({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const payload = (await request.json()) as WebhookPayload;
        const tx = payload.data ?? {};
        if ((payload.event ?? payload.type) !== "payment.received" || !tx.id) {
          return Response.json({ ok: true });
        }

        const detail = await getTransaction(tx.id); // bukti dari API
        const paid = ["paid", "success", "settled"].includes(String(detail.status).toLowerCase());
        if (!paid) return Response.json({ ok: true });

        if (processed.has(detail.id)) return Response.json({ ok: true });
        processed.add(detail.id);

        await fulfill(detail.customer.email, detail.paymentLink.id);
        return Response.json({ ok: true });
      },
    },
  },
});
```

## Test

```bash
npx -y mayar@latest --sandbox webhook register https://<domain-atau-tunnel>/api/webhooks/mayar
```

Belum punya domain: tunnel (cloudflared/ngrok) atau server function polling `getTransaction` tiap 5-10 detik.

## Checklist

- [ ] Transaksi sandbox `paid`, `fulfill` tereksekusi
- [ ] Webhook duplikat tidak double-fulfill
- [ ] Go-live: `MAYAR_ENV=production` + key production + webhook URL production
