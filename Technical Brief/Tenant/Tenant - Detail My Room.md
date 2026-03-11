# Technical Brief: My Room Detail & Lifecycle Management

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Tenant Room Detail (Active Lease Management)
- **Objective:** Memberikan akses kepada tenant untuk memantau detail unit, status deposit, invoice bulanan, serta melakukan manajemen akhir masa sewa (checkout/extend).
- **Access Control:** Authenticated Tenant (Resource Owner Only).

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Payment Gateway:** Midtrans / Xendit (untuk deposit & invoice bulanan)
- **State Management:** React Hook Form + Zod (untuk form checkout)
- **Database:** PostgreSQL + Drizzle ORM

## 3. Business Logic & Automation

### A. Recurring Invoice (Skema 1 & 2)

- **Target:** Tenant dengan Skema 1 (Open Subscription) dan Skema 2 (Komitmen).
- **Trigger:** Tanggal 27 setiap akhir bulan.
- **Action:** Sistem otomatis men-generate invoice sewa untuk bulan berikutnya. Untuk Skema 3 (Bayar di Muka), invoice tidak diterbitkan karena sudah lunas di awal.

### B. Deposit Handling

- **Payment:** Jika tenant belum membayar deposit, tersedia tombol "Bayar Deposit".
- **Verification:** Status dicek melalui webhook payment gateway. Jika sukses, UI berubah menjadi label "Sudah Bayar Deposit".
- **Refund Info:** Saat proses checkout, sistem menampilkan estimasi uang deposit yang akan dikembalikan (Refundable Amount).

### C. Checkout & Extend Logic

- **Open Subscription:** Menggunakan form checkout untuk berhenti sewa.
- **Skema Komitmen/Bayar Dimuka:** Form yang sama digunakan untuk konfirmasi perpanjangan durasi sewa (extend) atau persiapan keluar.

---

## 4. Functional Requirements (User Stories)

| ID         | Function             | User Story                                                             | Acceptance Criteria                                                                |
| :--------- | :------------------- | :--------------------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| **MYD-01** | **Room Detail Info** | Sebagai Tenant, saya melihat informasi detail kamar yang saya sewa.    | Menampilkan No. Kamar, Nama Gedung, Fasilitas, dan Tanggal Check-in.               |
| **MYD-02** | **Cost Information** | Sebagai Tenant, saya melihat informasi data biaya kamar saya.          | Menampilkan rincian harga sewa per periode sesuai skema yang dipilih.              |
| **MYD-03** | **Deposit Status**   | Sebagai Tenant, saya melihat keterangan pembayaran deposit kamar saya. | Menampilkan status "Belum Bayar" atau "Sudah Bayar Deposit" secara real-time.      |
| **MYD-04** | **Pay Deposit**      | Sebagai Tenant, saya dapat melakukan pembayaran deposit.               | Tombol bayar mengarahkan ke Payment Gateway; label update otomatis setelah sukses. |
| **MYD-05** | **Monthly Invoice**  | Sebagai Tenant, saya melihat invoice sewa setiap akhir bulan (Tgl 27). | Khusus Skema 1 & 2. Invoice muncul di tab "Riwayat Pembayaran".                    |
| **MYD-06** | **Checkout Trigger** | Sebagai Tenant, saya bisa klik tombol checkout di halaman detail.      | Tombol membuka akses ke wizard/form pengajuan keluar.                              |
| **MYD-07** | **Checkout Form**    | Sebagai Tenant, saya bisa mengisi form checkout kamar.                 | Field: Tanggal Checkout, Keterangan, dan Checkbox T&C.                             |
| **MYD-08** | **Refund Preview**   | Sebagai Tenant, saya melihat estimasi deposit yang dikembalikan.       | Muncul pada ringkasan form checkout sebelum submit.                                |

---

## 5. Implementation Detail (Form & Logic)

### A. Form Checkout Field (Validation)

1. **Tanggal Checkout:** (Input Date) - Validasi minimal H+1 dari hari ini.
2. **Keterangan:** (Textarea) - Alasan keluar atau catatan untuk admin.
3. **T&C Checkbox:** (Boolean) - Persetujuan mengenai kondisi kamar dan aturan deposit (misal: deposit hangus jika komitmen tidak terpenuhi).

### B. Database Query: Monthly Billing

```typescript
// Logic filter untuk menampilkan invoice bulanan kepada tenant
const monthlyInvoices = await db
  .select()
  .from(invoices)
  .where(
    and(
      eq(invoices.tenantId, currentUserId),
      eq(invoices.orderId, currentOrderId),
      // Hanya tampilkan invoice rutin (bukan booking fee awal)
      sql`EXTRACT(DAY FROM ${invoices.createdAt}) = 27`,
    ),
  );
```

## 6. UI/UX Specifications

- Status Highlights: Gunakan banner hijau untuk "Lunas/Deposit Terbayar" dan banner kuning untuk "Tagihan Baru Tersedia".

- Checkout Wizard: Tampilkan ringkasan kontrak saat ini sebelum user mengisi form checkout untuk meminimalkan kesalahan tanggal.

- Deposit Refund Card: Visualisasi yang jelas mengenai total deposit awal vs potongan (jika ada) vs total yang akan diterima tenant.

## 7. Security & Business Rules

- Access Control: User hanya bisa mengakses detail kamar dengan orderId yang terikat pada userId miliknya.

- Commitment Guard: Jika tenant Skema 2 mencoba checkout sebelum masa komitmen berakhir, sistem akan menampilkan peringatan otomatis: "Deposit Anda akan hangus karena melanggar masa komitmen."

- Checkout Deadline: Sesuai aturan sistem, request checkout idealnya diajukan sesuai aturan H-1 jam 19:00 untuk pemrosesan otomatis.

## 8. Data Integrity

- Invoice Record: Setiap invoice tanggal 27 harus tersimpan dengan metadata billingCycle yang jelas (misal: April 2026).

- Refund Logging: Data pengembalian deposit harus tercatat di sistem admin agar terjadi sinkronisasi saat admin melakukan transfer balik manual/otomatis ke tenant.
