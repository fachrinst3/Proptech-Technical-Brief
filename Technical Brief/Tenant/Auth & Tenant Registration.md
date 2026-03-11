# Technical Brief: Authentication & Tenant Registration

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Auth & Register
- **Objective:** Menyediakan sistem otentikasi yang aman untuk tenant, mencakup pendaftaran akun baru, login ke dashboard pribadi, dan manajemen sesi berbasis role.
- **Access Control:** Public (Register/Login), Tenant (Dashboard/Logout).

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Auth Library:** Better-Auth / NextAuth.js v5
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Validation:** Zod
- **Encryption:** Argon2 / Bcrypt

## 3. Database Schema (Auth Integration)

Skema tabel standar untuk integrasi otentikasi:

```typescript
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("user", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  password: text("password").notNull(),
  role: text("role").$type<"admin" | "tenant">().default("tenant").notNull(),
  emailVerified: boolean("emailVerified").default(false).notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().notNull(),
});

export const sessions = pgTable("session", {
  id: text("id").primaryKey(),
  userId: text("userId").references(() => users.id, { onDelete: "cascade" }),
  token: text("token").notNull(),
  expiresAt: timestamp("expiresAt").notNull(),
});
```

## 4. Functional Requirements (User Stories)

## 5. Implementation Detail & Logic

A. Registration Logic

- Validation: Menggunakan Zod untuk memastikan format email valid dan password minimal 8 karakter.

- Duplication Check: Sistem mengecek ketersediaan email sebelum melakukan INSERT.

- Hashing: Password di-hash (Argon2/Bcrypt) sebelum masuk ke database.

B. Session Management

- Cookie-Based: Menyimpan session token dalam HttpOnly Cookie agar aman dari XSS.
- Role Redirect: Middleware memastikan Admin masuk ke /admin dan Tenant ke /tenant.

## 6. UI/UX Specifications

- Form Feedback: Menampilkan pesan error atau sukses menggunakan Toast/Alert.

- Loading State: Tombol login/register menampilkan spinner saat proses request berlangsung.

- Password Visibility: Icon "eye" untuk melihat atau menyembunyikan password saat diketik.

- Responsive: Layout form yang fleksibel untuk perangkat desktop maupun mobile.

## 7. Security & Performance

- Rate Limiting: Membatasi percobaan login yang salah untuk mencegah Brute Force.

- CSRF Protection: Setiap request POST dilindungi oleh CSRF token dari library Auth.

- Index Optimization: Kolom email pada database di-index untuk pencarian akun yang cepat.
