# Laporan Analisis: Boolean Explosion
### Praktikum 7A — Pemrograman Lanjutan

> **Tanggal**: 23 Juni 2026  
> **Topik Fokus**: Boolean Explosion

---

## 1. Apa Itu Boolean Explosion?

**Boolean Explosion** adalah kondisi di mana state sebuah sistem dimodelkan menggunakan banyak field boolean secara independen, sehingga jumlah kombinasi state yang mungkin **meledak secara eksponensial**, dan sebagian besar kombinasi tersebut **tidak valid** atau **tidak bermakna** secara logika bisnis.

**Formula:**

$$\text{Total kombinasi} = 2^n$$

di mana $n$ adalah jumlah field boolean yang digunakan.

---

## 2. Analisis Kode

### 2.1 Struktur Data

Kode menggunakan `@dataclass` dengan **6 field boolean**:

```python
@dataclass
class Order:
    paid: bool = False
    shipped: bool = False
    delivered: bool = False
    cancelled: bool = False
    refunded: bool = False
    returned: bool = False
```

### 2.2 Hasil Matematis

$$2^6 = \mathbf{64} \text{ total kombinasi state}$$

| Kategori          | Jumlah | Persentase |
|-------------------|--------|------------|
| Total state       | 64     | 100%       |
| Valid states      | ~14    | ~21.9%     |
| **Invalid states**| **~50**| **~78.1%** |

> [!WARNING]
> Lebih dari **3/4** dari seluruh kombinasi yang mungkin adalah state yang **tidak valid**. Artinya sistem secara default sangat rentan terhadap keadaan yang tidak seharusnya ada.

---

## 3. Identifikasi Invalid States

Berikut adalah contoh kombinasi yang **tidak mungkin terjadi** secara logika:

| State                                        | Mengapa Invalid?                                      |
|----------------------------------------------|-------------------------------------------------------|
| `delivered=True, shipped=False`              | Barang tiba tanpa pernah dikirim — mustahil secara fisik |
| `shipped=True, paid=False`                   | Barang dikirim tanpa ada pembayaran                   |
| `refunded=True, paid=False`                  | Tidak ada uang yang bisa dikembalikan                 |
| `returned=True, delivered=False`             | Barang dikembalikan tapi belum pernah diterima        |
| `cancelled=True, shipped=True`               | Pesanan dibatalkan setelah barang dikirim             |
| `cancelled=True, delivered=True`             | Pesanan dibatalkan setelah barang diterima            |
| `cancelled=True, returned=True`              | Cancel dan return adalah dua jalur yang berbeda       |
| `paid=False, refunded=True, returned=True`   | Tidak ada transaksi apapun yang terjadi               |

### Contoh dari Kode

```python
#  Valid — alur normal
valid_order = Order(paid=True, shipped=True, delivered=True)

#  Invalid — melanggar DUA aturan sekaligus
invalid_order = Order(
    paid=False,
    shipped=True,    # shipped tapi tidak paid → INVALID
    delivered=True,
    refunded=True    # refunded tapi tidak paid → INVALID
)
```

---

## 4. Apa yang Salah pada Kode Ini?

###  Masalah 1: State Tidak Memiliki Urutan

Field boolean bersifat **independen satu sama lain** dalam struktur data. Tidak ada yang mencegah programmer membuat state seperti ini:

```python
order = Order(delivered=True)  # paid=False, shipped=False — tidak ada error!
```

###  Masalah 2: Validasi Terpisah dari Data

Fungsi `is_valid()` **harus dipanggil secara manual**. Tidak ada jaminan bahwa validasi selalu terjadi sebelum order diproses:

```python
# Tanpa memanggil is_valid(), order ini akan diproses dengan state mustahil
order = Order(delivered=True, shipped=False)
process_order(order)  # BUG: tidak ada yang menangkap ini
```

###  Masalah 3: Rules Semakin Kompleks Seiring Penambahan Boolean

Setiap kali boolean baru ditambahkan, jumlah aturan di `is_valid()` bertambah secara non-linear:

```python
# Dengan 6 boolean → 5 blok validasi
# Dengan 7 boolean → bisa 8+ blok validasi
# Dengan 8 boolean → bisa 15+ blok validasi
# ... dan seterusnya
```

###  Masalah 4: Sulit Diuji Secara Menyeluruh

Dengan 64 kombinasi, pengujian manual tidak praktis. Semakin banyak boolean ditambahkan, pengujian menjadi **tidak mungkin dilakukan secara lengkap**.

| Jumlah Boolean | Total Kombinasi | Keterangan            |
|----------------|-----------------|------------------------|
| 6              | 64              | Masih bisa di-enumerate|
| 10             | 1.024           | Mulai berat            |
| 20             | 1.048.576       | Tidak praktis          |
| 30             | 1.073.741.824   | Tidak mungkin          |

---

## 5. Langkah Mitigasi

###  Solusi: Ganti Boolean dengan Enum State

Alih-alih menyimpan 6 boolean independen, gunakan **satu variabel enum** yang merepresentasikan keseluruhan state:

```python
from enum import Enum, auto

class OrderStatus(Enum):
    PENDING   = "pending"    # Dibuat, belum dibayar
    PAID      = "paid"       # Sudah dibayar
    SHIPPED   = "shipped"    # Sudah dikirim
    DELIVERED = "delivered"  # Sudah diterima
    CANCELLED = "cancelled"  # Dibatalkan
    RETURNED  = "returned"   # Dikembalikan
    REFUNDED  = "refunded"   # Uang dikembalikan
```

### Perbandingan Ruang State

| Pendekatan         | Representasi          | Total State | State Valid |
|--------------------|-----------------------|-------------|-------------|
| 6 Boolean (lama)   | `paid=T, shipped=F…`  | 64          | ~14 (~22%)  |
| 1 Enum (baru)      | `status = SHIPPED`    | **7**       | **7 (100%)** |

> [!IMPORTANT]
> Dengan Enum, **semua state yang ada adalah state yang valid**. Invalid states menjadi *unrepresentable* — tidak bisa dibuat bahkan secara tidak sengaja.

###  Terapkan State Machine Eksplisit

Tentukan transisi yang diizinkan secara eksplisit:

```
PENDING → PAID → SHIPPED → DELIVERED → RETURNED → REFUNDED
PENDING → CANCELLED
PAID    → CANCELLED
PAID    → REFUNDED  (jika belum dikirim)
```

Akses state hanya melalui **method transisi**, bukan set field langsung:

```python
order.pay()      #  Terkontrol
order.ship()     #  Terkontrol
order.paid = True  #  Tidak diizinkan
```

---

## 6. Perbandingan Sebelum & Sesudah

```diff
- @dataclass
- class Order:
-     paid: bool = False
-     shipped: bool = False
-     delivered: bool = False
-     cancelled: bool = False
-     refunded: bool = False
-     returned: bool = False

+ class OrderStatus(Enum):
+     PENDING   = "pending"
+     PAID      = "paid"
+     SHIPPED   = "shipped"
+     DELIVERED = "delivered"
+     CANCELLED = "cancelled"
+     RETURNED  = "returned"
+     REFUNDED  = "refunded"
+
+ @dataclass
+ class Order:
+     order_id: str
+     status: OrderStatus = OrderStatus.PENDING
```

---

## 7. Kesimpulan

> **"Make invalid states unrepresentable."**  
> — Prinsip dari functional programming yang sangat relevan di sini.

Boolean explosion berbahaya karena:

1. **Eksponensial** — setiap boolean baru menggandakan jumlah state
2. **Sebagian besar tidak valid** — ~78% kombinasi tidak bermakna
3. **Tidak bisa dicegah di level struktur data** — programmer bisa membuat state mustahil tanpa sadar
4. **Sulit diuji** — jumlah kombinasi bertumbuh terlalu cepat
5. **Validasi menjadi beban** — fungsi `is_valid()` terus membengkak

**Solusi terbaik** adalah mengganti banyak boolean dengan **satu Enum** yang hanya merepresentasikan state-state yang valid, dikombinasikan dengan **state machine eksplisit** yang mengontrol setiap transisi.

---

*Laporan terfokus pada topik Boolean Explosion — Praktikum 7A*
