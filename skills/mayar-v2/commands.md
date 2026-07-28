# Mayar CLI — Katalog Command (branch OPS)

Snapshot v1.2.0. Sumber authoritative = CLI itu sendiri:
`npx -y mayar@latest <command> --help` dan `npx -y mayar@latest docs <query>`.
Kalau file ini dan `--help` beda, ikuti `--help`.

Semua command diawali `npx -y mayar@latest`. Tambahkan `--json` untuk output machine-readable, `--sandbox`/`--production` untuk pilih environment.

## Setup & Config

```bash
npx -y mayar@latest whoami                    # verifikasi key + identitas merchant
npx -y mayar@latest status                    # environment, identitas, status key
npx -y mayar@latest api-key <key>             # simpan key non-interaktif
npx -y mayar@latest login [--no-browser]      # OAuth via browser
npx -y mayar@latest config show               # path config + key termask
npx -y mayar@latest balance                   # saldo akun
```

## Docs (cari schema API resmi)

```bash
npx -y mayar@latest docs <query>                      # cari topik (top 5)
npx -y mayar@latest docs <slug>                       # isi docs lengkap
npx -y mayar@latest docs payment --json --compact     # hemat token untuk agent
npx -y mayar@latest docs --refresh                    # refresh cache index
```

## Invoices

```bash
npx -y mayar@latest invoice list [--limit N --after CURSOR]
npx -y mayar@latest invoice get <id>
npx -y mayar@latest invoice create --data '<json|@file>'
npx -y mayar@latest invoice edit <id> --data '<json|@file>'
npx -y mayar@latest invoice status <id> <open|close|active|closed|unlisted>
npx -y mayar@latest invoice close <id>
npx -y mayar@latest invoice reopen <id>
npx -y mayar@latest invoice filter --email <email>
```

## Products & Payment Links

```bash
npx -y mayar@latest product list [--limit N --search Q --type T]
npx -y mayar@latest product search <keyword>
npx -y mayar@latest product get <id>
npx -y mayar@latest product create --type <ebook|course|membership|saas|event|webinar> --data '<json|@file>'
npx -y mayar@latest product edit <id> --data '<json|@file>'
npx -y mayar@latest product status <id> <open|close|active|closed|unlisted>
npx -y mayar@latest product transactions <id>
npx -y mayar@latest payment-link edit <id> --data '<json|@file>'
```

## Single Payments

```bash
npx -y mayar@latest payment list [--status paid|unpaid|closed]
npx -y mayar@latest payment get <id>
npx -y mayar@latest payment create --data '<json|@file>'
npx -y mayar@latest payment edit <id> --data '<json|@file>'
npx -y mayar@latest payment status <id> <open|close|active|closed|unlisted>
```

## Customers

```bash
npx -y mayar@latest customer list
npx -y mayar@latest customer get <id>
npx -y mayar@latest customer create --data '<json|@file>'
npx -y mayar@latest customer search <email>
npx -y mayar@latest customer update <fromEmail> <toEmail>
npx -y mayar@latest customer magic-link <email>
```

## Transactions

```bash
npx -y mayar@latest tx list [--status S --customerId ID --startAt T --endAt T]
npx -y mayar@latest tx unpaid
npx -y mayar@latest tx daily
npx -y mayar@latest tx product <productId>
```

## Reviews

```bash
npx -y mayar@latest review list [--status S --paymentLinkId ID --rating N]
npx -y mayar@latest review stats [productId]
npx -y mayar@latest review create --data '<json|@file>'
npx -y mayar@latest review update <id> --data '<json|@file>'
npx -y mayar@latest review bulk-status --data '<json|@file>'
```

## QR & Payment Channels

```bash
npx -y mayar@latest qrcode <amount_idr>     # dynamic QRIS
npx -y mayar@latest qrcode static           # static QRIS image
npx -y mayar@latest qrcode channels         # channel aktif
```

## Webhooks

```bash
npx -y mayar@latest webhook register <url>
npx -y mayar@latest webhook test <url>
npx -y mayar@latest webhook history
npx -y mayar@latest webhook new-history
npx -y mayar@latest webhook retry <historyId>
```

## Membership & Licensing

```bash
npx -y mayar@latest membership members --productId <id>
npx -y mayar@latest membership tiers --productId <id>
npx -y mayar@latest membership register --data '<json|@file>'
npx -y mayar@latest saas activate <licenseCode> <productId>
npx -y mayar@latest saas deactivate <licenseCode> <productId>
npx -y mayar@latest saas verify <licenseCode> <productId>
npx -y mayar@latest software verify <licenseCode> <productId>
```

## Credit (API only — tidak ada subcommand CLI)

Credit usage-based dikelola via HTTP API langsung, bukan CLI. Gunakan `mayarFetch` dari `_pattern.md`:

| Aksi | Method | Path |
|---|---|---|
| Tambah credit customer | POST | `/customers/{customerId}/credits/add` |
| Kurangi credit (spend) | POST | `/customers/{customerId}/credits/spend` |
| Cek saldo credit | GET | `/customers/{customerId}/credits/balance` |
| Riwayat credit | GET | `/customers/{customerId}/credits/history` |
| Register customer ke produk credit | POST | `/customers/register-credit-usage` |

Konfirmasi path & body schema via `npx -y mayar@latest docs add-customer-credit --json`.

## Global Flags

| Flag | Fungsi |
| --- | --- |
| `--json` | Output JSON mentah |
| `--compact` | JSON ringkas (khusus `docs`) |
| `--limit N` | Page size (default 10, max 50) |
| `--after CURSOR` | Cursor pagination (`nextStartingAfter` dari response sebelumnya) |
| `--api-key <key>` | Override key per invocation |
| `--sandbox` / `--production` / `--env <v>` | Pilih environment |
| `--data <json|@file>` | Body request, inline atau file |
| `--refresh` | Paksa refetch cache (`docs`) |

## Error

- Exit code non-zero = gagal.
- `"valid": false` dari `whoami` = key tidak ada / salah. Lihat Setup di `SKILL.md`.
- HTTP error: `{ "statusCode": 400, "messages": "..." }`.
