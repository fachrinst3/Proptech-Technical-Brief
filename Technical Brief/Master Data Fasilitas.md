# Technical Brief: Master Data Fasilitas

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Master Data Fasilitas
- **Objective:** Mengelola daftar fasilitas standar yang akan digunakan sebagai referensi saat input data Hunian (Building Facilities) dan data Kamar (Room Facilities).
- **Access Control:** Restricted to **Admin** role only.

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Authentication:** Better-Auth
- **Validation:** Zod

---

## 3. Functional Requirements (User Stories)

Admin memerlukan manajemen fasilitas yang terorganisir berdasarkan kategorinya:

| ID         | User Story               | Acceptance Criteria                                                             |
| :--------- | :----------------------- | :------------------------------------------------------------------------------ |
| **FAS-01** | **Tambah Fasilitas**     | Admin dapat menambah nama fasilitas dan menentukan tipenya (Hunian atau Kamar). |
| **FAS-02** | **Lihat Data Fasilitas** | Menampilkan daftar seluruh fasilitas beserta tipenya dalam satu tabel.          |
| **FAS-03** | **Sunting Fasilitas**    | Admin dapat mengubah nama fasilitas atau memindahkan kategori tipenya.          |
| **FAS-04** | **Hapus Fasilitas**      | Admin dapat menghapus fasilitas yang tidak lagi digunakan.                      |
| **FAS-05** | **Pencarian Fasilitas**  | Admin dapat mencari fasilitas berdasarkan nama melalui fitur search.            |

---

## 4. Database Schema (Drizzle ORM)

Skema tabel `facilities` menggunakan `enum` atau `text` untuk membedakan kategori fasilitas.

```typescript
import { pgTable, text, timestamp, uuid, pgEnum } from "drizzle-orm/pg-core";

// Definisi tipe fasilitas
export const facilityTypeEnum = pgEnum("facility_type", ["hunian", "kamar"]);

export const facilities = pgTable("facilities", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  type: facilityTypeEnum("type").notNull().default("hunian"),
  icon: text("icon"), // Opsional: Menyimpan nama icon (misal: 'wifi', 'air-conditioner')
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

## 5. Authentication & Authorization Logic

- **Gatekeeping**: Sama seperti modul master lainnya, akses dibatasi melalui middleware bagi user dengan role: 'admin'.

- **Server-Side Guard**: Memastikan fungsi create, update, dan delete mengecek validitas session admin untuk mencegah bypass API.

## 6. Implementation Detail (Server Actions)

1. getFacilities(search?: string, type?: 'hunian' | 'kamar'):

- Mengambil data fasilitas dengan filter pencarian nama dan filter opsional berdasarkan tipe.

2. createFacility(data):

- Validasi Zod: name (required), type (must be 'hunian' or 'kamar').

3. updateFacility(id, data):

- Memperbarui informasi fasilitas berdasarkan ID.

4. deleteFacility(id):

- Menghapus record. Implementasikan pengecekan relasi agar tidak merusak data Hunian atau Kamar yang sudah terlanjur menggunakan fasilitas tersebut.

## 7. UI/UX Specifications

- **Badge Type**: Pada tabel utama, tampilkan tipe fasilitas menggunakan Badge (misal: Biru untuk 'Hunian', Hijau untuk 'Kamar') agar mudah dibedakan secara visual.

- **Filter Toggle**: Sediakan filter cepat di atas tabel untuk memfilter tampilan: "Semua", "Fasilitas Hunian", atau "Fasilitas Kamar".

- **Form Selection**:

  \*Saat input Master Hunian: Hanya tampilkan fasilitas dengan type = 'hunian'.

  \*Saat input Master Kamar: Hanya tampilkan fasilitas dengan type = 'kamar'.

- **Icon Picker** (Opsional): Jika memungkinkan, sediakan pilihan icon sederhana agar tampilan di sisi Tenant lebih menarik.

## 8. Security & Data Integrity

- **Normalized Data:** Pastikan nama fasilitas disimpan dalam format standar (contoh: "WiFi", bukan "wifi " atau "WIFI").

- **Referential Integrity**: Gunakan tabel junction (misal: property_facilities dan room_facilities) untuk menghubungkan fasilitas dengan Hunian atau Kamar secara Many-to-Many.

- **Duplicate Prevention**: Gunakan composite unique constraint pada name dan type untuk menghindari duplikasi fasilitas yang sama di kategori yang sama.
