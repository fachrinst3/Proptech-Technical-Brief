# Technical Brief: Visitor Home Page (Landing Page)

## 1. Project Overview

**Project Name:** PropTech Management System (Sistem Kos-kosan)
**Module:** Visitor Home Page
**Objective:** Menyediakan antarmuka publik bagi pengunjung untuk menjelajahi ketersediaan hunian, melakukan pencarian berbasis lokasi atau landmark, serta melihat properti terbaru.
**Access Control:** Public (All Roles).

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Search Engine:** Simple PG Search (ILike) atau integrasi Algolia/Meilisearch (Opsional untuk skala besar).
- **UI Components:** Tailwind CSS + Framer Motion (untuk animasi transisi list).

## 3. Database Interaction Logic

Halaman ini bersifat _read-only_ bagi pengunjung dan menggunakan data dari tabel-tabel berikut:

- **Properties & Rooms:** Menampilkan data hunian dan unit kamar yang tersedia.
- **Locations:** Menampilkan daftar area/wilayah untuk filter utama.
- **Landmarks:** Menampilkan titik poin penting untuk pencarian spesifik (misal: "Dekat Kampus").

## 4. Functional Requirements (User Stories)

| ID          | Function              | User Story                                                                                | Acceptance Criteria                                                                                        |
| :---------- | :-------------------- | :---------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------- |
| **HOME-01** | **Landing Access**    | Sebagai User, saya bisa melihat halaman utama platform.                                   | Menampilkan hero section, fitur unggulan, dan ringkasan platform saat pertama kali diakses.                |
| **HOME-02** | **Location Filter**   | Sebagai User, saya bisa melihat filter untuk mencari data hunian berdasarkan nama lokasi. | Tersedia dropdown atau search bar yang terhubung dengan tabel `locations`.                                 |
| **HOME-03** | **Property Listing**  | Sebagai User, saya bisa melihat list data hunian yang tersedia pada platform.             | Menampilkan kartu hunian (Card) yang memuat Foto, Nama, Lokasi, dan Start Harga.                           |
| **HOME-04** | **Room Search (Loc)** | Sebagai User, saya bisa mencari data kamar berdasarkan filter lokasi.                     | Menampilkan unit kamar yang tersedia di wilayah spesifik yang dipilih user.                                |
| **HOME-05** | **Room Search (Lnd)** | Sebagai User, saya bisa mencari data kamar berdasarkan filter landmark.                   | Menampilkan unit kamar yang berada dalam radius/terhubung dengan landmark tertentu (misal: Dekat Stasiun). |
| **HOME-06** | **Property Search**   | Sebagai Tenant/User, saya bisa mencari data kamar berdasarkan nama hunian.                | Pencarian teks (Search Bar) yang mencocokkan input user dengan nama properti/hunian.                       |
| **HOME-07** | **Latest Properties** | Sebagai User, saya bisa melihat top 5 hunian terbaru yang ada di platform.                | Menampilkan seksi "Terbaru" yang mengurutkan properti berdasarkan `createdAt` (Descending).                |

## 5. Implementation Detail (Server-Side Logic)

1. **`getPublicProperties(filters)`**:
   - Logic: Menggabungkan filter `locationId`, `landmarkId`, dan `searchString`.
   - Query: Menggunakan `LEFT JOIN` antara `properties` dan `rooms` untuk memastikan hanya menampilkan hunian yang memiliki kamar dengan status `Available`.
2. **`getLatestProperties()`**:
   - Query: `db.select().from(properties).orderBy(desc(properties.createdAt)).limit(5)`.
3. **`getSearchReference()`**:
   - Fetch data `locations` dan `landmarks` secara paralel untuk mengisi opsi pada komponen filter/search.

---

## 6. UI/UX Specifications

- **Hero Search:** Search bar utama yang mencolok di bagian atas halaman dengan integrasi auto-complete.
- **Property Card:**
  - Thumbnail Gambar (dengan Lazy Loading).
  - Badge Lokasi & Fasilitas Utama.
  - Harga Terendah (Start from Rp...).
- **Responsive Grid:** Tampilan grid 1 kolom (Mobile), 2 kolom (Tablet), dan 3-4 kolom (Desktop).
- **Zero Result State:** Tampilan informatif jika hunian atau kamar yang dicari tidak ditemukan, dilengkapi saran lokasi lain.

## 7. Performance & SEO

1. **Server Side Rendering (SSR):** Gunakan SSR atau ISR (Incremental Static Regeneration) untuk halaman Home agar terindeks baik oleh Search Engine (SEO).
2. **Image Optimization:** Menggunakan komponen `next/image` untuk kompresi otomatis foto hunian guna mempercepat _First Contentful Paint_.
3. **Caching:** Cache hasil query filter lokasi dan landmark selama 24 jam karena data ini jarang berubah.

## 8. Data Integrity

- **Availability Check:** Sistem harus memastikan hunian yang ditampilkan memiliki minimal 1 kamar dengan status `Available`. Jika seluruh kamar `Occupied`, hunian tersebut secara otomatis disembunyikan dari list publik atau diberi label "Penuh".
- **Sync Data:** Perubahan data pada Master Data Hunian (oleh Admin) harus langsung tercermin di halaman Home setelah proses revalidasi cache.
