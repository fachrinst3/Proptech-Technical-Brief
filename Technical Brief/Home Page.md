# Technical Brief: Master Data User Management

## 1. Project Overview

**Project Name:** PropTech Management System (Sistem Kos-kosan)
**Module:** Master Data User
**Objective:** Menyediakan antarmuka bagi Admin untuk mengelola seluruh akun pengguna (Tenant/Staf) dalam platform secara terpusat.
**Access Control:** Restricted to **Admin** role only.

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
