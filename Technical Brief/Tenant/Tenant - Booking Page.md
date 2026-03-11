# Technical Brief: Booking Wizard & Payment Flow

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Booking & Payment Wizard
- **Objective:** Menyediakan proses pemesanan unit kamar yang terstruktur melalui wizard form, pemilihan skema harga, dan integrasi pembayaran hingga penerbitan bukti bayar PDF.
- **Access Control:** Authenticated Tenant Only.

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **State Management:** React Hook Form + Zod (Multi-step state)
- **Payment Gateway:** Midtrans / Xendit
- **PDF Generation:** `react-pdf` atau `puppeteer` (Server-side PDF)
- **Database:** PostgreSQL + Drizzle ORM

## 3. Booking Logic & Pricing Schemes

Sistem mendukung tiga skema utama yang dipilih pada Step 1 :

1. **Open Subscription:** Sewa fleksibel (harian/mingguan/bulanan) tanpa ikatan jangka panjang.
2. **Komitmen Sewa:** \* User memilih durasi tertentu (misal: 3, 6, atau 12 bulan).
   - **Mandatory Note:** "Jika pembayaran tidak sesuai dengan komitmen (berhenti di tengah jalan), maka uang deposit akan dinyatakan hangus."
3. **Bayar Langsung:** Pembayaran penuh di muka untuk periode panjang (biasanya dengan diskon khusus).

---

## 4. Functional Requirements (User Stories)

| ID         | Function             | User Story                                                                         | Acceptance Criteria                                                          |
| :--------- | :------------------- | :--------------------------------------------------------------------------------- | :--------------------------------------------------------------------------- |
| **BOK-01** | **Wizard Interface** | Sebagai Tenant, saya bisa melihat halaman booking dalam bentuk wizard form.        | Layout form dibagi menjadi 2 Step utama (Review & Payment).                  |
| **BOK-02** | **Step 1: Review**   | Sebagai Tenant, saya bisa melihat review booking dan mengisi data diri.            | Input: Nama Penghuni, No. KTP, No. WA. Tampilan: Rincian kamar yang dipilih. |
| **BOK-03** | **Scheme Selection** | Sebagai Tenant, saya bisa memilih skema (Open Sub, Komitmen, atau Bayar Langsung). | Opsi skema muncul dinamis sesuai data master kamar.                          |
| **BOK-04** | **Commitment Note**  | Sebagai Tenant, saya mendapat peringatan pada skema komitmen sewa.                 | Menampilkan teks peringatan tentang hangusnya deposit jika komitmen langgar. |
| **BOK-05** | **Step 2: Payment**  | Sebagai Tenant, saya bisa melihat halaman pembayaran.                              | Menampilkan ringkasan tagihan final sebelum memilih metode bayar.            |
| **BOK-06** | **Payment Method**   | Sebagai Tenant, saya bisa memilih metode pembayaran (VA, E-Wallet, CC).            | Integrasi pilihan metode bayar dari Payment Gateway (Xendit/Midtrans).       |
| **BOK-07** | **Bill Calculation** | Sebagai Tenant, saya melihat rincian nominal pembayaran.                           | Breakdown: Sewa Bulan Pertama + Deposit + Biaya Admin (jika ada).            |
| **BOK-08** | **Success State**    | Sebagai Tenant, saya mendapat bukti pemesanan setelah bayar.                       | Tampilan Success Page dengan ringkasan transaksi pasca-callback gateway.     |
| **BOK-09** | **PDF Invoice**      | Sebagai Tenant, saya bisa mencetak bukti bayar/invoice dalam bentuk PDF.           | Tombol "Cetak PDF" yang men-generate invoice resmi secara otomatis.          |

---

## 5. Implementation Detail (Wizard Flow)

### Step 1: Data Diri & Rincian Pembayaran

- **Validation:** Validasi NIK (16 digit) dan kelengkapan dokumen pendukung.
- **Calculation Engine:** Sistem menghitung total di frontend secara reaktif saat user berpindah antar skema harga.

### Step 2: Checkout & Payment Gateway

1. Saat user klik "Lanjut ke Pembayaran", sistem membuat record `Order` dengan status `Waiting for Payment`.
2. Server memanggil API Payment Gateway untuk mendapatkan `payment_url` atau `snap_token`.
3. User diarahkan ke antarmuka pembayaran gateway atau memilih VA di dalam sistem.

---

## 6. UI/UX Specifications

- **Progress Indicator:** Progress bar di bagian atas (Step 1: Data & Skema -> Step 2: Bayar).
- **Summary Sidebar:** Di sisi kanan layar, tampilkan ringkasan: Nama Kamar, Tanggal Check-in, dan Total Harga yang terus terupdate.
- **Commitment Warning:** Box berwarna kuning/merah khusus untuk skema komitmen agar tidak terlewat oleh user.
- **PDF Template:** Desain invoice yang profesional mencakup Logo, Nomor Invoice, Detail Unit, dan QR Code validasi.

---

## 7. Security & Business Rules

1. **Idempotency:** Mencegah pembuatan double order jika user menekan tombol "Bayar" berkali-kali.
2. **Session Timeout:** Jika dalam 2 jam (atau sesuai setting gateway) tidak dibayar, order otomatis `Expired` dan kamar kembali tersedia (H+2 logic tetap berlaku).
3. **Data Persistence:** Jika user tidak sengaja me-refresh halaman pada Step 2, data Step 1 tetap tersimpan (via LocalStorage atau Session).

---

## 8. Data Integrity (PDF Generation)

- **Server-Side Generation:** PDF di-generate di sisi server untuk memastikan data tidak bisa dimanipulasi oleh user melalui inspect element.
- **Storage:** Invoice PDF yang sudah terbit disimpan di S3/Supabase Storage agar bisa diunduh kapan saja melalui riwayat invoice.
