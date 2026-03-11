# Technical Brief: Master Data Lokasi

## 1. Project Overview

**Project Name:** PropTech Management System (Sistem Kos-kosan)
**Module:** Master Data Lokasi
**Objective:** Mengelola data referensi lokasi (seperti Area, Cabang, atau Wilayah) yang akan digunakan sebagai pengelompokan pada saat input data Hunian/Kos-kosan.

**Access Control:** Restricted to **Admin** role only.

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Authentication:** Better-Auth
- **Validation:** Zod (untuk skema form lokasi)

---

## 3. Functional Requirements (User Stories)

Admin memerlukan manajemen lokasi yang efisien untuk memastikan data hunian terorganisir dengan baik:

| ID         | User Story            | Acceptance Criteria                                                                          |
| :--------- | :-------------------- | :------------------------------------------------------------------------------------------- |
| **LOC-01** | **Tambah Lokasi**     | Admin dapat menambah nama lokasi baru (contoh: "BSD City", "Bintaro", "Malang").             |
| **LOC-02** | **Lihat Data Lokasi** | Menampilkan daftar lokasi dalam tabel beserta jumlah hunian yang terikat (optional).         |
| **LOC-03** | **Sunting Lokasi**    | Admin dapat mengubah nama lokasi jika terdapat kesalahan penulisan.                          |
| **LOC-04** | **Hapus Lokasi**      | Admin dapat menghapus lokasi (dengan validasi jika lokasi masih digunakan oleh data hunian). |
| **LOC-05** | **Pencarian Lokasi**  | Admin dapat memfilter daftar lokasi berdasarkan nama lokasi.                                 |

---

## 4. Database Schema (Drizzle ORM)

Skema tabel `locations` pada `src/db/schema.ts`. Tabel ini akan direferensikan oleh tabel `properties` (Hunian) nantinya.

```typescript
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const locations = pgTable("locations", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  description: text("description"), // Tambahan Opsional: Detail alamat/keterangan
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

## 5. Authentication & Authorization Logic

- **Gatekeeping**: Menggunakan logic yang sama dengan Master User. Middleware akan mengecek session role === 'admin'.

- **Action Security**: Setiap Server Action untuk Lokasi wajib memvalidasi session admin sebelum melakukan eksekusi ke database.

## 6. Implementation Detail (Server Actions)

1. getLocations(search?: string):

Mengambil semua lokasi. Jika ada parameter search, gunakan ilike(locations.name, %${search}%).

2. createLocation(data):

Validasi nama lokasi tidak boleh kosong.

Cek apakah nama lokasi sudah ada (Unique check) untuk menghindari duplikasi.

3. updateLocation(id, data):

Melakukan update nama atau deskripsi lokasi berdasarkan ID.

4. deleteLocation(id):

Penting: Tambahkan pengecekan apakah ID lokasi ini sedang digunakan di tabel properties. Jika ya, cegah penghapusan atau berikan peringatan (Restricted Delete).

## 7. UI/UX Specifications

- **Form Input**: Minimal terdiri dari name (Input Text) dan description (Textarea).

- **Selection Tool**: Data dari tabel ini akan di-fetch dan ditampilkan sebagai Dropdown/Select Component pada form "Tambah Hunian".

- **Consistency**: Gunakan desain tabel dan modal yang konsisten dengan modul Master User untuk menjaga User Experience.

## 8. Security & Data Integrity

- **Foreign Key Constraint**: Pastikan integritas data terjaga. Jika lokasi dihapus, sistem harus menangani apakah hunian di bawahnya ikut terhapus (Cascade) atau dicegah (Restrict). Rekomendasi: Restrict.

- **Input Trimming**: Pastikan input nama lokasi di-trim (dihapus spasi depan/belakang) untuk menghindari data ganda seperti "BSD" dan " BSD".
