# Technical Brief: Admin Dashboard Analytics

## 1. Project Overview

- **Project Name:** PropTech Management System (Sistem Kos-kosan)
- **Module:** Dashboard Admin
- **Objective:** Memberikan visualisasi data analitik mengenai performa hunian, tingkat okupansi kamar, tren transaksi bulanan, dan peringkat properti terpopuler.
- **Access Control:** Restricted to **Admin** role only.

## 2. Tech Stack Reference

- **Framework:** Next.js (App Router)
- **Data Visualization:** Recharts atau Shadcn Charts (Chart.js/Tremor)
- **Database:** PostgreSQL
- **ORM:** Drizzle ORM
- **Styling:** Tailwind CSS + Shadcn UI Cards

## 3. Database Aggregation Logic

Data dashboard tidak memerlukan tabel baru, melainkan hasil agregasi (query) dari tabel yang sudah ada:

- **Okupansi:** Dihitung dari perbandingan total kamar vs kamar dengan status `Occupied` di tabel `rooms`.
- **Transaksi:** Dihitung dari jumlah transaksi sukses (`PAID`) di tabel `invoices` atau `orders`.
- **Top Properti:** Dihitung dari jumlah relasi `Ongoing` sewa di tabel `orders` yang dikelompokkan berdasarkan `propertyId`.

## 4. Functional Requirements (User Stories)

| ID          | Function               | User Story                                                                                       | Acceptance Criteria                                                                                                                       |
| :---------- | :--------------------- | :----------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| **DASH-01** | **Preview Analitik**   | Sebagai Admin, saya ingin melihat halaman preview analitik dari seluruh data hunian dan kamar.   | Menampilkan ringkasan total unit hunian, total seluruh kamar, total kamar tersedia, dan total kamar dalam pemeliharaan (Repair/Cleaning). |
| **DASH-02** | **Statistik Okupansi** | Sebagai Admin, saya ingin melihat statistik data okupansi tiap hunian yang ada di sistem.        | Menampilkan visualisasi persentase (%) keterisian kamar untuk masing-masing gedung hunian (Progress Bar/Pie Chart).                       |
| **DASH-03** | **Tren Transaksi**     | Sebagai Admin, saya ingin melihat tingkat transaksi sewa tenant perbulan dalam bentuk bar chart. | Bar Chart menampilkan tren total revenue atau jumlah transaksi sukses bulanan selama 1 tahun terakhir.                                    |
| **DASH-04** | **Top 3 Hunian**       | Sebagai Admin, saya ingin melihat top 3 hunian yang memiliki tenant aktif terbanyak.             | Menampilkan daftar 3 nama hunian dengan jumlah status sewa `Ongoing` tertinggi secara berurutan.                                          |

---

---

## 5. Implementation Detail (Server-Side Logic)

1. **`getGlobalStats()`**:
   - Query: Menghitung `count()` pada tabel `properties` dan `rooms` (dengan filter status).
2. **`getOccupancyByProperty()`**:
   - Query: `GROUP BY properties.id` untuk menghitung rasio kamar `Occupied` terhadap total kamar per properti.
3. **`getTransactionMonthlyChart()`**:
   - Query: Agregasi `sum(amount)` dari tabel `invoices` di mana `status = 'PAID'`, dikelompokkan berdasarkan bulan.
4. **`getTopProperties()`**:
   - Query: `db.select().from(orders).where(status = 'Ongoing').limit(3)` dengan sorting jumlah terbanyak.

## 6. UI/UX Specifications

- **Grid Layout:** Menggunakan sistem grid 4 kolom untuk metric utama (Total Rooms, Occupied, Available, Revenue).
- **Interactive Charts:** Bar Chart harus memiliki fitur _tooltip_ yang menunjukkan angka detail saat kursor diarahkan ke batang chart.
- **Occupancy Visuals:** Properti dengan tingkat okupansi rendah (misal < 20%) diberikan indikator warna merah sebagai peringatan bagi Admin.
- **Refresh Rate:** Data di-cache menggunakan Next.js `revalidate` setiap 5-10 menit untuk menjaga performa.

## 7. Security & Performance

1. **Query Optimization:** Gunakan _Index_ pada kolom `status` di tabel `rooms` dan `orders` untuk mempercepat proses agregasi data yang besar.
2. **Role Protection:** Dashboard hanya dapat di-render jika `session.user.role === 'admin'`. Jika tidak, arahkan ke halaman Unauthorized.
3. **Skeleton Loading:** Gunakan React Suspense atau Skeleton UI saat data analitik sedang di-fetch untuk memberikan pengalaman pengguna yang halus.

## 8. Data Integrity

- Dashboard harus mencerminkan data real-time. Jika seorang tenant melakukan checkout, angka okupansi di dashboard harus berkurang secara otomatis pada saat halaman di-refresh atau melalui revalidasi data.
