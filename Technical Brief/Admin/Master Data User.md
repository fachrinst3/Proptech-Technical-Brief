# Technical Brief: Master Data User Management

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Master Data User
- **Objective:** Menyediakan antarmuka bagi Admin untuk mengelola seluruh akun pengguna (Tenant/Staf) dalam platform secara terpusat.
- **Access Control:** Restricted to **Admin** role only.

---

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Authentication:** Better-Auth (with Role-Based Access Control)
- **Styling:** Tailwind CSS + Shadcn UI (Recommended for consistency)

---

## 3. Functional Requirements (User Stories)

Berdasarkan kebutuhan Product Owner, sistem harus memenuhi kriteria berikut:

| ID        | User Story          | Acceptance Criteria                                                                                |
| :-------- | :------------------ | :------------------------------------------------------------------------------------------------- |
| **US-01** | **Tambah User**     | Admin dapat mengisi form (Nama, Email, Role, Password) dan menyimpannya ke database.               |
| **US-02** | **Lihat Data User** | Menampilkan daftar user dalam bentuk tabel dengan kolom: Nama, Email, Role, dan Tanggal Dibuat.    |
| **US-03** | **Sunting User**    | Admin dapat mengubah informasi user yang sudah ada (kecuali email jika diperlukan untuk keamanan). |
| **US-04** | **Hapus User**      | Admin dapat menghapus user dengan konfirmasi (soft delete atau hard delete sesuai kebijakan).      |
| **US-05** | **Pencarian**       | Filter tabel secara real-time berdasarkan input nama user.                                         |

---

## 4. Database Schema (Drizzle ORM)

Implementasi pada `src/db/schema.ts` menggunakan integrasi tabel `user` dari Better-Auth dengan tambahan field `role`.

```typescript
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("user", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  emailVerified: boolean("emailVerified").notNull(),
  image: text("image"),
  role: text("role").$type<"admin" | "tenant">().default("tenant").notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().notNull(),
});
```

## 5. Authentication & Authorization Logic

Keamanan akses halaman dikelola melalui kombinasi Middleware dan Server-side check:

- **Role Check**: Hanya user dengan role: 'admin' yang dapat mengakses route /dashboard/users.

- **Session Management**: Menggunakan auth.getSession() dari Better-Auth untuk memverifikasi identitas Admin pada setiap request.

- **Middleware**: Redirect otomatis ke halaman login jika user tidak terautentikasi atau bukan Admin.

## 6. Implementation Detail (Server Actions)

Gunakan pola Next.js Server Actions untuk memastikan logika bisnis berada di sisi server:

1. getUsers(search?: string):

- Query: db.select().from(users).where(ilike(users.name, %${search}%))

2. createUser(data):

- Melakukan validasi Zod sebelum db.insert.

- Mengirimkan email verifikasi (opsional) via Better-Auth.

3. updateUser(id, data):

- Melakukan db.update pada user ID spesifik.

4. deleteUser(id):

- Eksekusi db.delete dengan proteksi agar Admin tidak menghapus dirinya sendiri.

## 7. UI/UX Specifications

Antarmuka harus intuitif dan responsif:

-**Table Component**: Menggunakan DataTable (TanStack Table) dengan fitur pagination.

-**Search Input**: Dilengkapi dengan debounce (minimal 300ms) untuk optimasi performa query.

-**Modal Form**: Form tambah/edit user muncul dalam bentuk Dialog/Modal agar Admin tidak kehilangan konteks halaman.

-**Feedback System**: Notifikasi visual (Toast) untuk status "Berhasil" atau "Gagal" pada setiap aksi.

## 8. Security Considerations

- Strict RBAC: Pastikan API endpoint (Server Actions) divalidasi ulang untuk mengecek role user, bukan hanya mengandalkan proteksi di level UI.
- Data Integrity: Gunakan transaksi database jika operasi melibatkan lebih dari satu tabel.
- Audit Trail: Simpan informasi updatedAt secara otomatis setiap kali ada perubahan data user.
- Error Handling: Jangan menampilkan error database mentah ke Admin; gunakan pesan error yang user-friendly.
