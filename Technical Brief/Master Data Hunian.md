# Technical Brief: Master Data Hunian (Properties)

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Master Data Hunian
- **Objective:** Mengelola data utama properti/kos-kosan, mengintegrasikan lokasi, landmark, dan fasilitas gedung, serta memantau inventori kamar.
- **Access Control:** Restricted to **Admin** role only.

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Authentication:** Better-Auth
- **Validation:** Zod (Complex object validation)

---

## 3. Functional Requirements (User Stories)

Admin bertindak sebagai pengelola aset yang mengatur profil setiap properti:

| ID          | User Story            | Acceptance Criteria                                                                         |
| :---------- | :-------------------- | :------------------------------------------------------------------------------------------ |
| **PROP-01** | **Tambah Hunian**     | Admin dapat input data hunian dengan memilih Lokasi, Landmark, dan Fasilitas yang tersedia. |
| **PROP-02** | **Lihat Data Hunian** | Menampilkan daftar seluruh properti beserta ringkasan jumlah kamar.                         |
| **PROP-03** | **Sunting Hunian**    | Admin dapat memperbarui informasi, foto, atau mengubah daftar fasilitas gedung.             |
| **PROP-04** | **Hapus Hunian**      | Admin dapat menghapus properti (hanya jika sudah tidak ada tenant aktif).                   |
| **PROP-05** | **Pencarian Hunian**  | Filter daftar berdasarkan nama hunian.                                                      |
| **PROP-06** | **Cek Ketersediaan**  | Admin dapat melihat statistik kamar yang tersedia (kosong) vs terisi per hunian.            |

---

## 4. Database Schema (Drizzle ORM)

Skema ini melibatkan relasi dengan tabel yang telah dibuat sebelumnya.

```typescript
import { pgTable, text, timestamp, uuid, integer } from "drizzle-orm/pg-core";
import { locations } from "./locations";

export const properties = pgTable("properties", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  description: text("description"),
  address: text("address").notNull(),
  locationId: uuid("location_id")
    .references(() => locations.id)
    .notNull(),
  thumbnailUrl: text("thumbnail_url"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

// Junction Table untuk Fasilitas (Many-to-Many)
export const propertyFacilities = pgTable("property_facilities", {
  propertyId: uuid("property_id").references(() => properties.id, {
    onDelete: "cascade",
  }),
  facilityId: uuid("facility_id").references(() => facilities.id),
});

// Junction Table untuk Landmark (Many-to-Many)
export const propertyLandmarks = pgTable("property_landmarks", {
  propertyId: uuid("property_id").references(() => properties.id, {
    onDelete: "cascade",
  }),
  landmarkId: uuid("landmark_id").references(() => landmarks.id),
  distance: text("distance"), // Misal: "500m" atau "5 Menit"
});
```

## 5. Authentication & Authorization Logic

- **Admin Verification**: Middleware memastikan hanya user dengan role admin yang bisa memodifikasi data.

- **Data Scoping**: Server actions harus memastikan integrity check (misal: memastikan locationId, LandmarkId, dan FasilitasId yang dikirim benar-benar ada di DB).

## 6. Implementation Detail (Server Actions)

1. getProperties(search?: string):

- Melakukan join dengan tabel locations dan menghitung jumlah kamar kosong dari tabel rooms.

2. createProperty(data):

- Menggunakan DB Transaction untuk insert data properti sekaligus mengisi tabel junction propertyFacilities dan propertyLandmarks.

3. updateProperty(id, data):

- Melakukan sinkronisasi (Upsert/Delete) pada data relasi landmark dan fasilitas.

4. getAvailability(propertyId):

- Query agregasi: count(rooms.id) di mana rooms.status = 'available'.

## 7. UI/UX Specifications

- Multi-Step Form (Optional): Karena data yang diinput banyak (Informasi Umum -> Lokasi -> Fasilitas & Landmark), gunakan form bertahap atau Tab.

- Select Components: - Lokasi: Dropdown (Single Select).

- Landmark: Combobox (Multi Select dengan input jarak).

- Fasilitas: Checkbox Group (Hanya menampilkan tipe 'hunian').

- Dashboard Card: Tampilkan indikator warna untuk hunian yang kamarnya hampir penuh atau kosong semua.

## 8. Security & Data Integrity

- **Delete Protection**: Cegah penghapusan Hunian jika masih terdapat data Kamar yang terikat dengan kontrak Tenant aktif.

- **Image Upload**: Gunakan library upload (misal: UploadThing atau Cloudinary) dan simpan URL-nya di PostgreSQL.

- **Transactional Consistency**: Gunakan db.transaction saat menyimpan relasi Many-to-Many untuk mencegah data parsial jika terjadi error di tengah jalan.
