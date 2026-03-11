# Technical Brief: Detail Hunian & Kamar (Unified View)

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Detailed Property & Room Inquiry
- **Objective:** Menyediakan alur informasi dari level gedung hingga level unit kamar, serta menerapkan filter ketersediaan yang ketat untuk menjamin unit siap dihuni.
- **Access Control:** All Roles (Public View), Booking restricted to Authenticated Tenants.

## 2. Navigation Flow

- **Halaman Detail Hunian (`/property/[id]`):** Fokus pada profil gedung, area publik, dan daftar unit kamar yang tersedia.
- **Halaman Detail Kamar (`/property/[id]/room/[room_id]`):** Fokus pada spesifikasi unit, foto interior, fasilitas dalam kamar, dan opsi skema harga untuk booking.

## 3. Business Logic: Room Availability (Aturan H+2)

Sistem hanya akan menampilkan kamar yang memenuhi kriteria berikut:

1.  **Room Status:** Wajib berstatus `Available` atau `Cleaning`.
2.  **Order Conflict:** Kamar tidak memiliki pesanan aktif pada rentang tanggal yang dicari.
3.  **Maintenance Buffer (H+2):** \* Kamar hanya ditampilkan jika `Check-in Date` > `Last Checkout Date + 2 Hari`.
    - Periode ini digunakan untuk memastikan pembersihan (cleaning) dan pengecekan kerusakan selesai sebelum tenant baru masuk.
    - _Contoh:_ Jika tenant sebelumnya checkout 15 Maret, kamar baru bisa dipesan/tampil untuk check-in tanggal 18 Maret.

## 4. Functional Requirements (User Stories)

| ID         | Function                | User Story                                                                 | Acceptance Criteria                                                            |
| :--------- | :---------------------- | :------------------------------------------------------------------------- | :----------------------------------------------------------------------------- |
| **DET-01** | **Property Gallery**    | Sebagai User, saya melihat galeri foto area publik gedung hunian.          | Menampilkan carousel foto eksterior, lobi, dan fasilitas umum gedung.          |
| **DET-02** | **Price Averaging**     | Sebagai User, saya melihat rata-rata harga di level gedung.                | Sistem menghitung rata-rata harga `Open Subscription` dari unit yang tersedia. |
| **DET-03** | **Property Facilities** | Sebagai User, saya melihat informasi fasilitas umum gedung.                | List fasilitas tipe 'hunian' (e.g., Parkir, WiFi, Security).                   |
| **DET-04** | **Room Listing**        | Sebagai User, saya melihat semua kamar tersedia di gedung tersebut.        | Grid/List kamar yang lolos filter status 'Available/Cleaning' & Buffer H+2.    |
| **DET-05** | **Room Quick Info**     | Sebagai User, saya melihat info ringkas kamar (No. Kamar, Harga Mulai).    | User melihat summary sebelum memutuskan masuk ke detail kamar.                 |
| **DET-06** | **Room Detail Access**  | Sebagai User, saya dapat memilih unit untuk melihat detail spesifik kamar. | Navigasi ke sub-halaman detail unit tanpa kehilangan konteks gedung.           |
| **DET-07** | **Interior Gallery**    | Sebagai User, saya melihat foto interior spesifik unit tersebut.           | Menampilkan foto khusus kamar tersebut (bukan foto umum gedung).               |
| **DET-08** | **Room Pricing Detail** | Sebagai User, saya melihat 3 skema harga lengkap di detail kamar.          | Menampilkan harga detail untuk Open Sub, Komitmen, dan Bayar di Muka.          |
| **DET-09** | **Room Facilities**     | Sebagai User, saya melihat data fasilitas pada Kamar.                      | List fasilitas tipe 'kamar' (e.g., AC, Kasur, Kamar Mandi Dalam).              |
| **DET-10** | **Booking CTA**         | Sebagai Tenant, saya diarahkan ke halaman booking saat klik CTA Booking.   | Tombol mengarahkan ke `/order/[room_id]` setelah validasi status auth.         |

## 5. Implementation Detail (Database Logic)

Sistem menggunakan query untuk mengecek ketersediaan dengan memperhatikan buffer 2 hari:

```typescript
// Pseudocode Logic untuk menampilkan Kamar yang Eligible
const availableRooms = await db
  .select()
  .from(rooms)
  .where(
    and(
      eq(rooms.propertyId, targetPropertyId),
      or(eq(rooms.status, "Available"), eq(rooms.status, "Cleaning")),
      // Penjagaan H+2: Memastikan tidak ada order yang overlap dalam buffer 2 hari
      notExists(
        db
          .select()
          .from(orders)
          .where(
            and(
              eq(orders.roomId, rooms.id),
              // Kamar disembunyikan jika (Checkout Terakhir + 2 Hari) > Tanggal Rencana Check-in
              sql`${orders.checkoutDate} + interval '2 days' > ${requestedCheckInDate}`,
            ),
          ),
      ),
    ),
  );
```

## 6. UI/UX Specifications

A. Level Hunian (Gedung)
Summary Info: Nama Gedung, Alamat, dan Fasilitas Umum.

- Room Grid: Kartu unit kamar menampilkan foto utama, nomor kamar, dan badge status (Ready atau Cleaning).

B. Level Unit (Kamar)
Breadcrumb: Navigasi jelas (Home > Nama Hunian > No. Kamar).

- Pricing Tabs: Interface tab untuk berpindah antara skema harga (Daily/Monthly/Yearly).
- Sticky CTA: Tombol "Pesan Sekarang" tetap berada di posisi bawah (mobile) untuk akses cepat.

## 7. Security & Business Rules

- Auth Guard: Pengunjung umum dapat melihat hingga detail kamar, namun diarahkan ke /auth/login jika menekan tombol Booking.

- Room Status Lock: Jika kamar berstatus Cleaning, sistem tetap mengizinkan booking selama tanggal check-in memenuhi aturan H+2. Jika status Repair, unit otomatis disembunyikan.

- Real-time Check: Validasi ketersediaan dilakukan dua kali (saat render halaman detail dan saat tombol Booking ditekan) untuk menghindari double booking.

## 8. Data Integrity

- Image Fallback: Jika kamar tidak memiliki foto spesifik, sistem secara otomatis menampilkan foto "Standard Room" dari gedung tersebut.

- Price Sync: Setiap perubahan harga oleh Admin di Master Data Kamar akan langsung mengupdate tampilan di Detail Kamar secara real-time.
