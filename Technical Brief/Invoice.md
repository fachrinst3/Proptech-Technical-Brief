# Technical Brief: Invoice Management System

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Invoice Management
- **Objective:** Mengelola penagihan sewa bulanan bagi tenant aktif, otomatisasi pembuatan invoice setiap akhir bulan, dan rekonsiliasi pembayaran.
- **Access Control:** Restricted to **Admin** role only.

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Task Scheduler:** Cron Job (via Vercel Cron, Inngest, atau GitHub Actions)
- **Payment Gateway:** Midtrans / Xendit

## 3. Database Schema (Drizzle ORM)

Skema tabel `invoices` yang berelasi dengan tabel `orders` (sebagai induk sewa) dan `users`.

```typescript
import {
  pgTable,
  text,
  timestamp,
  uuid,
  integer,
  pgEnum,
  date,
} from "drizzle-orm/pg-core";
import { orders } from "./orders";
import { users } from "./auth-schema";

export const invoiceStatusEnum = pgEnum("invoice_status", [
  "UNPAID",
  "PAID",
  "EXPIRED",
  "CANCELLED",
]);

export const invoices = pgTable("invoices", {
  id: uuid("id").defaultRandom().primaryKey(),
  invoiceNumber: text("invoice_number").unique().notNull(), // Contoh: INV/202603/001
  orderId: uuid("order_id").references(() => orders.id, {
    onDelete: "cascade",
  }),
  tenantId: text("tenant_id").references(() => users.id),

  amount: integer("amount").notNull(),
  billingCycle: date("billing_cycle").notNull(), // Periode bulan penagihan
  dueDate: timestamp("due_date").notNull(),

  status: invoiceStatusEnum("status").default("UNPAID").notNull(),
  externalId: text("external_id"), // ID dari Payment Gateway
  paymentUrl: text("payment_url"),
  paymentProof: text("payment_proof"), // Untuk transfer manual
  paidAt: timestamp("paid_at"),

  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

## 4. Functional Requirements (User Stories)

| ID         | Function               | User Story                                                                               | Acceptance Criteria                                                                                                           |
| :--------- | :--------------------- | :--------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------- |
| **INV-01** | **Invoice Monitoring** | Sebagai Admin, saya ingin melihat semua daftar invoice sewa yang ada di platform.        | Menampilkan tabel invoice dengan kolom: No. Invoice, Nama Tenant, Unit Hunian, Kamar, Nominal, Periode Penagihan, dan Status. |
| **INV-02** | **Monthly Filter**     | Sebagai Admin, saya ingin memfilter data invoice berdasarkan bulan.                      | Admin dapat memilih bulan tertentu (billing cycle) untuk melihat daftar tagihan yang terbit di periode tersebut.              |
| **INV-03** | **Status Filter**      | Sebagai Admin, saya ingin memfilter invoice berdasarkan status pembayaran.               | Tersedia opsi filter untuk status: `UNPAID` (Belum Bayar), `PAID` (Lunas), `EXPIRED`, dan `CANCELLED`.                        |
| **INV-04** | **Invoice Detail**     | Sebagai Admin, saya ingin melihat detail rincian dari suatu invoice.                     | Menampilkan breakdown biaya, detail unit yang disewa, informasi tenant, serta riwayat waktu (log) transaksi.                  |
| **INV-05** | **Export Data**        | Sebagai Admin, saya ingin mengekspor data invoice berdasarkan filter yang saya terapkan. | Admin dapat mengunduh data dalam format `.xlsx` atau `.csv` sesuai dengan hasil filter bulan dan status yang aktif.           |
| **INV-06** | **Proof of Payment**   | Sebagai Admin, saya ingin melihat bukti bayar setiap invoice yang berstatus sukses.      | Admin dapat melihat lampiran foto bukti transfer manual atau referensi ID transaksi dari Payment Gateway (Midtrans/Xendit).   |

---

## 4.1 Business Rules & Constraints

1. **Billing Cycle:** Invoice diterbitkan setiap tanggal 27 untuk meng-cover masa sewa bulan depan.
2. **Duplicate Check:** Sistem tidak akan menerbitkan invoice jika untuk `Tenant ID` dan `Periode Bulan` yang sama sudah terdapat invoice yang aktif (mencegah _double billing_).
3. **Audit Trail:** Setiap perubahan status invoice (dari Unpaid ke Paid) harus mencatat stempel waktu (`paidAt`) untuk keperluan laporan keuangan.
4. **Manual Payment:** Admin tetap memiliki otorisasi untuk melakukan "Mark as Paid" secara manual jika tenant membayar tunai atau melalui transfer bank di luar Payment Gateway.

## 5. Automation Logic: Monthly Recurring Billing

Sistem menggunakan Cron Job untuk melakukan pengecekan setiap tanggal 27 pukul 00:01:

- Selection: Sistem mencari semua data di tabel orders yang memiliki status = 'Ongoing'.

- Validation: Memastikan tenant tersebut belum memiliki invoice untuk periode bulan berikutnya.

- Generation:
  - Membuat record baru di tabel invoices.

  - Memanggil API Payment Gateway untuk membuat Link Pembayaran (Invoice).

  - Mengirim notifikasi (Email) kepada tenant bahwa invoice baru telah terbit.

- Due Date: Batas bayar diset secara sistem (misal: 3 hari sebelum masa sewa bulan depan dimulai).

## 6. Implementation Detail (Server Actions)

1. getInvoices(filters): Query join antara invoices, orders, properties, dan rooms.

2. generateMonthlyInvoices(): Fungsi internal (Cron) untuk memproses pembuatan invoice massal.

3. handleInvoiceWebhook(): Webhook handler khusus untuk memperbarui status invoice saat tenant melunasi tagihan di Payment Gateway.

## 7. UI/UX Specifications

- Invoice Status Badge:
  - UNPAID: Kuning/Orange
  - PAID: Hijau
  - EXPIRED/CANCELLED: Abu-abu/Merah

- Billing Timeline: Tampilkan informasi periode sewa (misal: "Sewa Periode April 2026") pada list invoice agar admin tidak bingung.

- Payment Preview: Modal pop-up untuk melihat bukti transfer manual jika tenant tidak menggunakan payment gateway.

## 8. Security & Data Integrity

- **Duplicate Protection**: Logic generateMonthlyInvoices wajib memiliki pengecekan unique berdasarkan orderId + billingCycle untuk mencegah double billing.

- **Amount Locking**: Nominal invoice tidak boleh berubah setelah diterbitkan, kecuali dilakukan pembatalan dan pembuatan ulang oleh Admin.

- **Consistency**: Jika invoice dibayar, sistem dapat secara otomatis memperbarui checkout_date pada tabel orders untuk memperpanjang masa eligible checkout tenant.
