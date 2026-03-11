# Technical Brief: Order & Booking Management (Final & Updated)

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Order / Booking
- **Objective:** Mengelola siklus hidup transaksi secara otomatis, mulai dari pemesanan, verifikasi pembayaran melalui Payment Gateway, manajemen sewa aktif, hingga proses checkout tenant.
- **Access Control:** Restricted to **Admin** role only (for Management).

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL (PostgreSQL via Supabase/Local)
- **ORM:** Drizzle ORM
- **Payment Gateway:** Midtrans / Xendit (Webhook Integration)
- **Export Tool:** `exceljs` (Excel) / `json2csv` (CSV)

## 3. Order & Operational Status

### A. Order Status (Transaction Flow)

- **Waiting for Payment:** Booking dibuat, menunggu pembayaran via Gateway atau Admin.
- **Deposit Received:** Pembayaran uang jaminan/booking fee berhasil diverifikasi.
- **Deposit and Payment Received:** Pembayaran penuh (Sewa + Deposit) lunas.
- **Ongoing:** Tenant aktif menghuni kamar (Status Sewa Aktif).
- **Done:** Masa sewa berakhir secara normal dan tenant sudah checkout.
- **Cancel:** Pembatalan manual oleh Admin (biasanya untuk booking offline/admin-side).
- **Expired:** Tenant tidak melakukan pembayaran hingga batas waktu link pembayaran berakhir.

### B. Room Status (Sync Logic)

Sistem harus mensinkronisasi status kamar berdasarkan status order:

- Order `Ongoing` ➔ Kamar: `Occupied`
- Order `Done`/`Cancel` ➔ Kamar: `Cleaning` ➔ `Available` (setelah dibersihkan).

## 4. Database Schema (Drizzle ORM)

Skema ini mengintegrasikan data tenant, unit, dan detail transaksi payment gateway.

```typescript
import {
  pgTable,
  text,
  timestamp,
  uuid,
  integer,
  pgEnum,
  date,
  boolean,
} from "drizzle-orm/pg-core";
import { users } from "./auth-schema";
import { properties } from "./properties";
import { rooms } from "./rooms";

export const orderStatusEnum = pgEnum("order_status", [
  "Waiting for Payment",
  "Deposit Received",
  "Deposit and Payment Received",
  "Cancel",
  "Expired",
  "Ongoing",
  "Done",
]);

export const orders = pgTable("orders", {
  id: uuid("id").defaultRandom().primaryKey(),
  tenantId: text("tenant_id")
    .references(() => users.id)
    .notNull(),
  propertyId: uuid("property_id")
    .references(() => properties.id)
    .notNull(),
  roomId: uuid("room_id")
    .references(() => rooms.id)
    .notNull(),

  // Detail Sewa
  totalAmount: integer("total_amount").notNull(),
  checkInDate: date("check_in_date").notNull(),
  duration: integer("duration").notNull(), // Bulan/Hari
  checkoutDate: date("checkout_date"),

  // Payment Gateway Integration
  status: orderStatusEnum("status").default("Waiting for Payment").notNull(),
  externalId: text("external_id").unique(), // Referensi Midtrans/Xendit
  paymentUrl: text("payment_url"),
  paymentMethod: text("payment_method"),
  paymentProof: text("payment_proof"), // Manual upload backup
  paidAt: timestamp("paid_at"),

  // Checkout Logic
  isCheckoutRequested: boolean("is_checkout_requested").default(false),
  requestedCheckoutDate: date("requested_checkout_date"),

  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

## 5. User Stories (Admin Role)

| ID         | Function                        | User Story                                                                            | Acceptance Criteria                                                                                                              |
| :--------- | :------------------------------ | :------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------- |
| **ORD-01** | **Global Monitoring**           | Sebagai Admin, saya ingin melihat semua data booking/order di platform.               | Menampilkan list dengan field: ID Order, Nama Tenant, Unit Hunian, No. Kamar, Harga, Tanggal Check-in, Durasi, dan Status Order. |
| **ORD-02** | **Bulan & Status Filter**       | Sebagai Admin, saya ingin melakukan filter data berdasarkan bulan dan status.         | Admin dapat memfilter tampilan berdasarkan bulan transaksi dan kategori status (Waiting, Ongoing, Done, dll).                    |
| **ORD-03** | **Order Detail View**           | Sebagai Admin, saya ingin melihat detail lengkap dari satu pesanan.                   | Menampilkan informasi rinci tenant, riwayat pembayaran, informasi unit, dan lampiran bukti bayar.                                |
| **ORD-04** | **Data Export**                 | Sebagai Admin, saya ingin mengekspor data booking berdasarkan filter yang diterapkan. | Tersedia tombol export ke format `.xlsx` atau `.csv` yang mengikuti parameter filter aktif (Bulan/Status).                       |
| **ORD-05** | **Payment Gateway Integration** | Sebagai Admin, saya ingin verifikasi pembayaran terjadi secara otomatis.              | Integrasi Midtrans/Xendit memperbarui status ke `Deposit and Payment Received` secara otomatis via Webhook.                      |
| **ORD-06** | **Proof of Payment Access**     | Sebagai Admin, saya ingin melihat bukti bayar untuk setiap transaksi sukses.          | Admin dapat mengakses log sukses gateway atau melihat upload foto bukti bayar untuk transaksi manual.                            |
| **ORD-07** | **Active Rental Stats**         | Sebagai Admin, saya ingin melihat total jumlah sewa yang aktif saat ini.              | Menampilkan widget "Single Data Stats" yang menghitung total order dengan status `Ongoing`.                                      |
| **ORD-08** | **Payment History**             | Sebagai Admin, saya ingin melihat riwayat seluruh pembayaran sewa seorang tenant.     | Tampilan log kronologis dari semua invoice dan transaksi yang terkait dengan ID Tenant tertentu.                                 |
| **ORD-09** | **Checkout Monitoring**         | Sebagai Admin, saya ingin melihat request checkout otomatis dari tenant.              | Sistem menampilkan flag atau notifikasi pada tenant yang melakukan request checkout (khusus skema Open Subscription).            |
| **ORD-10** | **Manual Checkout**             | Sebagai Admin, saya ingin bisa melakukan proses checkout tenant secara manual.        | Admin memiliki tombol aksi untuk mengubah status sewa menjadi `Done` dan mengosongkan status kamar secara paksa.                 |

## 6. Implementation Logic (Checkout & Payments)

A. Otomasi Payment Gateway (Webhook)

- Endpoint: /api/webhooks/payment
- Logic:

1. Validasi signature/token gateway.
2. Jika status settlement: Update order ke Deposit and Payment Received.
3. Jika status expire: Update order ke Expired dan set kamar kembali ke Available.

B. Logika Checkout (Eligibilitas)

- Open Subscription Case: - Tenant eligible checkout jika H-1 sebelum jam 19:00.
  - Tanggal tercepat: Tanggal terakhir invoice lunas (misal: invoice berakhir 31 Maret, checkout tercepat 31 Maret).

- Manual Override: Admin memiliki tombol "Force Checkout" untuk menyelesaikan masa sewa secara manual di sistem.

## 7. UI/UX Specifications

- **Order Table**: Menampilkan status dalam bentuk Badge berwarna (Yellow: Waiting, Green: Ongoing, Red: Cancel/Expired).

- **Detail Modal**: Menampilkan riwayat pembayaran, data unit kamar, dan link ke profil tenant.

- **Proof Preview**: Fitur "Click to Zoom" pada bukti bayar yang diunggah manual.

- **Export UI**: Tombol export yang mengikuti state filter bulan dan status yang aktif di dashboard.

## 8. Security & Data Integrity

- **Idempotency**: Webhook gateway harus ditangani secara idempoten (cegah update ganda untuk notifikasi yang sama).

- **Room Lock**: Kamar otomatis terkunci (tidak bisa di-book user lain) selama status order masih Waiting for Payment hingga masa berlaku invoice berakhir.

- **Audit Trail**: Mencatat log setiap perubahan status order untuk keperluan rekonsiliasi keuangan.
