# Technical Brief: Tenant Dashboard (Overview)

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Tenant Dashboard
- **Objective:** Memberikan ringkasan informasi kritis kepada tenant mengenai unit aktif, sisa tagihan, masa sewa, dan status deposit dalam satu halaman utama.
- **Access Control:** Authenticated Tenant Only.

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **UI Components:** Shadcn UI (Cards, Progress Bar, Badges)

## 3. Data Aggregation Logic

Dashboard ini menampilkan data hasil agregasi dari beberapa tabel:

- **Current Room:** Data dari tabel `rooms` yang terhubung dengan `orders` berstatus `Ongoing`.
- **Sisa Pembayaran:** Akumulasi nominal dari tabel `invoices` yang berstatus `UNPAID` untuk `orderId` terkait.
- **Masa Sewa:** Selisih hari antara `checkInDate` dan `checkoutDate` (atau durasi berjalan untuk Open Sub).
- **Status Deposit:** Status kolom `depositStatus` pada tabel `orders` atau `invoices` kategori deposit.

## 4. Functional Requirements (User Stories)

| ID         | Function                | User Story                                                                        | Acceptance Criteria                                                                 |
| :--------- | :---------------------- | :-------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------- |
| **TSH-01** | **Active Room Summary** | Sebagai Tenant, saya bisa melihat data kamar yang sekarang sedang saya sewa.      | Menampilkan Card berisi Nama Hunian, No. Kamar, dan Foto Utama unit.                |
| **TSH-02** | **Outstanding Balance** | Sebagai Tenant, saya bisa melihat sisa pembayaran yang harus saya lakukan.        | Menampilkan total nominal tagihan (Invoice Aktif + Deposit jika belum bayar).       |
| **TSH-03** | **Lease Period Detail** | Sebagai Tenant, saya bisa melihat detail masa sewa kamar saya.                    | Menampilkan tanggal Check-in, estimasi Checkout, dan progres durasi (Progress Bar). |
| **TSH-04** | **Deposit Status Info** | Sebagai Tenant, saya bisa melihat keterangan status pembayaran uang deposit saya. | Menampilkan label status: `BELUM BAYAR`, `PROSES VERIFIKASI`, atau `LUNAS`.         |

---

## 5. Implementation Detail (Server-Side Logic)

### A. Dashboard Data Fetching

```typescript
// Mengambil data ringkasan untuk Dashboard Tenant
const dashboardData = await db
  .select({
    roomName: rooms.roomNumber,
    propertyName: properties.name,
    checkIn: orders.checkInDate,
    checkOut: orders.checkOutDate,
    depositStatus: orders.depositStatus,
    totalUnpaid: sql<number>`sum(${invoices.amount}) filter (where ${invoices.status} = 'UNPAID')`,
  })
  .from(orders)
  .innerJoin(rooms, eq(orders.roomId, rooms.id))
  .innerJoin(properties, eq(rooms.propertyId, properties.id))
  .leftJoin(invoices, eq(orders.id, invoices.orderId))
  .where(and(eq(orders.tenantId, currentUserId), eq(orders.status, "Ongoing")))
  .groupBy(orders.id, rooms.id, properties.id);
```

## 6. UI/UX Specifications

- Metric Cards: Menggunakan 4 kartu utama di bagian atas untuk (1) Unit Aktif, (2) Total Tagihan, (3) Sisa Hari Sewa, (4) Status Deposit.

- Lease Progress Bar: Visualisasi durasi sewa yang sudah terlewati vs total durasi kontrak (khusus Skema Komitmen).

- Quick Actions: Tombol cepat "Bayar Sekarang" jika terdapat sisa pembayaran (totalUnpaid > 0).

- Mobile Responsive: Layout satu kolom pada mobile untuk memastikan angka-angka keuangan mudah dibaca.

## 7. Security & Performance

- Data Isolation: Memastikan filter where(tenantId === session.userId) selalu diterapkan untuk mencegah kebocoran data antar tenant.

- Caching: Menggunakan revalidate setiap 1 jam, namun melakukan force refresh jika tenant baru saja menyelesaikan pembayaran gateway.

- Empty State: Jika tenant belum memiliki sewa aktif, dashboard menampilkan pesan sambutan dan tombol "Cari Kamar Pertama Kamu".

## 8. Data Integrity

- Real-time Sync: Angka "Sisa Pembayaran" harus otomatis berkurang detik itu juga setelah webhook Payment Gateway mengirimkan status settlement.

- Date Consistency: Tanggal masa sewa harus sinkron dengan data yang tertera pada Invoice PDF agar tidak membingungkan tenant.
