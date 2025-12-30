# PROSEDUR, FUNCTION, & TRIGGER TANAMI 
## FUNCTIONS (5 buah)
1. [HitungTotalPesanan](#1-hitungtotalpesanan) - Hitung total pesanan dengan diskon
2. [ValidasiStokTersedia](#2-validasistokttersedia) - Cek stok tersedia untuk produk
3. [HitungDiskonKupon](#3-hitungdiskonkupon) - Hitung diskon dari kupon
4. [StatusPesananValid](#4-statuspesananvalid) - Validasi apakah status transition valid
5. [HitungTotalPendapatanPetani](#5-hitungtotalpendapatanpetani) - Total pendapatan petani

## TRIGGERS (6 buah)
1. [trg_log_perubahan_status](#trigger-1-trg_log_perubahan_status) - Log setiap perubahan status
2. [trg_hitung_subtotal_item](#trigger-2-trg_hitung_subtotal_item) - Auto hitung subtotal item
3. [trg_release_stok_batal](#trigger-3-trg_release_stok_batal) - Release stok saat batal
4. [trg_update_reserved_stok](#trigger-4-trg_update_reserved_stok) - Update reserved stok
5. [trg_validasi_stok_checkout](#trigger-5-trg_validasi_stok_checkout) - Validasi stok saat order
6. [trg_create_escrow_otomatis](#trigger-6-trg_create_escrow_otomatis) - Create escrow otomatis

## PROSEDUR (10 buah)
1. [DaftarPengguna](#1-daftar-pengguna) - Registrasi user baru
2. [TambahProduk](#2-tambah-produk) - Tambah produk baru
3. [LihatDetailProduk](#3-lihat-detail-produk) - Detail produk by ID
4. [TambahKeKeranjang](#4-tambah-ke-keranjang) - Tambah produk ke cart
5. [BuatPesanan](#5-buat-pesanan) - Create order dari cart
6. [VerifikasiPembayaran](#6-verifikasi-pembayaran) - Verifikasi bukti bayar
7. [UpdateStatusPesanan](#7-update-status-pesanan) - Update status order
8. [LihatDetailPesanan](#8-lihat-detail-pesanan) - Detail pesanan lengkap
9. [TolakPesanan](#9-tolak-pesanan) - Tolak pesanan dengan alasan
10. [KonfirmasiPesananSelesai](#10-konfirmasi-pesanan-selesai) - Pembeli konfirmasi selesai



# FUNCTION 
## 1. HITUNG TOTAL PESANAN

**Fungsi:** Menghitung total pesanan dengan diskon dan ongkir

```sql
DELIMITER $$

CREATE FUNCTION HitungTotalPesanan(
  p_subtotal DECIMAL(10,2),
  p_diskon DECIMAL(10,2),
  p_ongkir DECIMAL(10,2)
) 
RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
  DECLARE v_total DECIMAL(10,2);
  SET v_total = p_subtotal - p_diskon + p_ongkir;
  RETURN v_total;
END$$

DELIMITER ;
```

### Cara Menggunakan

```sql
-- Test Function
SELECT HitungTotalPesanan(100000, 10000, 20000) AS 'Total Pesanan';
-- Output: 110000

-- Gunakan di dalam query
SELECT 
  id_pesanan,
  subtotal,
  diskon,
  ongkir,
  HitungTotalPesanan(subtotal, diskon, ongkir) AS 'total_otomatis'
FROM pesanan;

-- Gunakan di dalam procedure
SET p_total_bayar = HitungTotalPesanan(v_subtotal, v_diskon, v_ongkir);
```

---

## 2. VALIDASI STOK TERSEDIA

**Fungsi:** Mengecek apakah stok tersedia cukup untuk pembelian

```sql
DELIMITER $$

CREATE FUNCTION ValidasiStokTersedia(
  p_product_id INT,
  p_jumlah_beli INT
)
RETURNS INT
DETERMINISTIC
READS SQL DATA
BEGIN
  DECLARE v_stok INT;
  DECLARE v_reserved INT;
  DECLARE v_tersedia INT;
  
  SELECT stok, stok_direserve INTO v_stok, v_reserved
  FROM produk
  WHERE id_produk = p_product_id;
  
  SET v_tersedia = v_stok - v_reserved;
  
  -- Return 1 jika cukup, 0 jika tidak cukup
  IF v_tersedia >= p_jumlah_beli THEN
    RETURN 1;
  ELSE
    RETURN 0;
  END IF;
END$$

DELIMITER ;
```

### Cara Menggunakan

```sql
-- Cek validasi stok
SELECT ValidasiStokTersedia(1, 5) AS 'Stok Cukup?';
-- Output: 1 (cukup) atau 0 (tidak cukup)

-- Gunakan di dalam IF statement di procedure
IF ValidasiStokTersedia(p_id_produk, p_jumlah) = 0 THEN
  SET p_message = 'Stok tidak cukup';
END IF;

-- Gunakan di dalam query filtering
SELECT p.id_produk, p.nama_produk
FROM produk p
WHERE ValidasiStokTersedia(p.id_produk, 2) = 1;
```

---

## 3. HITUNG DISKON KUPON

**Fungsi:** Menghitung besar diskon berdasarkan kode kupon

```sql
DELIMITER $$

CREATE FUNCTION HitungDiskonKupon(
  p_kode_kupon VARCHAR(20),
  p_subtotal DECIMAL(10,2)
)
RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
  DECLARE v_nominal_diskon DECIMAL(10,2) DEFAULT 0;
  DECLARE v_persen_diskon DECIMAL(5,2) DEFAULT 0;
  DECLARE v_tipe_diskon VARCHAR(20);
  DECLARE v_min_belanja DECIMAL(10,2);
  DECLARE v_kupon_id INT;
  
  -- Cari kupon
  SELECT id_kupon, tipe_diskon, nominal_diskon, persen_diskon, min_belanja
  INTO v_kupon_id, v_tipe_diskon, v_nominal_diskon, v_persen_diskon, v_min_belanja
  FROM kupon
  WHERE kode_kupon = p_kode_kupon
    AND is_aktif = 1
    AND tgl_mulai <= NOW()
    AND tgl_selesai >= NOW();
  
  -- Jika kupon tidak ditemukan, return 0
  IF v_kupon_id IS NULL THEN
    RETURN 0;
  END IF;
  
  -- Cek minimal belanja
  IF p_subtotal < v_min_belanja THEN
    RETURN 0;
  END IF;
  
  -- Hitung diskon
  IF v_tipe_diskon = 'nominal' THEN
    RETURN v_nominal_diskon;
  ELSE
    RETURN (p_subtotal * v_persen_diskon / 100);
  END IF;
END$$

DELIMITER ;
```

### Cara Menggunakan

```sql
-- Cek diskon kupon
SELECT HitungDiskonKupon('PROMO10', 100000) AS 'Diskon Rp';
-- Output: 10000 (jika kupon nominal) atau 20000 (jika kupon persen)

-- Gunakan di dalam procedure
SET v_diskon = HitungDiskonKupon(p_kode_kupon, v_subtotal);

-- Gunakan di dalam query
SELECT 
  id_pesanan,
  subtotal,
  HitungDiskonKupon('PROMO10', subtotal) AS 'diskon_kupon'
FROM pesanan;
```

---

## 4. STATUS PESANAN VALID

**Fungsi:** Validasi apakah status transition diperbolehkan

```sql
DELIMITER $$

CREATE FUNCTION StatusPesananValid(
  p_status_lama VARCHAR(50),
  p_status_baru VARCHAR(50)
)
RETURNS INT
DETERMINISTIC
BEGIN
  -- Status transition yang valid
  -- pending → menunggu_verifikasi → dibayar → diproses → dikirim → terkirim → selesai
  
  IF p_status_lama = 'pending' AND p_status_baru IN ('menunggu_verifikasi', 'dibatalkan') THEN
    RETURN 1;
  ELSEIF p_status_lama = 'menunggu_verifikasi' AND p_status_baru IN ('dibayar', 'pending') THEN
    RETURN 1;
  ELSEIF p_status_lama = 'dibayar' AND p_status_baru IN ('diproses', 'dibatalkan') THEN
    RETURN 1;
  ELSEIF p_status_lama = 'diproses' AND p_status_baru IN ('dikirim', 'dibatalkan') THEN
    RETURN 1;
  ELSEIF p_status_lama = 'dikirim' AND p_status_baru = 'terkirim' THEN
    RETURN 1;
  ELSEIF p_status_lama = 'terkirim' AND p_status_baru IN ('selesai', 'minta_refund') THEN
    RETURN 1;
  ELSEIF p_status_lama = 'minta_refund' AND p_status_baru = 'direfund' THEN
    RETURN 1;
  ELSE
    RETURN 0;
  END IF;
END$$

DELIMITER ;
```

### Cara Menggunakan

```sql
-- Validasi status transition
SELECT StatusPesananValid('pending', 'menunggu_verifikasi') AS 'Valid?';
-- Output: 1 (valid) atau 0 (tidak valid)

-- Gunakan di dalam procedure
IF StatusPesananValid(v_status_lama, p_status_baru) = 0 THEN
  SET p_message = 'Transisi status tidak valid';
END IF;
```

---

## 5. HITUNG TOTAL PENDAPATAN PETANI

**Fungsi:** Menghitung total pendapatan petani dari pesanan yang selesai

```sql
DELIMITER $$

CREATE FUNCTION HitungTotalPendapatanPetani(
  p_id_petani INT
)
RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
  DECLARE v_total_pendapatan DECIMAL(10,2) DEFAULT 0;
  
  SELECT COALESCE(SUM(ip.subtotal), 0) INTO v_total_pendapatan
  FROM item_pesanan ip
  JOIN pesanan p ON ip.id_pesanan = p.id_pesanan
  WHERE ip.id_petani = p_id_petani
    AND p.status_pesanan = 'selesai';
  
  RETURN v_total_pendapatan;
END$$

DELIMITER ;
```

### Cara Menggunakan

```sql
-- Cek total pendapatan petani
SELECT HitungTotalPendapatanPetani(2) AS 'Total Pendapatan Rp';
-- Output: 150000 (total dari semua pesanan selesai)

-- Gunakan untuk dashboard petani
SELECT 
  id_pengguna,
  nama_lengkap,
  HitungTotalPendapatanPetani(id_pengguna) AS 'total_pendapatan'
FROM pengguna
WHERE role_pengguna = 'petani';
```

---

# PROSEDUR
## 1. DAFTAR PENGGUNA

**Fungsi:** Registrasi user baru (admin, petani, atau pembeli)

```sql
DELIMITER $$

CREATE PROCEDURE DaftarPengguna(
  IN p_nama_lengkap VARCHAR(100),
  IN p_email VARCHAR(100),
  IN p_password VARCHAR(255),
  IN p_role_pengguna ENUM('admin','petani','pembeli'),
  IN p_alamat TEXT,
  IN p_no_hp VARCHAR(20),
  OUT p_user_id INT,
  OUT p_message VARCHAR(100)
)
BEGIN
  INSERT INTO pengguna (nama_lengkap, email, password, role_pengguna, alamat, no_hp, is_verified)
  VALUES (p_nama_lengkap, p_email, p_password, p_role_pengguna, p_alamat, p_no_hp, 0);
  
  SET p_user_id = LAST_INSERT_ID();
  SET p_message = 'Pengguna berhasil terdaftar';
END$$

DELIMITER ;
```
### Cara Memanggil

```sql
CALL DaftarPengguna(
  'Budi Santoso',                  -- p_nama_lengkap
  'budi@pembeli.com',              -- p_email
  '$2y$10$hashedpassword',         -- p_password (bcrypt)
  'pembeli',                       -- p_role_pengguna
  'Jakarta Selatan',               -- p_alamat
  '081234567890',                  -- p_no_hp
  @user_id,                        -- output user_id
  @message                         -- output message
);

SELECT @user_id AS 'ID User', @message AS 'Pesan';
```

### Output

```
ID User | Pesan
--------|---------------------------
7       | Pengguna berhasil terdaftar
```

---

## 2. TAMBAH PRODUK

**Fungsi:** Menambah produk baru (hanya petani)

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE TambahProduk(
  IN p_id_petani INT,
  IN p_id_kategori INT,
  IN p_nama_produk VARCHAR(100),
  IN p_slug_produk VARCHAR(100),
  IN p_harga DECIMAL(10,2),
  IN p_stok INT,
  IN p_satuan VARCHAR(20),
  IN p_deskripsi TEXT,
  OUT p_product_id INT,
  OUT p_message VARCHAR(100)
)
BEGIN
  INSERT INTO produk 
    (id_petani, id_kategori, nama_produk, slug_produk, harga, stok, stok_direserve, satuan, deskripsi, is_aktif)
  VALUES 
    (p_id_petani, p_id_kategori, p_nama_produk, p_slug_produk, p_harga, p_stok, 0, p_satuan, p_deskripsi, 1);
  
  SET p_product_id = LAST_INSERT_ID();
  SET p_message = 'Produk berhasil ditambahkan';
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL TambahProduk(
  2,                    -- p_id_petani (Pak Tono)
  1,                    -- p_id_kategori (Sayuran)
  'Kangkung Segar',     -- p_nama_produk
  'kangkung-segar',     -- p_slug_produk
  6000.00,              -- p_harga
  50,                   -- p_stok
  'kg',                 -- p_satuan
  'Kangkung segar organik dari Bogor', -- p_deskripsi
  @product_id,          -- output product_id
  @message              -- output message
);

SELECT @product_id AS 'ID Produk', @message AS 'Pesan';
```

### Output

```
ID Produk | Pesan
----------|---------------------------
7         | Produk berhasil ditambahkan
```

---

## 3. LIHAT DETAIL PRODUK

**Fungsi:** Menampilkan detail produk berdasarkan ID

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE LihatDetailProduk(
  IN p_product_id INT
)
BEGIN
  SELECT 
    p.id_produk,
    p.nama_produk,
    p.slug_produk,
    p.harga,
    p.stok,
    (p.stok - p.stok_direserve) AS 'stok_tersedia',
    p.satuan,
    p.deskripsi,
    u.nama_lengkap AS 'nama_petani',
    u.no_hp AS 'hp_petani',
    k.nama_kategori,
    p.tgl_dibuat
  FROM produk p
  JOIN pengguna u ON p.id_petani = u.id_pengguna
  JOIN kategori k ON p.id_kategori = k.id_kategori
  WHERE p.id_produk = p_product_id
    AND p.is_aktif = 1;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL LihatDetailProduk(1);  -- Lihat detail produk ID 1
```

### Output

```
id_produk | nama_produk | harga | stok | stok_tersedia | satuan | nama_petani | nama_kategori
----------|-------------|-------|------|---------------|--------|-------------|---------------
1         | Wortel Organik | 10000 | 50 | 50 | kg | Pak Tono | Sayuran
```

---

## 4. TAMBAH KE KERANJANG

**Fungsi:** Menambah produk ke keranjang belanja

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE TambahKeKeranjang(
  IN p_id_pengguna INT,
  IN p_id_produk INT,
  IN p_jumlah INT,
  OUT p_message VARCHAR(100)
)
BEGIN
  DECLARE v_stok_tersedia INT;
  
  -- Cek stok tersedia
  SELECT (stok - stok_direserve) INTO v_stok_tersedia
  FROM produk
  WHERE id_produk = p_id_produk;
  
  IF v_stok_tersedia < p_jumlah THEN
    SET p_message = 'Stok tidak cukup';
  ELSE
    -- Update or Insert ke keranjang
    INSERT INTO keranjang (id_pengguna, id_produk, jumlah)
    VALUES (p_id_pengguna, p_id_produk, p_jumlah)
    ON DUPLICATE KEY UPDATE jumlah = jumlah + p_jumlah;
    
    SET p_message = 'Produk berhasil ditambahkan ke keranjang';
  END IF;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL TambahKeKeranjang(
  4,              -- p_id_pengguna (Budi)
  1,              -- p_id_produk (Wortel)
  2,              -- p_jumlah
  @message        -- output message
);

SELECT @message AS 'Pesan';
```

### Output

```
Pesan
---------------------------
Produk berhasil ditambahkan ke keranjang
```

---

## 5. BUAT PESANAN

**Fungsi:** Membuat pesanan dari keranjang dengan auto-generate batas bayar

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE BuatPesanan(
  IN p_id_pembeli INT,
  IN p_id_kota_tujuan INT,
  IN p_kode_kupon VARCHAR(20),
  OUT p_order_id INT,
  OUT p_total_bayar DECIMAL(10,2),
  OUT p_batas_bayar TIMESTAMP,
  OUT p_message VARCHAR(100)
)
BEGIN
  DECLARE v_subtotal DECIMAL(10,2) DEFAULT 0;
  DECLARE v_ongkir DECIMAL(10,2) DEFAULT 0;
  DECLARE v_diskon DECIMAL(10,2) DEFAULT 0;
  DECLARE v_kupon_id INT;
  DECLARE v_tipe_diskon VARCHAR(20);
  DECLARE v_nominal_diskon DECIMAL(10,2);
  DECLARE v_persen_diskon DECIMAL(5,2);
  DECLARE v_min_belanja DECIMAL(10,2);
  DECLARE v_cart_count INT;
  
  -- Validasi keranjang tidak kosong
  SELECT COUNT(*) INTO v_cart_count FROM keranjang WHERE id_pengguna = p_id_pembeli;
  
  IF v_cart_count = 0 THEN
    SET p_message = 'Keranjang kosong';
  ELSE
    -- Hitung subtotal dari keranjang
    SELECT COALESCE(SUM(k.jumlah * p.harga), 0) INTO v_subtotal
    FROM keranjang k
    JOIN produk p ON k.id_produk = p.id_produk
    WHERE k.id_pengguna = p_id_pembeli;
    
    -- Ambil ongkir dari kota tujuan
    SELECT ongkir INTO v_ongkir
    FROM kota
    WHERE id_kota = p_id_kota_tujuan;
    
    -- Validasi kupon jika ada
    IF p_kode_kupon IS NOT NULL THEN
      SELECT id_kupon, tipe_diskon, nominal_diskon, persen_diskon, min_belanja
      INTO v_kupon_id, v_tipe_diskon, v_nominal_diskon, v_persen_diskon, v_min_belanja
      FROM kupon
      WHERE kode_kupon = p_kode_kupon
        AND is_aktif = 1
        AND tgl_mulai <= NOW()
        AND tgl_selesai >= NOW();
      
      IF v_kupon_id IS NOT NULL AND v_subtotal >= v_min_belanja THEN
        IF v_tipe_diskon = 'nominal' THEN
          SET v_diskon = v_nominal_diskon;
        ELSE
          SET v_diskon = (v_subtotal * v_persen_diskon / 100);
        END IF;
      END IF;
    END IF;
    
    -- Hitung total bayar
    SET p_total_bayar = v_subtotal - v_diskon + v_ongkir;
    SET p_batas_bayar = DATE_ADD(NOW(), INTERVAL 24 HOUR);
    
    -- Create pesanan
    INSERT INTO pesanan 
      (id_pembeli, id_kota_tujuan, subtotal, ongkir, diskon, total_bayar, status_pesanan, batas_bayar)
    VALUES 
      (p_id_pembeli, p_id_kota_tujuan, v_subtotal, v_ongkir, v_diskon, p_total_bayar, 'pending', p_batas_bayar);
    
    SET p_order_id = LAST_INSERT_ID();
    
    -- Insert item pesanan dari keranjang
    INSERT INTO item_pesanan (id_pesanan, id_produk, id_petani, jumlah, harga_snapshot, subtotal)
    SELECT p_order_id, k.id_produk, p.id_petani, k.jumlah, p.harga, (k.jumlah * p.harga)
    FROM keranjang k
    JOIN produk p ON k.id_produk = p.id_produk
    WHERE k.id_pengguna = p_id_pembeli;
    
    -- Reserve stok
    UPDATE produk p
    JOIN item_pesanan ip ON p.id_produk = ip.id_produk
    SET p.stok_direserve = p.stok_direserve + ip.jumlah
    WHERE ip.id_pesanan = p_order_id;
    
    -- Record pemakaian kupon
    IF v_kupon_id IS NOT NULL THEN
      INSERT INTO pemakaian_kupon (id_kupon, id_pengguna, id_pesanan, diskon_dipakai)
      VALUES (v_kupon_id, p_id_pembeli, p_order_id, v_diskon);
    END IF;
    
    -- Create escrow
    INSERT INTO escrow (id_pesanan, jumlah, status_escrow, tgl_ditahan)
    VALUES (p_order_id, p_total_bayar, 'ditahan', NOW());
    
    -- Hapus keranjang
    DELETE FROM keranjang WHERE id_pengguna = p_id_pembeli;
    
    SET p_message = 'Pesanan berhasil dibuat';
  END IF;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL BuatPesanan(
  4,              -- p_id_pembeli (Budi)
  1,              -- p_id_kota_tujuan (Jakarta)
  'PROMO10',      -- p_kode_kupon (opsional, bisa NULL)
  @order_id,      -- output order_id
  @total_bayar,   -- output total_bayar
  @batas_bayar,   -- output batas_bayar
  @message        -- output message
);

SELECT @order_id AS 'ID Pesanan', 
       @total_bayar AS 'Total Bayar Rp', 
       @batas_bayar AS 'Batas Bayar',
       @message AS 'Pesan';
```

### Output

```
ID Pesanan | Total Bayar Rp | Batas Bayar         | Pesan
-----------|----------------|---------------------|---------------------------
1          | 59000          | 2025-12-31 12:58:18 | Pesanan berhasil dibuat
```

---

## 6. VERIFIKASI PEMBAYARAN

**Fungsi:** Admin/petani verifikasi bukti pembayaran

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE VerifikasiPembayaran(
  IN p_id_pesanan INT,
  IN p_id_verifikator INT,
  IN p_approve TINYINT,
  IN p_alasan_tolak TEXT,
  OUT p_message VARCHAR(100)
)
BEGIN
  DECLARE v_status_saat_ini VARCHAR(50);
  
  -- Ambil status saat ini
  SELECT status_pesanan INTO v_status_saat_ini
  FROM pesanan
  WHERE id_pesanan = p_id_pesanan;
  
  IF v_status_saat_ini != 'menunggu_verifikasi' THEN
    SET p_message = 'Status pesanan tidak sesuai untuk diverifikasi';
  ELSE
    IF p_approve = 1 THEN
      -- APPROVE pembayaran
      UPDATE pesanan
      SET status_pesanan = 'dibayar',
          tgl_verifikasi = NOW(),
          id_verifikator = p_id_verifikator
      WHERE id_pesanan = p_id_pesanan;
      
      SET p_message = 'Pembayaran disetujui, pesanan diproses';
    ELSE
      -- REJECT pembayaran
      UPDATE pesanan
      SET status_pesanan = 'pending',
          alasan_tolak = p_alasan_tolak
      WHERE id_pesanan = p_id_pesanan;
      
      -- Release reserved stok
      UPDATE produk p
      JOIN item_pesanan ip ON p.id_produk = ip.id_produk
      SET p.stok_direserve = p.stok_direserve - ip.jumlah
      WHERE ip.id_pesanan = p_id_pesanan;
      
      SET p_message = 'Pembayaran ditolak, stok dirilis';
    END IF;
  END IF;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL VerifikasiPembayaran(
  1,              -- p_id_pesanan
  1,              -- p_id_verifikator (Admin)
  1,              -- p_approve (1=terima, 0=tolak)
  NULL,           -- p_alasan_tolak
  @message        -- output message
);

SELECT @message AS 'Pesan';
```

### Output

```
Pesan
---------------------------
Pembayaran disetujui, pesanan diproses
```

---

## 7. UPDATE STATUS PESANAN

**Fungsi:** Update status pesanan (pending → dibayar → diproses → dikirim → terkirim → selesai)

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE UpdateStatusPesanan(
  IN p_id_pesanan INT,
  IN p_status_baru VARCHAR(50),
  IN p_no_resi VARCHAR(100),
  IN p_catatan TEXT,
  OUT p_message VARCHAR(100)
)
BEGIN
  UPDATE pesanan
  SET status_pesanan = p_status_baru,
      no_resi = IFNULL(p_no_resi, no_resi),
      catatan = IFNULL(p_catatan, catatan),
      tgl_update = NOW()
  WHERE id_pesanan = p_id_pesanan;
  
  SET p_message = 'Status pesanan berhasil diupdate';
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL UpdateStatusPesanan(
  1,                      -- p_id_pesanan
  'dikirim',              -- p_status_baru
  'JNE-123456789',        -- p_no_resi
  'Sedang dalam perjalanan', -- p_catatan
  @message                -- output message
);

SELECT @message AS 'Pesan';
```

### Output

```
Pesan
---------------------------
Status pesanan berhasil diupdate
```

---

## 8. LIHAT DETAIL PESANAN

**Fungsi:** Menampilkan detail pesanan lengkap dengan item dan info pembeli

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE LihatDetailPesanan(
  IN p_id_pesanan INT
)
BEGIN
  -- Header pesanan
  SELECT 
    p.id_pesanan,
    p.status_pesanan,
    u.nama_lengkap AS 'nama_pembeli',
    u.email,
    u.no_hp,
    k.nama_kota,
    p.subtotal,
    p.ongkir,
    p.diskon,
    p.total_bayar,
    p.batas_bayar,
    p.tgl_dibuat,
    p.tgl_verifikasi,
    p.no_resi,
    p.catatan
  FROM pesanan p
  JOIN pengguna u ON p.id_pembeli = u.id_pengguna
  JOIN kota k ON p.id_kota_tujuan = k.id_kota
  WHERE p.id_pesanan = p_id_pesanan;
  
  -- Detail item pesanan
  SELECT 
    ip.id_item,
    ip.id_produk,
    pr.nama_produk,
    pt.nama_lengkap AS 'nama_petani',
    ip.jumlah,
    ip.harga_snapshot,
    ip.subtotal
  FROM item_pesanan ip
  JOIN produk pr ON ip.id_produk = pr.id_produk
  JOIN pengguna pt ON ip.id_petani = pt.id_pengguna
  WHERE ip.id_pesanan = p_id_pesanan;
  
  -- Escrow info
  SELECT 
    id_escrow,
    jumlah,
    status_escrow,
    tgl_ditahan,
    tgl_kirim
  FROM escrow
  WHERE id_pesanan = p_id_pesanan;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL LihatDetailPesanan(1);
```

### Output

```
HEADER:
id_pesanan | status_pesanan | nama_pembeli | total_bayar | tgl_dibuat
-----------|----------------|--------------|-------------|-------------------
1          | dibayar        | Budi Santoso | 59000       | 2025-12-30 12:58:18

ITEMS:
id_item | nama_produk | nama_petani | jumlah | harga_snapshot | subtotal
--------|-------------|-------------|--------|----------------|----------
1       | Wortel Organik | Pak Tono | 2 | 10000 | 20000

ESCROW:
id_escrow | jumlah | status_escrow | tgl_ditahan
----------|--------|---------------|-------------------
1         | 59000  | ditahan       | 2025-12-30 12:58:18
```

---

## 9. TOLAK PESANAN

**Fungsi:** Tolak pesanan dengan alasan dan release stok

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE TolakPesanan(
  IN p_id_pesanan INT,
  IN p_alasan_batal TEXT,
  OUT p_message VARCHAR(100)
)
BEGIN
  DECLARE v_status_saat_ini VARCHAR(50);
  
  SELECT status_pesanan INTO v_status_saat_ini
  FROM pesanan
  WHERE id_pesanan = p_id_pesanan;
  
  IF v_status_saat_ini IN ('selesai', 'terkirim', 'dikirim') THEN
    SET p_message = 'Pesanan tidak bisa dibatalkan pada status ini';
  ELSE
    -- Update pesanan
    UPDATE pesanan
    SET status_pesanan = 'dibatalkan',
        alasan_batal = p_alasan_batal,
        tgl_dibatalkan = NOW(),
        tgl_update = NOW()
    WHERE id_pesanan = p_id_pesanan;
    
    -- Release reserved stok
    UPDATE produk p
    JOIN item_pesanan ip ON p.id_produk = ip.id_produk
    SET p.stok_direserve = p.stok_direserve - ip.jumlah
    WHERE ip.id_pesanan = p_id_pesanan;
    
    -- Update escrow ke refund
    UPDATE escrow
    SET status_escrow = 'direfund_ke_pembeli',
        tgl_kirim = NOW()
    WHERE id_pesanan = p_id_pesanan;
    
    SET p_message = 'Pesanan dibatalkan, stok dirilis, escrow siap refund';
  END IF;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL TolakPesanan(
  1,                              -- p_id_pesanan
  'Produk tidak sesuai pesanan',  -- p_alasan_batal
  @message                        -- output message
);

SELECT @message AS 'Pesan';
```

### Output

```
Pesan
---------------------------
Pesanan dibatalkan, stok dirilis, escrow siap refund
```

---

## 10. KONFIRMASI PESANAN SELESAI

**Fungsi:** Pembeli konfirmasi pesanan sudah diterima dan selesai

### Syntax Procedure

```sql
DELIMITER $$

CREATE PROCEDURE KonfirmasiPesananSelesai(
  IN p_id_pesanan INT,
  IN p_id_pembeli INT,
  OUT p_message VARCHAR(100)
)
BEGIN
  DECLARE v_status_saat_ini VARCHAR(50);
  
  SELECT status_pesanan INTO v_status_saat_ini
  FROM pesanan
  WHERE id_pesanan = p_id_pesanan
    AND id_pembeli = p_id_pembeli;
  
  IF v_status_saat_ini IS NULL THEN
    SET p_message = 'Pesanan tidak ditemukan atau bukan milik Anda';
  ELSE IF v_status_saat_ini != 'terkirim' THEN
    SET p_message = 'Pesanan harus berstatus "terkirim" untuk dikonfirmasi selesai';
  ELSE
    -- Update pesanan
    UPDATE pesanan
    SET status_pesanan = 'selesai',
        tgl_selesai = NOW(),
        id_konfirmasi = p_id_pembeli,
        tgl_update = NOW()
    WHERE id_pesanan = p_id_pesanan;
    
    -- Release escrow ke petani
    UPDATE escrow e
    JOIN item_pesanan ip ON ip.id_pesanan = e.id_pesanan
    SET e.status_escrow = 'dikirim_ke_petani',
        e.tgl_kirim = NOW(),
        e.id_penerima = ip.id_petani,
        e.catatan = 'Dana dikirim ke petani setelah konfirmasi pembeli'
    WHERE e.id_pesanan = p_id_pesanan
    GROUP BY e.id_pesanan;
    
    SET p_message = 'Pesanan dikonfirmasi selesai, dana dikirim ke petani';
  END IF;
  END IF;
END$$

DELIMITER ;
```

### Cara Memanggil

```sql
CALL KonfirmasiPesananSelesai(
  1,              -- p_id_pesanan
  4,              -- p_id_pembeli (Budi)
  @message        -- output message
);

SELECT @message AS 'Pesan';
```

### Output

```
Pesan
---------------------------
Pesanan dikonfirmasi selesai, dana dikirim ke petani
```



# TRIGGERS
## TRIGGER 1: trg_log_perubahan_status

**Fungsi:** Otomatis mencatat setiap perubahan status pesanan ke tabel `histori_status`

```sql
DELIMITER $$

CREATE TRIGGER trg_log_perubahan_status
AFTER UPDATE ON pesanan
FOR EACH ROW
BEGIN
  IF OLD.status_pesanan != NEW.status_pesanan THEN
    INSERT INTO histori_status 
      (id_pesanan, status_lama, status_baru, id_pengubah, alasan, tgl_dibuat)
    VALUES 
      (NEW.id_pesanan, OLD.status_pesanan, NEW.status_pesanan, NEW.id_verifikator, NEW.alasan_batal, NOW());
  END IF;
END$$

DELIMITER ;
```

### Testing

```sql
-- Update status pesanan
UPDATE pesanan 
SET status_pesanan = 'dibayar', id_verifikator = 1 
WHERE id_pesanan = 1;

-- Cek histori
SELECT * FROM histori_status WHERE id_pesanan = 1;
-- Output: Akan otomatis tercatat perubahan dari pending → dibayar
```

## TRIGGER 2: trg_hitung_subtotal_item

**Fungsi:** Otomatis menghitung subtotal item pesanan saat insert

```sql
DELIMITER $$

CREATE TRIGGER trg_hitung_subtotal_item
BEFORE INSERT ON item_pesanan
FOR EACH ROW
BEGIN
  SET NEW.subtotal = NEW.jumlah * NEW.harga_snapshot;
END$$

DELIMITER ;
```

### Testing

```sql
-- Insert item pesanan
INSERT INTO item_pesanan 
  (id_pesanan, id_produk, id_petani, jumlah, harga_snapshot)
VALUES 
  (1, 1, 2, 5, 10000);

-- Cek hasil
SELECT * FROM item_pesanan WHERE id_pesanan = 1;
-- Output: subtotal otomatis 50000 (5 * 10000)
```

---

## TRIGGER 3: trg_release_stok_batal

**Fungsi:** Otomatis release stok yang direserve saat pesanan dibatalkan

```sql
DELIMITER $$

CREATE TRIGGER trg_release_stok_batal
AFTER UPDATE ON pesanan
FOR EACH ROW
BEGIN
  -- Jika status berubah ke dibatalkan, release stok
  IF NEW.status_pesanan = 'dibatalkan' AND OLD.status_pesanan != 'dibatalkan' THEN
    UPDATE produk p
    JOIN item_pesanan ip ON p.id_produk = ip.id_produk
    SET p.stok_direserve = p.stok_direserve - ip.jumlah
    WHERE ip.id_pesanan = NEW.id_pesanan;
  END IF;
END$$

DELIMITER ;
```

### Testing

```sql
-- Batalkan pesanan
UPDATE pesanan 
SET status_pesanan = 'dibatalkan', alasan_batal = 'Pembeli request' 
WHERE id_pesanan = 1;

-- Cek stok direserve berkurang otomatis
SELECT id_produk, stok, stok_direserve FROM produk WHERE id_produk = 1;
-- Output: stok_direserve berkurang otomatis
```

---

## TRIGGER 4: trg_update_reserved_stok

**Fungsi:** Otomatis update reserved stok saat item pesanan ditambahkan

```sql
DELIMITER $$

CREATE TRIGGER trg_update_reserved_stok
AFTER INSERT ON item_pesanan
FOR EACH ROW
BEGIN
  UPDATE produk
  SET stok_direserve = stok_direserve + NEW.jumlah
  WHERE id_produk = NEW.id_produk;
END$$

DELIMITER ;
```

### Testing

```sql
-- Insert item baru
INSERT INTO item_pesanan 
  (id_pesanan, id_produk, id_petani, jumlah, harga_snapshot)
VALUES 
  (1, 1, 2, 3, 10000);

-- Cek stok direserve
SELECT stok_direserve FROM produk WHERE id_produk = 1;
-- Output: stok_direserve naik otomatis 3
```

---

## TRIGGER 5: trg_validasi_stok_checkout

**Fungsi:** Validasi stok sebelum membuat pesanan (insert ke item_pesanan)

```sql
DELIMITER $$

CREATE TRIGGER trg_validasi_stok_checkout
BEFORE INSERT ON item_pesanan
FOR EACH ROW
BEGIN
  DECLARE v_stok INT;
  DECLARE v_reserved INT;
  DECLARE v_tersedia INT;
  
  SELECT stok, stok_direserve INTO v_stok, v_reserved
  FROM produk
  WHERE id_produk = NEW.id_produk;
  
  SET v_tersedia = v_stok - v_reserved;
  
  IF v_tersedia < NEW.jumlah THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Stok tidak cukup untuk produk ini';
  END IF;
END$$

DELIMITER ;
```

### Testing

```sql
-- Coba insert item dengan stok lebih dari tersedia
-- Akan error otomatis
INSERT INTO item_pesanan 
  (id_pesanan, id_produk, id_petani, jumlah, harga_snapshot)
VALUES 
  (1, 1, 2, 1000, 10000);
-- Output: ERROR - Stok tidak cukup untuk produk ini
```

---

## TRIGGER 6: trg_create_escrow_otomatis

**Fungsi:** Otomatis create escrow record saat pesanan dibuat

```sql
DELIMITER $$

CREATE TRIGGER trg_create_escrow_otomatis
AFTER INSERT ON pesanan
FOR EACH ROW
BEGIN
  INSERT INTO escrow 
    (id_pesanan, jumlah, status_escrow, tgl_ditahan)
  VALUES 
    (NEW.id_pesanan, NEW.total_bayar, 'ditahan', NOW());
END$$

DELIMITER ;
```

### Testing

```sql
-- Buat pesanan baru
INSERT INTO pesanan 
  (id_pembeli, id_kota_tujuan, subtotal, ongkir, diskon, total_bayar, status_pesanan, batas_bayar)
VALUES 
  (4, 1, 50000, 20000, 5000, 65000, 'pending', NOW() + INTERVAL 24 HOUR);

-- Cek escrow otomatis tercreate
SELECT * FROM escrow WHERE id_pesanan = LAST_INSERT_ID();
-- Output: Escrow otomatis dibuat
```

