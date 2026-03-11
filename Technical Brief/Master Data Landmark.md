# Technical Brief: Master Data Landmark

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Master Data Landmark
- **Objective:** Mengelola poin poin penting atau tempat ikonik (POIs) yang akan digunakan sebagai referensi kedekatan lokasi pada saat input data Hunian/Kos-kosan.
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

Admin dapat mengelola landmark untuk memperkaya informasi detail hunian:

| ID         | User Story              | Acceptance Criteria                                                                      |
| :--------- | :---------------------- | :--------------------------------------------------------------------------------------- |
| **LND-01** | **Tambah Landmark**     | Admin dapat menambah nama landmark (contoh: "Stasiun Sudirman", "Mall Grand Indonesia"). |
| **LND-02** | **Lihat Data Landmark** | Menampilkan daftar seluruh landmark dalam tabel.                                         |
| **LND-03** | **Sunting Landmark**    | Admin dapat memperbarui nama atau kategori landmark.                                     |
| **LND-04** | **Hapus Landmark**      | Admin dapat menghapus landmark yang tidak lagi relevan.                                  |
| **LND-05** | **Pencarian Landmark**  | Admin dapat mencari landmark berdasarkan nama menggunakan search bar.                    |

---

## 4. Database Schema (Drizzle ORM)

Skema tabel `landmarks` pada `src/db/schema.ts`. Relasi dengan Hunian nantinya bisa bersifat _Many-to-Many_ atau _One-to-Many_ tergantung kebutuhan bisnis.

```typescript
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const landmarks = pgTable("landmarks", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  category: text("category"), // Opsional: misal 'Transportasi', 'Pusat Perbelanjaan', 'Kampus'
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

## 5. Authentication & Authorization Logic

- **Gatekeeping**: Menggunakan middleware untuk memastikan hanya user dengan role: 'admin' yang bisa mengakses endpoint /admin/landmarks.

- **Server Action Protection**: Memverifikasi session melalui auth.getSession() sebelum mengeksekusi perintah Write (Create/Update/Delete).

## 6. Implementation Detail (Server Actions)

1. getLandmarks(search?: string):

- Mengambil data landmark dengan filter ilike pada field name.

2. createLandmark(data):

- Validasi Zod untuk memastikan name terisi.

- Normalisasi teks (Auto-capitalize atau Trim).

3. updateLandmark(id, data):

- Melakukan update pada kolom name atau category.

4. deleteLandmark(id):

- Menghapus record landmark. Catatan: Jika digunakan di Hunian, sistem harus menangani relasi tersebut agar tidak terjadi data orphan.

## 7. UI/UX Specifications

- **Search UX**: Implementasi pencarian sisi server (Server-side search) agar performa tetap ringan meski data landmark banyak.

- **Selection UI**: Pada form Hunian, Landmark ditampilkan dalam bentuk Multi-select atau Combobox (karena satu hunian bisa dekat dengan banyak landmark).

- **Feedback**: Gunakan Toast component untuk notifikasi keberhasilan CRUD.

## 8. Security & Data Integrity

- **Unique Constraint**: Disarankan memberikan constraint unique pada nama landmark untuk mencegah duplikasi yang membingungkan Admin.

- **Input Validation**: Batasi panjang karakter nama landmark (misal: max 100 karakter) untuk menjaga kerapian UI.

- **Relational Check**: Jika landmark dihapus, pastikan referensi pada tabel Hunian diset menjadi NULL atau record di tabel junction ikut terhapus (Cascade).
