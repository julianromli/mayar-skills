# Pattern: integrasi Mayar (framework-agnostic)

Semua recipe berbagi pola yang sama. File ini satu-satunya sumber pola; file framework hanya berisi wiring.

## Alur (5 tahap)

```
[1] client klik beli
[2] SERVER: create payment link/invoice via API → dapat hosted URL
[3] customer bayar di hosted page Mayar → redirectUrl balik ke app
[4] SERVER: webhook masuk = notifikasi (bukan bukti)
[5] SERVER: re-fetch status via API → paid? → provisioning (idempotent)
```

Yang menentukan kelayakan stack: **apakah ada server runtime** untuk tahap 2, 4, 5.

| Stack | Server primitive | Catatan |
|---|---|---|
| Next.js App Router | route handler (`app/api/**/route.ts`) | recipe: `nextjs.md` |
| TanStack Start | server route handlers | recipe: `tanstack-start.md` |
| Express / Hono / Fastify | route biasa | pakai pola ini apa adanya |
| Laravel | `routes/api.php` | pakai pola ini, HTTP via `Http::` |
| React Vite (SPA murni) | **tidak ada** | wajib 1 function kecil: `vite-react.md` |
| Cloudflare Workers | `fetch` handler | env dari parameter handler, bukan `process.env`. Adaptor: `vite-react.md` |

## Helper (TypeScript murni, jalan di semua server runtime JS)

Simpan sebagai `lib/mayar.ts` (atau setara). Base URL & quirk envelope: Reference di `SKILL.md`.

```ts
const BASE =
  process.env.MAYAR_ENV === "production"
    ? "https://api.mayar.id/hl/v2"
    : "https://api.mayar.club/hl/v2";

export async function mayarFetch<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${process.env.MAYAR_API_KEY}`,
      "Content-Type": "application/json",
      ...init?.headers,
    },
  });
  const body = await res.json();
  const msg = body.messages ?? body.message; // envelope quirk
  if (!res.ok || (body.statusCode ?? 500) >= 400) {
    throw new Error(`Mayar ${path} gagal: ${msg ?? res.status}`);
  }
  return body.data as T;
}

export function createPaymentLink(input: {
  name: string;
  description: string;
  amount: number;       // IDR
  redirectUrl: string;
  expiredAt?: string;   // ISO 8601
}) {
  return mayarFetch<{ id: string; link: string }>("/products/payment-link/create", {
    method: "POST",
    body: JSON.stringify(input),
  });
}

export function getTransaction(id: string) {
  return mayarFetch<{
    id: string;
    status: string;
    amount: number;
    customer: { id: string; email: string; name: string };
    paymentLink: { id: string; type: string };
  }>(`/transactions/${id}`)
}
```

Runtime non-Node (PHP, Python, Go): tiru 3 fungsi ini. Yang penting: Bearer key di server, envelope defensif, endpoint sama.

## Logika webhook (sama di semua stack)

```
parse payload → event != payment.received? balas 200
ambil tx.id dari payload
detail = getTransaction(tx.id)          // bukti dari API, bukan payload
detail.status paid? (paid|success|settled)
detail.id sudah diproses? balas 200     // dedupe: simpan ID di DB, bukan memory
fulfill(customerEmail, productId)       // idempotent
balas 200
```

Komentar wajib di handler: `// TODO: ganti verify-by-fetch dengan signature verification saat Mayar merilis HMAC webhook`.

## 4 keputusan wiring per framework

1. **Di mana route create** (checkout) → balas `{ url }` → client redirect.
2. **Di mana route webhook** → path publik, contoh `/api/webhooks/mayar`.
3. **Cara baca env** → `MAYAR_API_KEY`, `MAYAR_ENV`, `APP_URL`. Server-only.
4. **Redirect target** → halaman sukses di app (`/thanks`), bisa SPA route.

Setelah wiring: register webhook `npx -y mayar@latest --sandbox webhook register <url>`, test end-to-end, dedupe via DB. Detail: checklist di recipe masing-masing.
