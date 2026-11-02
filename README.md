# UAS_DatabaseAdvance_Dani_Riyanto_25120500028
Detail Pengerjaan Tugas
# 🗄️ PostgreSQL Database Project Documentation

---

## 🛠️ Prasyarat (Requirements)

Sebelum memulai, pastikan perangkat Anda telah terinstal:
- **PostgreSQL**: pgAdmin
- **Klien Database**: `pgAdmin MongoDB Compass (jika hybrid)
- **Ekstensi PostgreSQL**: `pgcrypto`, `pg_trgm` (Opsional)

---

#Latar Belakang
SentraMart (PT Sentra Niaga Nusantara) adalah marketplace lokal yang sejak 2022 menghubungkan lebih dari 1.000 seller dengan sekitar 1,2 juta pengguna. Seluruh transaksi berjalan di satu server PostgreSQL dengan skema yang dirancang tim awal saat perusahaan masih startup.

Pada Flash Sale Harbolnas 12.12.2025 pukul 20.00–20.30 WIB, trafik melonjak belasan kali lipat. Akibatnya :
Produk flash sale terjual melebihi kuota (overselling)
Ratusan pelanggan ditagih dua kali (double charge)
Checkout keranjang gagal karena deadlock
Halaman penting aplikasi mengalami timeout.

Investigasi juga menemukan bahwa semua layanan memakai satu akun superuser, data pribadi pelanggan tersimpan sebagai teks biasa, dan backup disimpan di server yang sama.

Manajemen menargetkan volume order naik 3× dalam 18 bulan ke depan. CTO membentuk Database Reliability Task Force dan meminta rekomendasi yang dibuktikan dengan data dan implementasi, bukan asumsi. Detail insiden, dan skema

---

##Rumusan Masalah
Bagaimana merancang ulang database SentraMart agar:
Data transaksi tetap konsisten pada konkurensi tinggi
Query kritis memenuhi SLA
Data dengan struktur fleksibel dikelola dengan model data yang tepat
Arsitektur siap tumbuh 3× dengan keandalan dan keamanan yang memadai.

---

## 📁 Struktur Direktori

```text
.
├── sql/
│   ├── 01_audit.sql          # Skrip audit & profiling skema legacy
│   ├── 02_schema_v2.sql       # DDL pembuatan tabel, partisi, constraint, & indeks
│   ├── 03_migration.sql      # Skrip ETL / migrasi data dari v1 ke v2
│   ├── 04_optimization.sql   # Query optimasi & materialized views
│   ├── 05_transactions.sql   # Stored procedure/fungsi transaksi & perbaikan konkurensi
│   └── 06_security.sql       # Konfigurasi Role, RLS, & Masking PII
├── data/                     # File dataset (CSV/JSONL)
├── evidence/                 # Tangkapan layar bukti uji coba & output EXPLAIN
└── README.md

---

##ERD
[ categories ] 1 ------ N [ products ] N ------ 1 [ sellers ]
                             |      |
                    1        |      | 1
                    |        |      |
                    N        |      N
  [ product_attributes ]     |    [ flash_sale_items ] N ------ 1 [ flash_sale_events ]
                             |
                             N
                      [ order_items ] N ------ 1 [ orders ] 1 ------ 1 [ payments ]
                                                    |
                                                    N
                                                    |
                                                    1
                                              [ customers ]



---

###Code Relasi
-- 1. Tabel Customers
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    full_name VARCHAR(150) NOT NULL,
    email VARCHAR(150),
    phone VARCHAR(50),
    city VARCHAR(100),
    province VARCHAR(100),
    registered_at TIMESTAMP,
    tier VARCHAR(50)
);

-- 2. Tabel Sellers
CREATE TABLE sellers (
    seller_id INT PRIMARY KEY,
    seller_name VARCHAR(150) NOT NULL,
    seller_type VARCHAR(50),
    city VARCHAR(100),
    province VARCHAR(100),
    joined_at TIMESTAMP,
    rating NUMERIC(3, 2)
);

-- 3. Tabel Categories
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL
);

-- 4. Tabel Products
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    seller_id INT REFERENCES sellers(seller_id),
    category_id INT REFERENCES categories(category_id),
    product_name VARCHAR(255) NOT NULL,
    price NUMERIC(12, 2) NOT NULL,
    stock INT DEFAULT 0,
    is_flash_sale BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP
);

-- 5. Tabel Product Attributes
CREATE TABLE product_attributes (
    product_id INT REFERENCES products(product_id),
    attr_name VARCHAR(100) NOT NULL,
    attr_value VARCHAR(255) NOT NULL,
    PRIMARY KEY (product_id, attr_name)
);

-- 6. Tabel Flash Sale Events
CREATE TABLE flash_sale_events (
    event_id VARCHAR(50) PRIMARY KEY,
    event_name VARCHAR(150) NOT NULL,
    start_at TIMESTAMP NOT NULL,
    end_at TIMESTAMP NOT NULL
);

-- 7. Tabel Flash Sale Items
CREATE TABLE flash_sale_items (
    event_id VARCHAR(50) REFERENCES flash_sale_events(event_id),
    product_id INT REFERENCES products(product_id),
    flash_price NUMERIC(12, 2) NOT NULL,
    quota INT NOT NULL,
    PRIMARY KEY (event_id, product_id)
);

-- 8. Tabel Orders
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date TIMESTAMP NOT NULL,
    status VARCHAR(50) NOT NULL,
    shipping_fee NUMERIC(12, 2) DEFAULT 0,
    discount NUMERIC(12, 2) DEFAULT 0,
    total_amount NUMERIC(12, 2) NOT NULL,
    voucher_code VARCHAR(50),
    shipping_city VARCHAR(100)
);

-- 9. Tabel Order Items
CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    seller_id INT REFERENCES sellers(seller_id),
    qty INT NOT NULL,
    unit_price NUMERIC(12, 2) NOT NULL,
    subtotal NUMERIC(12, 2) NOT NULL
);

-- 10. Tabel Payments (Jika terpisah dari orders)
CREATE TABLE payments (
	payment_id INT PRIMARY KEY,
    order_id INT REFERENCES orders(order_id),
    method VARCHAR(50) NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    gateway_ref VARCHAR(100),
    created_at TIMESTAMP,
    paid_at TIMESTAMP
);


