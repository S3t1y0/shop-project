# Dokumen Spesifikasi Teknis: Web Toko Buku Online Berbasis Laravel

Dokumen ini merangkum seluruh perencanaan pengembangan platform e-commerce toko buku, mencakup Product Requirement Document (PRD), Entity Relationship Diagram (ERD), alur pengguna (User Flow), dan rencana implementasi teknis.

---

## 1. Product Requirement Document (PRD)

### 1.1 Ringkasan Eksekutif
Sistem ini dirancang untuk melayani transaksi jual beli buku fisik secara daring menggunakan kerangka kerja Laravel. Fokus utama proyek bertumpu pada kemudahan penelusuran katalog, kecepatan proses pemesanan, akurasi pemotongan stok secara waktu nyata, serta kejelasan pemantauan status pesanan bagi pelanggan dan pengelola toko.

### 1.2 Sasaran dan Batasan Sistem
* **Sasaran Fungsional:**
  * Pelanggan dapat menemukan buku berdasarkan judul, pengarang, ISBN, atau kategori dalam hitungan detik.
  * Transaksi pemesanan dapat diselesaikan melalui alur checkout yang ringkas.
  * Pelacakan status pengiriman dan pembayaran disajikan secara terperinci dan transparan.
  * Pengelola memiliki kendali penuh terhadap manajemen inventaris dan perubahan status faktur.
* **Batasan Rilis Awal (MVP):**
  * Pengiriman dibatasi untuk kurir domestik dengan skema tarif flat atau integrasi API logistik dasar.
  * Pembayaran mendukung transfer bank manual dengan bukti bayar dan integrasi satu jalur gerbang pembayaran digital.
  * Fitur ulasan, penilaian produk, dan sistem poin loyalitas dialokasikan untuk fase pengembangan berikutnya.

### 1.3 Matriks Peran dan Hak Akses

| Peran Pengguna | Deskripsi Peran | Batasan Akses dan Fungsionalitas |
|---|---|---|
| Pengunjung | Pengguna anonim tanpa sesi masuk | Menjelajahi etalase, mencari buku, membuka detail spesifikasi buku, menyimpan item pada keranjang sesi sementara. |
| Pelanggan | Pengguna terdaftar dan terverifikasi | Mengelola keranjang belanja permanen, melakukan proses checkout, melihat riwayat transaksi, memantau resi pengiriman. |
| Administrator | Pengelola operasional dan gudang | Mengelola entitas buku dan kategori, memvalidasi bukti pembayaran transfer, menerbitkan nomor resi, memperbarui status operasional. |

### 1.4 Kebutuhan Fungsional

#### A. Pengelolaan Katalog Buku
* **Antarmuka Katalog:** Menampilkan buku dalam bentuk kisi kartu interaktif berisi sampul buku, judul, nama penulis, harga satuan, dan indikator ketersediaan stok.
* **Pencarian dan Penyaringan:** Menyediakan fasilitas pencarian teks bebas (judul/ISBN) serta filter berdasarkan kategori genre, batas harga, dan ketersediaan barang.
* **Informasi Produk Lengkap:** Memuat sinopsis, jumlah halaman, penerbit, tanggal terbit, dimensi fisik, dan sisa stok fisik.
* **Penguncian Stok:** Stok berkurang seketika saat pesanan dibuat dan otomatis dikembalikan ke katalog jika transaksi melewati batas waktu pembayaran.

#### B. Keranjang Belanja dan Pemesanan
* **Manajemen Keranjang:** Memungkinkan penambahan item, modifikasi kuantitas, dan penghapusan buku dengan validasi batas stok secara langsung.
* **Formulir Checkout:** Merekam alamat lengkap tujuan pengiriman, nama penerima, kontak aktif, dan catatan pengiriman.
* **Penerbitan Faktur:** Menghasilkan kode unik transaksi berformat `INV-YYYYMMDD-XXXX` yang memuat rincian harga barang, tarif kirim, serta batas waktu pembayaran.

#### C. Riwayat dan Pelacakan Pesanan
* **Halaman Transaksi:** Daftar kronologis seluruh transaksi pengguna dengan status pemrosesan yang terbarui secara otomatis.
* **Linimasa Status:** Pelacakan tahapan pesanan yang terdiri dari:
  * `pending`: Menunggu pembayaran diselesaikan oleh pelanggan.
  * `verifying`: Bukti transfer telah diunggah dan menunggu persetujuan admin.
  * `processing`: Pembayaran terkonfirmasi; pesanan sedang disiapkan dan dikemas.
  * `shipped`: Paket telah diserahkan ke ekspedisi; nomor resi telah terbit.
  * `completed`: Paket diterima oleh pelanggan; transaksi selesai.
  * `cancelled`: Pesanan dibatalkan karena kedaluwarsa atau stok tidak mencukupi.

#### D. Dasbor Pengelola (Admin)
* Modul CRUD untuk katalog buku, manajemen kategori, dan unggah berkas gambar sampul.
* Antarmuka verifikasi pembayaran manual untuk memeriksa keabsahan bukti transfer.
* Antarmuka logistik untuk memasukkan nomor resi ekspedisi dan mengubah status pesanan.

### 1.5 Kebutuhan Non-Fungsional

* **Keamanan:**
  * Proteksi Cross-Site Request Forgery (CSRF) aktif pada seluruh rute pengiriman formulir.
  * Pengacakan kata sandi menggunakan enkripsi standar Bcrypt atau Argon2id.
  * Pembatasan laju permintaan (*rate limiting*) pada rute autentikasi dan pembuatan pesanan untuk mencegah serangan bot.
  * Pemisahan hak akses menggunakan middleware dan policy bawaan Laravel.
* **Keandalan dan Performa:**
  * Penerapan eager loading pada pemanggilan relasi Eloquent guna mengeliminasi kueri N+1.
  * Pengindeksan basis data pada kolom penelusuran: `isbn`, `slug`, `category_id`, dan `status`.
  * Waktu respons muat halaman katalog berada di bawah 1,5 detik pada koneksi internet standar.

---

## 2. Arsitektur Teknis dan Desain Basis Data

### 2.1 Tumpukan Teknologi
* **Framework:** Laravel 11.x (PHP 8.2+)
* **Basis Data:** MySQL 8.0 / PostgreSQL 16
* **Autentikasi:** Laravel Breeze
* **Antarmuka:** Blade, Tailwind CSS, Alpine.js (atau Laravel Livewire)
* **Manajemen Antrean:** Redis / Database Driver untuk pengiriman notifikasi email dan pembatalan faktur otomatis
* **Penyimpanan Berkas:** Local Storage / AWS S3 / Cloudflare R2

### 2.2 Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o{ ORDERS : "places"
    USERS ||--o{ CART_ITEMS : "has"
    CATEGORIES ||--o{ BOOKS : "contains"
    BOOKS ||--o{ CART_ITEMS : "referenced in"
    BOOKS ||--o{ ORDER_ITEMS : "ordered in"
    ORDERS ||--|{ ORDER_ITEMS : "composed of"
    ORDERS ||--o{ ORDER_STATUS_LOGS : "records"

    USERS {
        bigint id PK
        string name
        string email UK
        string password
        string phone
        string role "admin/customer"
        timestamp email_verified_at
        timestamp created_at
        timestamp updated_at
    }

    CATEGORIES {
        bigint id PK
        string name
        string slug UK
        text description
        timestamp created_at
        timestamp updated_at
    }

    BOOKS {
        bigint id PK
        bigint category_id FK
        string title
        string slug UK
        string author
        string publisher
        string isbn UK
        decimal price "12,2"
        integer stock
        string cover_image
        text description
        timestamp created_at
        timestamp updated_at
    }

    CART_ITEMS {
        bigint id PK
        bigint user_id FK
        bigint book_id FK
        integer quantity
        timestamp created_at
        timestamp updated_at
    }

    ORDERS {
        bigint id PK
        string order_number UK
        bigint user_id FK
        decimal total_amount "12,2"
        decimal shipping_fee "12,2"
        string status "pending/verifying/processing/shipped/completed/cancelled"
        text shipping_address
        string recipient_name
        string recipient_phone
        string tracking_number
        string payment_method
        string payment_proof
        timestamp expired_at
        timestamp created_at
        timestamp updated_at
    }

    ORDER_ITEMS {
        bigint id PK
        bigint order_id FK
        bigint book_id FK
        integer quantity
        decimal unit_price "12,2"
        decimal subtotal "12,2"
        timestamp created_at
        timestamp updated_at
    }

    ORDER_STATUS_LOGS {
        bigint id PK
        bigint order_id FK
        string status
        text notes
        string updated_by
        timestamp created_at
        timestamp updated_at
    }