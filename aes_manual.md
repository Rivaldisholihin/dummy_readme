# README – AES-128 Manual Implementation

## 1. Project Overview

File `aes_manual.py` merupakan implementasi algoritma **Advanced Encryption Standard (AES) 128-bit** yang dibuat secara manual menggunakan bahasa pemrograman Python.

Program ini tidak menggunakan library AES eksternal seperti `PyCryptodome`. Sebaliknya, seluruh proses utama AES dibuat sendiri, mulai dari substitusi byte, pergeseran baris, pencampuran kolom, hingga proses pembentukan Round Key.

Implementasi ini digunakan oleh Alice, Bob, dan KDC untuk melakukan proses enkripsi dan dekripsi data.

### 1.1. Tujuan AES-128

AES-128 merupakan algoritma kriptografi simetris. Artinya, proses enkripsi dan dekripsi menggunakan kunci rahasia yang sama.

Spesifikasi AES-128:

| Parameter | Nilai |
|---|---|
| Key Size | 128 bit |
| Block Size | 128 bit |
| Key Length | 16 bytes |
| Block Length | 16 bytes |
| Number of Rounds | 10 |
| Encryption Type | Symmetric Encryption |

**Mengapa disebut AES-128?**

Angka 128 menunjukkan panjang kunci yang digunakan, yaitu 128 bit atau 16 byte.

---

# 2. Import and Basic AES Components

## 2.1. S-Box (Substitution Box)

```python
SBOX = [
    0x63, 0x7c, 0x77, 0x7b,
    0xf2, 0x6b, 0x6f, 0xc5,
    # ... nilai lainnya
]
```

S-Box merupakan tabel substitusi yang berisi 256 nilai hexadecimal.

Fungsinya adalah mengganti setiap byte dari data dengan nilai baru berdasarkan tabel substitusi.

Contohnya:

```python
SBOX[0] = 0x63
SBOX[1] = 0x7c
```

Artinya:
- Jika nilai byte adalah `0x00`, hasil substitusinya adalah `0x63`.
- Jika nilai byte adalah `0x01`, hasil substitusinya adalah `0x7c`.

Dalam AES, proses substitusi ini disebut **SubBytes**.

Tujuannya adalah memberikan sifat non-linear pada proses enkripsi sehingga hubungan antara plaintext, ciphertext, dan key menjadi lebih sulit dianalisis.

## 2.2. Inverse S-Box

```python
INV_SBOX = [0] * 256

for i in range(256):
    INV_SBOX[SBOX[i]] = i
```

Inverse S-Box merupakan tabel kebalikan dari S-Box.

Jika S-Box digunakan untuk proses enkripsi, maka Inverse S-Box digunakan untuk proses dekripsi.

Contoh:

Jika:

```python
SBOX[0] = 0x63
```

Maka:

```python
INV_SBOX[0x63] = 0
```

Dengan demikian, nilai yang telah disubstitusi dapat dikembalikan ke nilai awalnya.

**Hubungannya:**
- S-Box → SubBytes (Encryption).
- Inverse S-Box → InvSubBytes (Decryption).

## 2.3. RCON (Round Constant)

```python
RCON = [
    0x00, 0x01, 0x02, 0x04,
    0x08, 0x10, 0x20, 0x40,
    0x80, 0x1B, 0x36
]
```

RCON merupakan kumpulan konstanta yang digunakan dalam proses Key Expansion.

Fungsinya adalah memberikan nilai tambahan pada proses pembentukan Round Key agar setiap putaran memiliki kunci yang berbeda.

Pada AES-128, terdapat 10 konstanta utama yang digunakan untuk membentuk 10 putaran tambahan dari kunci awal.

---

# 3. Galois Field Multiplication (gmul)

```python
def gmul(a, b):
    p = 0

    for _ in range(8):
        if b & 1:
            p ^= a

        hi_bit_set = a & 0x80
        a = (a << 1) & 0xFF

        if hi_bit_set:
            a ^= 0x1B

        b >>= 1

    return p
```

## 3.1. Pengertian

Fungsi `gmul()` digunakan untuk melakukan perkalian dua nilai byte dalam operasi matematika AES.

Perkalian ini bukan perkalian bilangan biasa, melainkan perkalian dalam Galois Field GF(2⁸).

AES menggunakan operasi matematika ini terutama pada proses MixColumns dan InvMixColumns.

## 3.2. Penjelasan Kode

**1. Inisialisasi hasil perkalian**

```python
p = 0
```

Variabel `p` digunakan untuk menyimpan hasil perkalian.

**2. Perulangan sebanyak 8 kali**

```python
for _ in range(8):
```

Karena satu byte terdiri dari 8 bit, operasi dilakukan sebanyak delapan kali.

**3. Pemeriksaan bit paling kanan**

```python
if b & 1:
    p ^= a
```

Operator `&` digunakan untuk memeriksa bit paling kanan dari `b`.

Jika nilainya 1, nilai `a` digabungkan ke hasil menggunakan XOR (`^`).

**4. Pemeriksaan bit tertinggi**

```python
hi_bit_set = a & 0x80
```

Memeriksa apakah bit tertinggi dari `a` bernilai 1.

`0x80` dalam biner adalah:

```text
10000000
```

**5. Pergeseran bit ke kiri**

```python
a = (a << 1) & 0xFF
```

Nilai `a` digeser satu posisi ke kiri dan dibatasi agar tetap berada dalam rentang 8 bit.

**6. Reduksi polinomial**

```python
if hi_bit_set:
    a ^= 0x1B
```

Jika bit tertinggi sebelumnya bernilai 1, dilakukan XOR dengan `0x1B`.

Nilai ini merupakan bagian dari mekanisme reduksi polinomial AES, yang berkaitan dengan polinomial irreducible:

\[
x^8+x^4+x^3+x+1
\]

**7. Pergeseran nilai b**

```python
b >>= 1
```

Nilai `b` digeser satu posisi ke kanan untuk memproses bit berikutnya.

**8. Mengembalikan hasil**

```python
return p
```

Menghasilkan nilai perkalian dalam GF(2⁸).

**Kesimpulan:** `gmul()` merupakan fungsi pendukung matematika AES yang sangat penting, terutama untuk pencampuran nilai byte pada MixColumns.

---

# 4. AES128Manual Class

```python
class AES128Manual:
```

Class ini merupakan inti dari seluruh implementasi AES-128.

Di dalamnya terdapat fungsi-fungsi untuk:
- Menyiapkan kunci.
- Membuat Round Key.
- Melakukan substitusi byte.
- Melakukan pergeseran baris.
- Melakukan pencampuran kolom.
- Mengenkripsi data.
- Mendekripsi data.
- Mengolah data menggunakan mode CBC.

## 4.1. Constructor – Initialization

```python
def __init__(self, key: bytes):
    if len(key) != 16:
        # Pad or truncate key to 16 bytes for AES-128
        key = (key + b'\x00' * 16)[:16]

    self.key_schedule = self._key_expansion(key)
```

Constructor dijalankan ketika objek AES dibuat.

Contoh penggunaan:

```python
aes = AES128Manual(b"1234567890123456")
```

Penjelasan:

**1. Validasi panjang key**

```python
if len(key) != 16:
```

Memeriksa apakah panjang kunci sudah 16 byte.

**2. Padding atau truncation**

```python
key = (key + b'\x00' * 16)[:16]
```

Jika panjang key kurang dari 16 byte, program menambahkan byte kosong (`0x00`).

Jika lebih dari 16 byte, program memotongnya menjadi 16 byte pertama.

**3. Membuat Key Schedule**

```python
self.key_schedule = self._key_expansion(key)
```

Setelah mendapatkan kunci sepanjang 16 byte, program menjalankan Key Expansion untuk menghasilkan Round Key.

Catatan: Penyesuaian panjang kunci ini hanya untuk kemudahan demonstrasi. Dalam penggunaan sebenarnya, kunci sebaiknya divalidasi dan ditolak jika panjangnya tidak sesuai, bukan diam-diam diubah.

---

# 5. Key Expansion

```python
def _key_expansion(self, key):
    w = list(key)

    for i in range(16, 176, 4):
        temp = w[i-4:i]

        if i % 16 == 0:
            temp = [SBOX[b] for b in temp[1:] + temp[:1]]
            temp[0] ^= RCON[i // 16]

        for j in range(4):
            w.append(w[i - 16 + j] ^ temp[j])

    return w
```

## 5.1. Pengertian Key Expansion

Key Expansion adalah proses menghasilkan Round Key dari satu kunci utama.

AES-128 membutuhkan 11 Round Key:
- 1 Round Key untuk initial AddRoundKey.
- 9 Round Key untuk putaran 1 sampai 9.
- 1 Round Key untuk putaran terakhir.

Setiap Round Key berukuran 16 byte.

Total kebutuhan:

\[
11 \times 16 = 176 \text{ bytes}
\]

Itulah sebabnya perulangan dalam kode berhenti pada 176 byte.

## 5.2. Penjelasan Kode

**1. Mengubah key menjadi list**

```python
w = list(key)
```

Mengubah 16 byte awal menjadi daftar nilai byte.

**2. Perulangan pembentukan key**

```python
for i in range(16, 176, 4):
```

Membentuk empat byte baru pada setiap iterasi.

**3. Mengambil empat byte sebelumnya**

```python
temp = w[i-4:i]
```

Mengambil empat byte terakhir yang sudah tersedia.

**4. RotWord dan SubWord**

```python
temp = [SBOX[b] for b in temp[1:] + temp[:1]]
```

Bagian ini menjalankan dua proses:

- RotWord: Menggeser susunan empat byte satu posisi ke kiri.
- SubWord: Mengganti setiap byte menggunakan S-Box.

Contoh RotWord:

```text
Sebelum:
[ A, B, C, D ]

Sesudah:
[ B, C, D, A ]
```

Setelah itu, setiap byte disubstitusi menggunakan S-Box.

**5. XOR dengan RCON**

```python
temp[0] ^= RCON[i // 16]
```

Menggabungkan byte pertama dengan Round Constant menggunakan XOR.

**6. Membentuk empat byte baru**

```python
for j in range(4):
    w.append(w[i - 16 + j] ^ temp[j])
```

Setiap byte baru diperoleh dari XOR antara byte pada posisi 16 langkah sebelumnya dengan byte hasil transformasi.

**7. Mengembalikan Key Schedule**

```python
return w
```

Menghasilkan total 176 byte yang akan digunakan dalam proses enkripsi dan dekripsi.

---

# 6. AddRoundKey

```python
def _add_round_key(self, state, round_key):
    return [s ^ rk for s, rk in zip(state, round_key)]
```

AddRoundKey merupakan proses menggabungkan State dengan Round Key menggunakan operasi XOR.

Penjelasan:

- `state`: Data yang sedang diproses.
- `round_key`: Kunci untuk putaran tersebut.
- `zip()`: Memasangkan setiap byte State dengan byte Round Key.
- `^`: Operasi XOR.

Contoh sederhana:

```text
State     : 10101010
Round Key : 11001100
             --------
XOR       : 01100110
```

AddRoundKey digunakan pada awal enkripsi, setiap putaran, dan putaran terakhir.

Pada proses dekripsi, fungsi yang sama juga digunakan karena XOR dengan nilai yang sama dapat mengembalikan nilai awal.

---

# 7. SubBytes and InvSubBytes

## 7.1. SubBytes

```python
def _sub_bytes(self, state):
    return [SBOX[b] for b in state]
```

Fungsi ini mengganti setiap byte dalam State menggunakan S-Box.

Tujuannya adalah mengubah nilai byte agar hubungan antara plaintext dan ciphertext menjadi lebih kompleks.

## 7.2. InvSubBytes

```python
def _inv_sub_bytes(self, state):
    return [INV_SBOX[b] for b in state]
```

Fungsi ini mengembalikan byte yang telah disubstitusi menggunakan Inverse S-Box.

Digunakan pada proses dekripsi.

| Function | Purpose |
|---|---|
| `_sub_bytes()` | Substitusi byte saat enkripsi. |
| `_inv_sub_bytes()` | Mengembalikan byte saat dekripsi. |

---

# 8. ShiftRows and InvShiftRows

## 8.1. ShiftRows

```python
def _shift_rows(self, s):
    return [
        s[0],  s[5],  s[10], s[15],
        s[4],  s[9],  s[14], s[3],
        s[8],  s[13], s[2],  s[7],
        s[12], s[1],  s[6],  s[11]
    ]
```

ShiftRows merupakan proses menggeser posisi byte pada setiap baris State.

Tujuannya adalah mengubah susunan byte agar nilai-nilai dalam satu kolom tidak selalu berasal dari posisi yang sama.

AES menggunakan State berukuran 4 × 4 byte.

Secara konseptual:

```text
Sebelum ShiftRows:

A0  A1  A2  A3
B0  B1  B2  B3
C0  C1  C2  C3
D0  D1  D2  D3

Sesudah ShiftRows:

A0  A1  A2  A3
B1  B2  B3  B0
C2  C3  C0  C1
D3  D0  D1  D2
```

Baris pertama tidak digeser, baris kedua digeser satu posisi, baris ketiga dua posisi, dan baris keempat tiga posisi.

## 8.2. InvShiftRows

```python
def _inv_shift_rows(self, s):
    return [
        s[0],  s[13], s[10], s[7],
        s[4],  s[1],  s[14], s[11],
        s[8],  s[5],  s[2],  s[15],
        s[12], s[9],  s[6],  s[3]
    ]
```

InvShiftRows merupakan kebalikan dari ShiftRows.

Fungsi ini menggeser setiap baris ke arah berlawanan agar susunan byte kembali seperti sebelum ShiftRows.

---

# 9. MixColumns and InvMixColumns

## 9.1. MixColumns

```python
def _mix_columns(self, s):
    out = [0] * 16

    for c in range(4):
        col = s[c*4:(c+1)*4]

        out[c*4] = (
            gmul(col[0], 2) ^
            gmul(col[1], 3) ^
            col[2] ^
            col[3]
        )

        out[c*4 + 1] = (
            col[0] ^
            gmul(col[1], 2) ^
            gmul(col[2], 3) ^
            col[3]
        )

        out[c*4 + 2] = (
            col[0] ^
            col[1] ^
            gmul(col[2], 2) ^
            gmul(col[3], 3)
        )

        out[c*4 + 3] = (
            gmul(col[0], 3) ^
            col[1] ^
            col[2] ^
            gmul(col[3], 2)
        )

    return out
```

MixColumns merupakan proses mencampurkan nilai byte dalam setiap kolom State.

Setiap kolom terdiri dari empat byte.

Proses ini menggunakan perkalian dalam Galois Field GF(2⁸), yang dilakukan oleh fungsi `gmul()`.

Matriks transformasi AES yang digunakan:

\[
\begin{bmatrix}
02 & 03 & 01 & 01\\
01 & 02 & 03 & 01\\
01 & 01 & 02 & 03\\
03 & 01 & 01 & 02
\end{bmatrix}
\]

Setiap kolom dikalikan dengan matriks tersebut menggunakan operasi matematika AES.

Tujuannya adalah menyebarkan pengaruh setiap byte ke seluruh byte dalam kolom.

## 9.2. InvMixColumns

```python
def _inv_mix_columns(self, s):
    out = [0] * 16

    for c in range(4):
        col = s[c*4:(c+1)*4]

        out[c*4] = (
            gmul(col[0], 0x0e) ^
            gmul(col[1], 0x0b) ^
            gmul(col[2], 0x0d) ^
            gmul(col[3], 0x09)
        )
```

Fungsi ini merupakan kebalikan dari MixColumns.

Prosesnya menggunakan matriks invers:

\[
\begin{bmatrix}
0E & 0B & 0D & 09\\
09 & 0E & 0B & 0D\\
0D & 09 & 0E & 0B\\
0B & 0D & 09 & 0E
\end{bmatrix}
\]

Tujuannya adalah mengembalikan nilai State ke kondisi sebelum proses MixColumns.

Perlu diperhatikan bahwa seluruh angka hexadecimal tersebut merupakan konstanta perkalian dalam GF(2⁸), bukan perkalian bilangan biasa.

---

# 10. AES-128 Encryption

```python
def encrypt_block(self, block: bytes) -> bytes:
    state = list(block)

    state = self._add_round_key(
        state,
        self.key_schedule[0:16]
    )

    for r in range(1, 10):
        state = self._sub_bytes(state)
        state = self._shift_rows(state)
        state = self._mix_columns(state)

        state = self._add_round_key(
            state,
            self.key_schedule[r*16:(r+1)*16]
        )

    state = self._sub_bytes(state)
    state = self._shift_rows(state)

    state = self._add_round_key(
        state,
        self.key_schedule[160:176]
    )

    return bytes(state)
```

## 10.1. Pengertian

Fungsi `encrypt_block()` merupakan fungsi utama untuk mengenkripsi satu blok data sepanjang 16 byte.

AES-128 memiliki 10 rounds dengan susunan proses tertentu.

## 10.2. Penjelasan Setiap Tahapan

**1. Mengubah Block menjadi State**

```python
state = list(block)
```

Data 16 byte diubah menjadi list agar dapat diproses.

**2. Initial AddRoundKey**

```python
state = self._add_round_key(
    state,
    self.key_schedule[0:16]
)
```

State digabungkan dengan Round Key pertama.

Tahapan ini dilakukan sebelum putaran utama.

**3. Round 1 sampai Round 9**

```python
for r in range(1, 10):
```

Setiap putaran menjalankan empat tahapan:

1. SubBytes.
2. ShiftRows.
3. MixColumns.
4. AddRoundKey.

**4. Final Round**

```python
state = self._sub_bytes(state)
state = self._shift_rows(state)
state = self._add_round_key(
    state,
    self.key_schedule[160:176]
)
```

Putaran terakhir hanya menjalankan tiga tahapan:
- SubBytes.
- ShiftRows.
- AddRoundKey.

Tidak terdapat MixColumns pada final round.

**5. Menghasilkan Ciphertext**

```python
return bytes(state)
```

Mengubah State menjadi bytes dan mengembalikan hasil enkripsi.

---

# 11. AES-128 Decryption

```python
def decrypt_block(self, block: bytes) -> bytes:
    state = list(block)

    state = self._add_round_key(
        state,
        self.key_schedule[160:176]
    )

    for r in range(9, 0, -1):
        state = self._inv_shift_rows(state)
        state = self._inv_sub_bytes(state)

        state = self._add_round_key(
            state,
            self.key_schedule[r*16:(r+1)*16]
        )

        state = self._inv_mix_columns(state)

    state = self._inv_shift_rows(state)
    state = self._inv_sub_bytes(state)

    state = self._add_round_key(
        state,
        self.key_schedule[0:16]
    )

    return bytes(state)
```

## 11.1. Pengertian

Fungsi `decrypt_block()` digunakan untuk mengembalikan ciphertext menjadi plaintext.

Prosesnya merupakan kebalikan dari AES encryption.

## 11.2. Penjelasan Setiap Tahapan

**1. Initial AddRoundKey**

```python
state = self._add_round_key(
    state,
    self.key_schedule[160:176]
)
```

Dekripsi dimulai dengan Round Key terakhir.

**2. Putaran 9 sampai 1**

```python
for r in range(9, 0, -1):
```

Setiap putaran menjalankan:

1. InvShiftRows.
2. InvSubBytes.
3. AddRoundKey.
4. InvMixColumns.

**3. Final Decryption Round**

```python
state = self._inv_shift_rows(state)
state = self._inv_sub_bytes(state)

state = self._add_round_key(
    state,
    self.key_schedule[0:16]
)
```

Pada tahap terakhir, program mengembalikan State menggunakan Round Key pertama.

**4. Menghasilkan Plaintext**

```python
return bytes(state)
```

Hasilnya berupa plaintext dalam bentuk bytes.

---

# 12. CBC Mode – Cipher Block Chaining

AES bekerja dengan blok data sepanjang 16 byte. Namun, pesan yang dikirimkan Alice dan Bob dapat memiliki panjang yang berbeda-beda.

Oleh karena itu, program menggunakan mode CBC agar beberapa blok dapat diproses secara berantai.

## 12.1. Encryption CBC

```python
def encrypt_cbc(self, plaintext: str, iv: bytes) -> str:
    data = plaintext.encode('utf-8')

    pad_len = 16 - (len(data) % 16)

    data += bytes([pad_len]) * pad_len

    ciphertext = bytearray()
    prev_block = iv
```

Penjelasan:

**1. Mengubah plaintext menjadi bytes**

```python
data = plaintext.encode('utf-8')
```

Mengubah teks menjadi byte agar dapat diproses oleh AES.

**2. Menghitung Padding**

```python
pad_len = 16 - (len(data) % 16)
```

Menghitung jumlah byte tambahan agar panjang data menjadi kelipatan 16 byte.

**3. Menambahkan PKCS7 Padding**

```python
data += bytes([pad_len]) * pad_len
```

Menambahkan byte padding sesuai jumlah yang dibutuhkan.

Contoh:

Jika data membutuhkan 3 byte tambahan, program akan menambahkan:

```text
03 03 03
```

Jika panjang plaintext sudah tepat kelipatan 16 byte, program tetap menambahkan satu blok padding penuh.

**4. Menyiapkan Ciphertext dan IV**

```python
ciphertext = bytearray()
prev_block = iv
```

- `ciphertext`: Menampung hasil enkripsi.
- `prev_block`: Menyimpan IV atau ciphertext blok sebelumnya.

### 12.1.1. Proses Enkripsi Setiap Blok

```python
for i in range(0, len(data), 16):
    block = data[i:i+16]

    xored = bytes([
        b ^ p
        for b, p in zip(block, prev_block)
    ])

    encrypted = self.encrypt_block(xored)

    ciphertext.extend(encrypted)

    prev_block = encrypted
```

Penjelasan:

1. Data dibagi menjadi blok-blok berukuran 16 byte.
2. Blok pertama di-XOR dengan IV.
3. Hasil XOR dienkripsi menggunakan AES.
4. Ciphertext disimpan.
5. Ciphertext yang dihasilkan menjadi input penghubung untuk blok berikutnya.

Inilah konsep utama CBC: setiap blok bergantung pada ciphertext blok sebelumnya.

**6. Menghasilkan Hexadecimal**

```python
return ciphertext.hex()
```

Mengubah ciphertext menjadi format hexadecimal agar lebih mudah dikirim dan ditampilkan sebagai teks.

## 12.2. Decryption CBC

```python
def decrypt_cbc(self, hex_ciphertext: str, iv: bytes) -> str:
```

Fungsi ini digunakan untuk mengembalikan ciphertext CBC menjadi plaintext.

**1. Mengubah Hexadecimal menjadi Bytes**

```python
ciphertext = bytes.fromhex(hex_ciphertext)
```

Mengubah data hexadecimal menjadi bytes.

**2. Menyiapkan Variabel**

```python
plaintext = bytearray()
prev_block = iv
```

Mempersiapkan tempat untuk hasil dekripsi dan menyimpan IV atau ciphertext sebelumnya.

**3. Memproses Setiap Blok**

```python
for i in range(0, len(ciphertext), 16):
    block = ciphertext[i : i + 16]

    decrypted = self.decrypt_block(block)

    xored = bytes([
        d ^ p
        for d, p in zip(decrypted, prev_block)
    ])

    plaintext.extend(xored)

    prev_block = block
```

Penjelasan:

1. Ciphertext dibagi menjadi blok 16 byte.
2. Setiap blok didekripsi menggunakan AES.
3. Hasil dekripsi di-XOR dengan IV atau ciphertext blok sebelumnya.
4. Hasilnya dimasukkan ke dalam plaintext.
5. Ciphertext saat ini menjadi referensi untuk blok berikutnya.

**4. Validasi Padding**

```python
pad_len = plaintext[-1]

if pad_len < 1 or pad_len > 16:
    raise ValueError("Padding tidak valid (Kunci Salah).")
```

Program membaca byte terakhir untuk menentukan jumlah padding.

**5. Menghapus Padding**

```python
return plaintext[:-pad_len].decode("utf-8")
```

Padding dihapus, kemudian hasilnya dikonversi kembali menjadi teks UTF-8.

**Catatan:** Validasi padding pada kode ini masih sederhana. Program hanya memeriksa nilai byte terakhir, belum memeriksa apakah seluruh byte padding memiliki nilai yang sama.

---

# 13. Complete AES-128 Process

Berikut merupakan gambaran keseluruhan proses AES-128 yang diimplementasikan dalam file ini.

## 13.1. Encryption Flow

```text
       ORIGINAL PLAINTEXT
               |
               v
        UTF-8 Encoding
               |
               v
         PKCS7 Padding
               |
               v
       Divide into 16-byte
             Blocks
               |
               v
          XOR with IV
               |
               v
          AES-128
               |
       +-------+-------+
       |               |
       v               |
     XOR with          |
 Previous Ciphertext   |
       |               |
       +---------------+
               |
               v
         HEX Encoding
               |
               v
          CIPHERTEXT
```

## 13.2. AES Encryption Round Structure

```text
         INITIAL STATE
               |
               v
          AddRoundKey
               |
               v
       +---------------+
       |  ROUND 1–9    |
       |               |
       |   SubBytes    |
       |   ShiftRows   |
       |   MixColumns  |
       |   AddRoundKey |
       +---------------+
               |
               v
         FINAL ROUND
               |
          SubBytes
               |
          ShiftRows
               |
          AddRoundKey
               |
               v
           CIPHERTEXT
```

## 13.3. Decryption Flow

```text
          CIPHERTEXT
               |
               v
         HEX Decoding
               |
               v
         AES Decryption
               |
               v
          XOR with IV
               |
               v
       Reconstruct Plaintext
               |
               v
       Remove PKCS7 Padding
               |
               v
         UTF-8 Decoding
               |
               v
       ORIGINAL PLAINTEXT
```

---

# 14. Relationship Between AES Manual, Alice, Bob, and KDC

File `aes_manual.py` merupakan komponen kriptografi yang digunakan oleh ketiga program lainnya.

Hubungan penggunaannya:

| Component | AES Function Used | Purpose |
|---|---|---|
| Alice | `encrypt_cbc()` | Mengenkripsi pesan untuk Bob. |
| Alice | `decrypt_cbc()` | Membuka Wrapped Data dan Ticket yang ditujukan kepadanya. |
| Bob | `encrypt_cbc()` | Mengenkripsi pesan untuk Alice. |
| Bob | `decrypt_cbc()` | Membuka Wrapped Data dan Ticket yang ditujukan kepadanya. |
| KDC | `encrypt_cbc()` | Mengenkripsi Ticket dan distribusi Session Key. |

Semua fungsi tersebut pada akhirnya menggunakan:
- `encrypt_block()` untuk enkripsi blok.
- `decrypt_block()` untuk dekripsi blok.
- `_key_expansion()` untuk membentuk Round Key.

Dengan kata lain, AES Manual merupakan mesin kriptografi yang mendukung proses keamanan komunikasi Alice, Bob, dan KDC.

---

# 15. Important Notes and Limitations

Implementasi ini dibuat untuk memahami konsep AES-128 secara manual. Ada beberapa keterbatasan yang perlu diperhatikan:

1. **Key Validation:** Constructor otomatis melakukan padding atau pemotongan jika panjang key bukan 16 byte. Hal ini tidak ideal untuk penggunaan keamanan sebenarnya.
2. **Block Validation:** Fungsi `encrypt_block()` dan `decrypt_block()` mengasumsikan input selalu tepat 16 byte.
3. **Padding Validation:** Fungsi dekripsi belum memvalidasi seluruh byte PKCS7 Padding.
4. **CBC Integrity:** Mode CBC tidak menyediakan autentikasi atau perlindungan integritas ciphertext dengan sendirinya.
5. **IV Reuse:** IV harus unik untuk setiap proses enkripsi menggunakan kunci yang sama. Implementasi sistem yang menggunakan AES-CBC perlu memastikan hal tersebut.
6. **Manual Implementation:** Untuk penggunaan produksi, lebih disarankan menggunakan library kriptografi yang telah diuji dan mendukung authenticated encryption seperti AES-GCM.

---

# 16. Conclusion

AES-128 Manual merupakan implementasi algoritma enkripsi simetris yang dibangun menggunakan Python tanpa library AES eksternal.

Prosesnya terdiri dari pembentukan Round Key melalui Key Expansion, transformasi data melalui SubBytes, ShiftRows, MixColumns, dan AddRoundKey, serta proses kebalikannya ketika melakukan dekripsi.

Untuk mengolah pesan dengan panjang bervariasi, implementasi ini menggunakan AES-CBC dengan PKCS7 Padding.

**The main idea is that AES-128 transforms plaintext into ciphertext using a secret key, while the decryption process restores the original message using the same key.**
