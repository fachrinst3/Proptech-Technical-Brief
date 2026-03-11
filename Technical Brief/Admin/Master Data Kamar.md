# Technical Brief: Master Data Kamar (Room)

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Master Data Kamar
- **Objective:** Mengelola detail fisik unit kamar, status operasional, serta konfigurasi skema komersial (harga dan deposit) per unit.
- **Access Control:** Restricted to **Admin** role only.

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Authentication:** Better-Auth
- **Validation:** Zod

---

## 3. General Data Kamar (Physical Attributes)

Setiap unit kamar wajib memiliki informasi dasar sebagai berikut:

- **Identitas Unit:** Nomor/Nama Kamar (misal: "103 C") dan Nomor Lantai.
- **Dimensi:** Ukuran kamar (misal: "3x4 meter").
- **Deskripsi:** Detail tambahan mengenai kondisi atau kelebihan unit tersebut.
- **Media:** Galeri foto interior kamar (Thumbnail & Additional Images).
- **Fasilitas Kamar:** Relasi ke Master Fasilitas (khusus `type: kamar`).

---

## 4. Functional Requirements (User Stories)

| ID          | User Story            | Acceptance Criteria                                                                 |
| :---------- | :-------------------- | :---------------------------------------------------------------------------------- |
| **ROOM-01** | **Tambah Data Kamar** | Admin dapat input data general, foto, fasilitas, dan skema harga unit.              |
| **ROOM-02** | **Update Status**     | Admin mengelola siklus unit: `Available`, `Occupied`, `Need to Repair`, `Cleaning`. |
| **ROOM-03** | **Manajemen Harga**   | Admin set nilai untuk: Open Subscription, Komitmen Sewa, dan Bayar Langsung.        |
| **ROOM-04** | **Filter & Search**   | Pencarian berdasarkan Nama Kamar dan Filter berdasarkan Hunian induk.               |

---

## 5. Database Schema (Drizzle ORM)

Skema ini menggabungkan data general dan skema pembayaran:

```typescript
import {
  pgTable,
  text,
  timestamp,
  uuid,
  integer,
  pgEnum,
  jsonb,
} from "drizzle-orm/pg-core";
import { properties } from "./properties";

export const roomStatusEnum = pgEnum("room_status", [
  "Available",
  "Occupied",
  "Need to Repair",
  "Cleaning",
]);

export const rooms = pgTable("rooms", {
  id: uuid("id").defaultRandom().primaryKey(),
  propertyId: uuid("property_id")
    .references(() => properties.id, { onDelete: "cascade" })
    .notNull(),

  // --- GENERAL DATA ---
  roomNumber: text("room_number").notNull(),
  floor: integer("floor").notNull(),
  dimension: text("dimension"), // Contoh: "3x4"
  description: text("description"),
  images: text("images").array(), // Menyimpan array URL foto

  // --- STATUS & DEPOSIT ---
  status: roomStatusEnum("status").default("Available").notNull(),
  depositAmount: integer("deposit_amount").default(0).notNull(),

  // --- PRICING SCHEMES ---
  openSubscriptionPrice: integer("open_subscription_price"), // Rp/Hari

  // Skema Komitmen (Contoh: 1 Bln, 3 Bln)
  commitmentSchemes:
    jsonb("commitment_schemes").$type<
      { duration: number; unit: "month" | "year"; price: number }[]
    >(),

  // Skema Bayar Langsung (Contoh: 6 Bln, 1 Thn)
  upfrontSchemes:
    jsonb("upfront_schemes").$type<
      { duration: number; unit: "month" | "year"; price: number }[]
    >(),

  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

## 6. Business Logic & Pricing Example

Admin wajib mengisi nilai pada field berikut sesuai skenario:

- **Data General Kamar**: Input Nomor Kamar "103 C", Lantai 1, Ukuran 3x4.

- **Open Subscription**: Digunakan untuk sewa jangka pendek/fleksibel (Rp/hari).

- **Komitmen Sewa**: Digunakan untuk kontrak bulanan (1 Bln, 3 Bln, dst) dengan harga grosir.

- **Pembayaran Langsung (Upfront)**: Digunakan untuk sewa jangka panjang dengan pembayaran di muka (6 Bln, 1 Thn) yang biasanya lebih murah dari harga bulanan akumulatif.

## 6. Logic Bisnis & Skema Pembayaran

Admin harus mengisi nilai berikut pada setiap unit:

- **Open Subscription**: Digunakan untuk sewa jangka pendek/fleksibel (Rp/hari).

- **Komitmen Sewa**: Digunakan untuk kontrak bulanan (1 Bln, 3 Bln, dst) dengan harga grosir.

- **Pembayaran Langsung (Upfront)**: Digunakan untuk sewa dengan pembayaran di muka (6 Bln, 1 Thn) yang biasanya lebih murah dari harga bulanan akumulatif.

## 7. UI/UX Specifications

- **Pricing Form**: Gunakan fitur Dynamic Fields (tambah baris) untuk skema komitmen dan bayar langsung agar Admin bisa menambah durasi sesuai keinginan (misal: nambah opsi 2 tahun).

- **Status Indicator**: Gunakan warna yang kontras untuk status:
  - Available: Hijau
  - Occupied: Merah
  - Need to Repair: Kuning/Orange
  - Cleaning: Biru

- **Room Card/Grid**: Di halaman list, tampilkan ringkasan harga harian dan bulanan untuk mempermudah Admin.

## 8. Data Integrity & Validation

- **Format Mata Uang**: Gunakan Intl.NumberFormat untuk menampilkan harga Rupiah di UI, namun tetap simpan sebagai integer di database (dalam satuan Rupiah penuh, bukan sen).

- **Logic Status**: Kamar yang berstatus Occupied tidak boleh dihapus dari sistem kecuali tenant terkait sudah dipindahkan/checkout.

- **Cross-Validation**: Saat input fasilitas kamar, sistem hanya menampilkan data dari master fasilitas yang memiliki type: 'kamar'.

## 9. Implementation Detail (Server Actions)

1. getRooms(filters: { propertyId?: string, search?: string }):

- Melakukan fetch data kamar dengan filter join ke tabel Hunian.

2. updateRoomStatus(roomId, newStatus):

- Aksi cepat untuk mengubah status kamar (misal: setelah tenant checkout, status otomatis/manual menjadi Cleaning).

3. savePricingSchemes(roomId, data):

- Server action khusus untuk memproses input harga yang kompleks (Open Sub, Commitment, Upfront).
