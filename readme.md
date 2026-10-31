# README – Alice, Key Distribution Center (KDC), and AES-128

## 1. Project Overview

Program ini merupakan simulasi komunikasi aman antara dua pengguna, yaitu **Alice dan Bob**, dengan bantuan **Key Distribution Center (KDC)**.

Tujuan utama sistem adalah memungkinkan Alice dan Bob bertukar pesan secara terenkripsi tanpa harus membagikan Session Key secara langsung.

Sistem menggunakan:
- **KDC (Key Distribution Center):** Menghasilkan dan mendistribusikan Session Key.
- **Alice:** Pengirim dan penerima pesan.
- **Bob:** Pengirim dan penerima pesan.
- **AES-128:** Algoritma enkripsi dan dekripsi untuk melindungi Session Key, Ticket, dan pesan.
- **TCP Socket:** Media komunikasi antara Alice, Bob, dan KDC.

### 1.1. Komponen Utama

| Komponen | Fungsi |
|---|---|
| Alice | Meminta Session Key kepada KDC dan melakukan komunikasi dengan Bob. |
| Bob | Meminta Session Key kepada KDC dan melakukan komunikasi dengan Alice. |
| KDC | Membuat Session Key dan mendistribusikannya secara aman kepada kedua pihak. |
| AES-128 | Melakukan proses enkripsi dan dekripsi menggunakan kunci 128-bit. |

---

# 2. ALICE – Client Implementation

## 2.1. Pengertian dan Fungsi Alice

Alice merupakan salah satu client dalam sistem komunikasi yang bertugas mengirimkan dan menerima pesan dari Bob.

Alice tidak membuat Session Key sendiri. Sebaliknya, Alice meminta Session Key kepada KDC agar komunikasi dengan Bob dapat dilakukan menggunakan kunci yang sama.

Alice memiliki dua jenis kunci:

1. **Alice Master Key:** Kunci rahasia yang hanya diketahui Alice dan KDC. Digunakan untuk membuka Session Key yang diberikan oleh KDC.
2. **Session Key:** Kunci sementara yang diberikan oleh KDC dan digunakan untuk mengenkripsi serta mendekripsi pesan dengan Bob.

Dengan demikian, Alice menggunakan Master Key untuk memperoleh Session Key, kemudian menggunakan Session Key untuk melakukan komunikasi.

## 2.2. Import Library

```python
import os
import json
import socket
import threading
from aes_manual import AES128Manual
```

Penjelasan:

- `os`: Mengambil konfigurasi jaringan melalui environment variables.
- `json`: Mengubah data menjadi format JSON untuk dikirim melalui jaringan.
- `socket`: Membuat koneksi komunikasi antara Alice, Bob, dan KDC.
- `threading`: Menjalankan listener agar Alice dapat menerima pesan tanpa menghentikan menu utama.
- `AES128Manual`: Menggunakan implementasi algoritma AES-128 yang dibuat secara manual dalam file `aes_manual.py`.

**Intinya:** Bagian ini mempersiapkan library yang dibutuhkan Alice untuk berkomunikasi dan melakukan enkripsi maupun dekripsi.

## 2.3. Network Configuration

```python
KDC_HOST = os.getenv("KDC_HOST", "192.168.1.33")
KDC_PORT = int(os.getenv("KDC_PORT", "5000"))

MY_HOST = os.getenv("ALICE_HOST", "0.0.0.0")
MY_PORT = int(os.getenv("ALICE_PORT", "5001"))

BOB_HOST = os.getenv("BOB_HOST", "192.168.1.34")
BOB_PORT = int(os.getenv("BOB_PORT", "5002"))
```

Penjelasan:

Bagian ini mengatur alamat IP dan port yang digunakan Alice untuk berkomunikasi.

| Variable | Fungsi |
|---|---|
| KDC_HOST | Alamat IP server KDC. |
| KDC_PORT | Port komunikasi menuju KDC. |
| MY_HOST | Alamat interface tempat Alice menerima koneksi. |
| MY_PORT | Port yang digunakan Alice untuk menerima pesan. |
| BOB_HOST | Alamat IP Bob. |
| BOB_PORT | Port komunikasi menuju Bob. |

Contoh konfigurasi:
- KDC menggunakan port 5000.
- Alice menggunakan port 5001.
- Bob menggunakan port 5002.

`0.0.0.0` berarti Alice menerima koneksi melalui semua interface jaringan IPv4 yang tersedia.

## 2.4. Alice Master Key

```python
ALICE_MASTER_KEY = "AliceMasterKey12"
```

Penjelasan:

Master Key merupakan kunci rahasia milik Alice yang juga disimpan oleh KDC.

Kunci ini digunakan untuk membuka Session Key yang dikirimkan oleh KDC dalam bentuk terenkripsi.

Pada implementasi ini, Master Key menggunakan format ASCII dengan panjang 16 byte agar sesuai dengan AES-128.

**Hubungannya dengan AES:**

```python
aes_m = AES128Manual(ALICE_MASTER_KEY.encode('utf-8'))
```

Kode tersebut mengubah Master Key menjadi byte agar dapat diproses oleh algoritma AES-128.

## 2.5. Fungsi Send JSON dan Receive JSON

```python
def send_json(conn, data):
    conn.sendall(json.dumps(data).encode("utf-8") + b"\n")
```

Fungsi ini digunakan untuk mengirimkan data melalui koneksi TCP dalam format JSON.

```python
def recv_json(conn):
    data = bytearray()
    while not data.endswith(b"\n"):
        chunk = conn.recv(4096)
        if not chunk:
            break
        data.extend(chunk)
    return json.loads(data.decode("utf-8"))
```

Fungsi ini digunakan untuk menerima data JSON dari pihak lain.

Penjelasan:
- `conn.recv(4096)`: Menerima data dari jaringan.
- `bytearray()`: Menampung data yang diterima.
- `decode("utf-8")`: Mengubah byte menjadi teks.
- `json.loads()`: Mengubah teks JSON menjadi objek Python.

**Intinya:** Kedua fungsi ini menjadi mekanisme pertukaran data antara Alice, Bob, dan KDC.

## 2.6. Start Listener – Menerima Pesan

```python
def start_listener():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind((MY_HOST, MY_PORT))
        s.listen()
```

Bagian ini membuat server listener pada Alice.

Penjelasan:
- `AF_INET`: Menggunakan alamat IPv4.
- `SOCK_STREAM`: Menggunakan protokol TCP.
- `SO_REUSEADDR`: Memungkinkan penggunaan kembali alamat socket.
- `bind()`: Menghubungkan socket dengan alamat dan port Alice.
- `listen()`: Menunggu koneksi masuk.

Selanjutnya:

```python
conn, addr = s.accept()
```

Alice menunggu koneksi dari pihak lain, misalnya Bob.

### 2.6.1. Membaca Ticket dan Ciphertext

```python
pkt = recv_json(conn)

ticket_hex = pkt["ticket"]
iv_hex = pkt["iv_hex"]
ciphertext_hex = pkt["ciphertext"]
```

Alice menerima tiga informasi utama:

1. `ticket`: Ticket terenkripsi yang dibuat oleh KDC.
2. `iv_hex`: Initialization Vector yang digunakan dalam AES-CBC.
3. `ciphertext`: Pesan yang sudah dienkripsi oleh pengirim.

### 2.6.2. Dekripsi Ticket untuk Alice

```python
aes_m = AES128Manual(ALICE_MASTER_KEY.encode('utf-8'))

ticket_plain = aes_m.decrypt_cbc(
    ticket_hex,
    bytes.fromhex(iv_hex)
)

sender, target, session_key_hex, _ = ticket_plain.split("|")
```

Bagian ini merupakan salah satu proses keamanan yang penting.

Penjelasan:

1. Alice menggunakan Master Key miliknya untuk membuat objek AES-128.
2. Alice melakukan dekripsi terhadap Ticket.
3. Ticket yang berhasil dibuka berisi identitas pengirim, penerima, Session Key, dan IV.
4. Alice mengambil Session Key yang diperlukan untuk membuka pesan.

Mengapa Alice menggunakan Master Key?

Karena Ticket yang diberikan KDC kepada Alice telah dienkripsi menggunakan Master Key Alice. Dengan demikian, pihak yang tidak memiliki Master Key tersebut tidak dapat membuka Ticket dengan benar.

### 2.6.3. Dekripsi Pesan Utama

```python
aes_session = AES128Manual(bytes.fromhex(session_key_hex))

msg = aes_session.decrypt_cbc(
    ciphertext_hex,
    bytes.fromhex(iv_hex)
)
```

Setelah mendapatkan Session Key, Alice menggunakannya untuk mendekripsi pesan utama dari Bob.

Prosesnya:
1. Mengubah Session Key dari format hexadecimal menjadi bytes.
2. Membuat objek AES-128 menggunakan Session Key.
3. Melakukan dekripsi terhadap ciphertext.
4. Menghasilkan plaintext atau pesan asli.

```python
print(f"[PESAN MASUK dari {sender}]: {msg}")
```

Pesan asli kemudian ditampilkan pada terminal Alice.

**Kesimpulan bagian Listener:**

Alice menerima Ticket dan Ciphertext, membuka Ticket menggunakan Master Key, kemudian menggunakan Session Key yang diperoleh untuk mendekripsi pesan.

## 2.7. Send Message – Mengirim Pesan ke Bob

Bagian ini merupakan proses utama ketika Alice ingin mengirimkan pesan kepada Bob.

### 2.7.1. Request Session Key ke KDC

```python
with socket.create_connection((KDC_HOST, KDC_PORT)) as conn:
    send_json(conn, {
        "type": "REQUEST_SESSION_KEY",
        "from": "Alice",
        "to": "Bob"
    })

    res = recv_json(conn)
```

Penjelasan:

Alice membuat koneksi dengan KDC dan mengirimkan permintaan Session Key.

Informasi yang dikirim:
- `type`: Jenis permintaan.
- `from`: Identitas pengirim, yaitu Alice.
- `to`: Identitas penerima, yaitu Bob.

KDC kemudian memproses permintaan tersebut dan mengirimkan respons kepada Alice.

### 2.7.2. Membuka Session Key dari KDC

```python
iv = bytes.fromhex(res["iv_hex"])

aes_m = AES128Manual(ALICE_MASTER_KEY.encode('utf-8'))

unwrapped = aes_m.decrypt_cbc(res["wrapped"], iv)

session_key_hex, iv_hex, ticket_for_bob = unwrapped.split("|")
```

Bagian ini digunakan untuk membuka respons terenkripsi dari KDC.

Penjelasan:

1. Alice menerima IV dan data `wrapped` dari KDC.
2. Alice menggunakan Master Key miliknya untuk mendekripsi data tersebut.
3. Setelah berhasil, Alice mendapatkan Session Key.
4. Alice juga mendapatkan IV dan Ticket khusus untuk Bob.

Mengapa ada `ticket_for_bob`?

Karena KDC sudah membuat Ticket khusus untuk Bob yang dienkripsi menggunakan Master Key Bob.

Alice tidak perlu mengetahui isi Ticket tersebut. Alice hanya meneruskannya kepada Bob.

### 2.7.3. Enkripsi Pesan Menggunakan Session Key

```python
pesan = input("Masukkan pesan ASCII untuk Bob: ")

aes_session = AES128Manual(bytes.fromhex(session_key_hex))

ciphertext_hex = aes_session.encrypt_cbc(
    pesan,
    bytes.fromhex(iv_hex)
)
```

Penjelasan:

Setelah mendapatkan Session Key, Alice memasukkan pesan yang ingin dikirimkan kepada Bob.

Pesan kemudian dienkripsi menggunakan AES-128 dengan mode CBC.

Hasil enkripsi disimpan dalam variabel `ciphertext_hex`.

**Perbedaan kedua kunci:**

- Master Key: Digunakan untuk membuka distribusi Session Key dari KDC.
- Session Key: Digunakan untuk mengenkripsi pesan yang dikirimkan kepada Bob.

### 2.7.4. Mengirim Pesan ke Bob

```python
with socket.create_connection((BOB_HOST, BOB_PORT)) as conn:
    send_json(conn, {
        "ticket": ticket_for_bob,
        "iv_hex": iv_hex,
        "ciphertext": ciphertext_hex
    })
```

Alice mengirimkan tiga data kepada Bob:

| Data | Kegunaan |
|---|---|
| Ticket | Memberikan informasi Session Key kepada Bob secara terenkripsi. |
| IV | Digunakan dalam proses AES-CBC. |
| Ciphertext | Pesan yang telah dienkripsi Alice. |

Bob kemudian membuka Ticket menggunakan Master Key miliknya untuk mendapatkan Session Key dan mendekripsi pesan.

## 2.8. Main Program – Menu Alice

```python
def main():
    threading.Thread(
        target=start_listener,
        daemon=True
    ).start()
```

Bagian ini menjalankan listener menggunakan thread terpisah.

Tujuannya agar Alice tetap dapat menerima pesan ketika menu utama sedang berjalan.

Menu Alice:

```python
while True:
    print("1. Kirim Pesan ke Bob")
    print("2. Ubah Master Key (Interaktif)")
    print("3. Keluar")
```

Terdapat tiga pilihan:

1. Kirim pesan kepada Bob.
2. Mengubah Master Key Alice secara lokal.
3. Keluar dari program.

---

# 3. KEY DISTRIBUTION CENTER (KDC)

## 3.1. Pengertian dan Fungsi KDC

KDC merupakan pusat pengelolaan dan distribusi kunci dalam sistem komunikasi yang menggunakan konsep symmetric key cryptography.

Dalam program ini, KDC berperan sebagai pihak ketiga yang dipercaya oleh Alice dan Bob.

Fungsi utama KDC adalah:
- Menyimpan Master Key Alice dan Bob.
- Membuat Session Key secara acak.
- Mengenkripsi Ticket untuk penerima.
- Mengenkripsi distribusi Session Key untuk peminta.
- Mengirimkan informasi yang diperlukan agar Alice dan Bob dapat berkomunikasi secara aman.

**Mengapa KDC diperlukan?**

Tanpa KDC, Alice dan Bob perlu memiliki mekanisme tersendiri untuk menyepakati dan membagikan kunci rahasia.

Dengan KDC, proses distribusi Session Key dapat dilakukan secara terpusat.

## 3.2. Import Library

```python
import os
import json
import socket
import secrets
from aes_manual import AES128Manual
```

Penjelasan:

- `os`: Mengambil konfigurasi jaringan.
- `json`: Mengelola format data komunikasi.
- `socket`: Membuat server komunikasi.
- `secrets`: Menghasilkan angka acak yang sesuai untuk kebutuhan kriptografi.
- `AES128Manual`: Melakukan enkripsi dan dekripsi AES-128.

Perlu diperhatikan bahwa `secrets` sangat penting karena digunakan untuk menghasilkan Session Key dan IV secara acak.

## 3.3. Network Configuration

```python
HOST = os.getenv("KDC_HOST", "0.0.0.0")
PORT = int(os.getenv("KDC_PORT", "5000"))
```

Penjelasan:

- `HOST`: Menentukan interface jaringan yang digunakan KDC.
- `PORT`: Menentukan port komunikasi KDC.

KDC menggunakan port 5000 sebagai port default.

## 3.4. Master Key Configuration

```python
ALICE_MASTER_KEY = b"AliceMasterKey12"

BOB_MASTER_KEY = b"BobMasterKey1234"
```

KDC menyimpan Master Key milik Alice dan Bob.

Kedua Master Key ini digunakan untuk mengenkripsi dan mendekripsi informasi Session Key.

Hubungan Master Key:

| Kunci | Diketahui oleh | Fungsi |
|---|---|---|
| Alice Master Key | Alice dan KDC | Melindungi distribusi Session Key untuk Alice. |
| Bob Master Key | Bob dan KDC | Melindungi Ticket untuk Bob. |
| Session Key | Alice, Bob, dan KDC saat distribusi | Melindungi komunikasi antara Alice dan Bob. |

Master Key digunakan dalam jangka panjang sebagai kunci bersama, sedangkan Session Key dibuat untuk kebutuhan komunikasi.

## 3.5. Fungsi Send JSON dan Receive JSON

```python
def send_json(conn, data):
    conn.sendall(
        json.dumps(data).encode("utf-8") + b"\n"
    )
```

Digunakan untuk mengirimkan respons dari KDC kepada client dalam format JSON.

```python
def recv_json(conn):
    data = bytearray()

    while not data.endswith(b"\n"):
        chunk = conn.recv(4096)

        if not chunk:
            break

        data.extend(chunk)

    return json.loads(data.decode("utf-8"))
```

Digunakan untuk menerima permintaan dari Alice atau Bob.

Pada implementasi KDC, terdapat tambahan validasi ukuran request agar data yang diterima tidak melebihi 1.000.000 byte.

## 3.6. Make Distribution – Proses Distribusi Session Key

```python
def make_distribution(sender: str, target: str):
```

Fungsi ini merupakan bagian paling penting dalam kode KDC.

Fungsi tersebut bertugas membuat Session Key dan menyiapkan data terenkripsi untuk pengirim dan penerima.

### 3.6.1. Menentukan Master Key

```python
if sender == "Alice" and target == "Bob":
    requester_key = ALICE_MASTER_KEY
    recipient_key = BOB_MASTER_KEY

elif sender == "Bob" and target == "Alice":
    requester_key = BOB_MASTER_KEY
    recipient_key = ALICE_MASTER_KEY
```

Penjelasan:

KDC memeriksa siapa yang meminta Session Key dan siapa penerimanya.

Contoh:

Jika Alice ingin mengirim pesan kepada Bob:
- Requester Key = Alice Master Key.
- Recipient Key = Bob Master Key.

Jika Bob ingin mengirim pesan kepada Alice:
- Requester Key = Bob Master Key.
- Recipient Key = Alice Master Key.

Dengan demikian, KDC dapat menentukan Master Key yang sesuai untuk masing-masing pihak.

### 3.6.2. Membuat Session Key dan IV

```python
session_key = secrets.token_bytes(16)

iv = secrets.token_bytes(16)

iv_hex = iv.hex()
```

Penjelasan:

KDC menghasilkan dua nilai acak.

**1. Session Key**

```python
session_key = secrets.token_bytes(16)
```

Menghasilkan kunci acak sepanjang 16 byte atau 128 bit.

Kunci ini akan digunakan Alice dan Bob untuk mengenkripsi serta mendekripsi pesan.

**2. Initialization Vector (IV)**

```python
iv = secrets.token_bytes(16)
```

Menghasilkan IV sepanjang 16 byte.

IV digunakan dalam mode AES-CBC untuk membantu menghasilkan ciphertext yang berbeda ketika plaintext yang sama dienkripsi dengan IV yang berbeda.

Perlu diingat bahwa IV bukan merupakan Secret Key.

### 3.6.3. Membuat Ticket untuk Penerima

```python
ticket_payload = (
    f"{sender}|{target}|{session_key.hex()}|{iv_hex}"
)

ticket_for_target = AES128Manual(
    recipient_key
).encrypt_cbc(ticket_payload, iv)
```

Bagian ini digunakan untuk membuat Ticket khusus penerima.

Isi Ticket terdiri dari:

1. Identitas pengirim.
2. Identitas penerima.
3. Session Key.
4. IV.

Selanjutnya, seluruh informasi tersebut dienkripsi menggunakan Recipient Master Key.

Contoh ketika Alice ingin mengirim pesan kepada Bob:

KDC mengenkripsi Ticket menggunakan Bob Master Key.

Dengan demikian, Alice dapat meneruskan Ticket tersebut kepada Bob tanpa harus membuka atau mengetahui proses pembukaan Ticket oleh Bob.

### 3.6.4. Membungkus Session Key untuk Pengirim

```python
requester_payload = (
    f"{session_key.hex()}|{iv_hex}|{ticket_for_target}"
)

wrapped_for_requester = AES128Manual(
    requester_key
).encrypt_cbc(requester_payload, iv)
```

Bagian ini digunakan untuk mengamankan distribusi Session Key kepada pihak yang meminta.

KDC memasukkan tiga informasi:
- Session Key.
- IV.
- Ticket untuk penerima.

Kemudian KDC mengenkripsi seluruh informasi tersebut menggunakan Requester Master Key.

Contoh Alice mengirim pesan kepada Bob:

1. KDC membuat Session Key.
2. KDC membuat Ticket yang dienkripsi menggunakan Bob Master Key.
3. KDC memasukkan Session Key dan Ticket ke dalam Requester Payload.
4. KDC mengenkripsi Requester Payload menggunakan Alice Master Key.
5. KDC mengirimkan hasilnya kepada Alice.

Alice kemudian dapat membuka Requester Payload menggunakan Alice Master Key.

### 3.6.5. Mengirimkan Hasil Distribusi

```python
return {
    "status": "ok",
    "iv_hex": iv_hex,
    "wrapped": wrapped_for_requester
}
```

KDC mengembalikan respons yang berisi:
- Status permintaan.
- IV.
- Wrapped Session Key beserta Ticket.

**Inti dari fungsi make_distribution:**

KDC membuat satu Session Key, kemudian mengamankannya untuk dua pihak menggunakan Master Key masing-masing.

---

## 3.7. Handle Client – Memproses Permintaan

```python
def handle_client(conn, address):
```

Fungsi ini menangani setiap koneksi yang masuk ke KDC.

### 3.7.1. Mengubah Master Key

```python
if req.get("type") == "CHANGE_KEYS":
    alice_key = req.get("alice_key", "").encode(
        "ascii", errors="strict"
    )

    bob_key = req.get("bob_key", "").encode(
        "ascii", errors="strict"
    )
```

Bagian ini disediakan untuk memperbarui Master Key Alice dan Bob.

Selanjutnya:

```python
if len(alice_key) != 16 or len(bob_key) != 16:
    raise ValueError(
        "Both master keys must be exactly 16 ASCII bytes"
    )
```

KDC memvalidasi agar kedua Master Key memiliki panjang tepat 16 byte.

Jika valid:

```python
ALICE_MASTER_KEY, BOB_MASTER_KEY = alice_key, bob_key
```

KDC memperbarui Master Key yang disimpan.

Catatan: Fitur ini tersedia di sisi KDC, tetapi menu perubahan Master Key pada program Alice dan Bob yang dilampirkan hanya mengubah variabel lokal. Belum ada pengiriman request `CHANGE_KEYS` dari kedua client.

### 3.7.2. Memeriksa Request Session Key

```python
if req.get("type") != "REQUEST_SESSION_KEY":
    raise ValueError("Unsupported request type")
```

KDC memastikan bahwa permintaan yang diterima merupakan permintaan distribusi Session Key.

### 3.7.3. Memproses Request

```python
sender, target = req.get("from"), req.get("to")

response = make_distribution(sender, target)

send_json(conn, response)
```

Penjelasan:

1. KDC membaca identitas pengirim.
2. KDC membaca identitas penerima.
3. KDC menjalankan fungsi `make_distribution()`.
4. KDC mengirimkan hasil distribusi kepada pihak yang meminta.

Contoh:

Alice mengirimkan request:

```python
{
    "type": "REQUEST_SESSION_KEY",
    "from": "Alice",
    "to": "Bob"
}
```

KDC kemudian membuat Session Key untuk komunikasi Alice dengan Bob.

## 3.8. Main Program – KDC Server

```python
def main():
    with socket.socket(
        socket.AF_INET,
        socket.SOCK_STREAM
    ) as server:

        server.setsockopt(
            socket.SOL_SOCKET,
            socket.SO_REUSEADDR,
            1
        )

        server.bind((HOST, PORT))
        server.listen()
```

Penjelasan:

Bagian ini menjalankan KDC sebagai server yang siap menerima koneksi dari Alice dan Bob.

```python
while True:
    conn, address = server.accept()
    handle_client(conn, address)
```

KDC terus menunggu permintaan dari client.

Ketika Alice atau Bob terhubung, KDC memproses permintaan melalui fungsi `handle_client()`.

---

# 4. AES-128 – Supporting Encryption Algorithm

## 4.1. Pengertian AES-128

AES (Advanced Encryption Standard) merupakan algoritma kriptografi simetris yang digunakan untuk melindungi informasi.

Disebut simetris karena proses enkripsi dan dekripsi menggunakan kunci rahasia yang sama.

AES-128 menggunakan:
- Panjang kunci: 128 bit.
- Panjang blok data: 128 bit.
- Jumlah putaran: 10 rounds.

Dalam program ini, AES-128 digunakan untuk:
1. Mengenkripsi Session Key yang diberikan KDC kepada Alice atau Bob.
2. Mengenkripsi Ticket untuk penerima.
3. Mengenkripsi pesan yang dikirimkan Alice dan Bob.
4. Mendekripsi informasi tersebut agar dapat dibaca kembali oleh pihak yang memiliki kunci yang sesuai.

## 4.2. Hubungan AES dengan Alice dan KDC

Ada tiga penggunaan AES yang perlu dipahami:

| Proses | Algoritma | Kunci yang digunakan |
|---|---|---|
| KDC mengenkripsi Ticket untuk Bob | AES-128 CBC | Bob Master Key |
| KDC mengenkripsi distribusi Session Key untuk Alice | AES-128 CBC | Alice Master Key |
| Alice mengenkripsi pesan untuk Bob | AES-128 CBC | Session Key |

Ketiga proses tersebut menggunakan algoritma AES-128, tetapi kunci yang digunakan berbeda sesuai kebutuhan.

## 4.3. Key Expansion

```python
def _key_expansion(self, key):
    w = list(key)

    for i in range(16, 176, 4):
        temp = w[i-4:i]

        if i % 16 == 0:
            temp = [
                SBOX[b]
                for b in temp[1:] + temp[:1]
            ]

            temp[0] ^= RCON[i // 16]
```

Penjelasan:

Key Expansion merupakan proses menghasilkan beberapa Round Key dari satu Master Key.

AES-128 membutuhkan 11 Round Key, masing-masing sepanjang 16 byte.

Round Key tersebut digunakan pada setiap tahapan enkripsi dan dekripsi.

## 4.4. AES Encryption Process

Dalam program, proses enkripsi terdiri dari beberapa tahapan.

**1. AddRoundKey**

```python
def _add_round_key(self, state, round_key):
    return [
        s ^ rk
        for s, rk in zip(state, round_key)
    ]
```

Menggabungkan data dengan Round Key menggunakan operasi XOR.

**2. SubBytes**

```python
def _sub_bytes(self, state):
    return [SBOX[b] for b in state]
```

Mengganti setiap byte menggunakan tabel substitusi S-Box.

**3. ShiftRows**

```python
def _shift_rows(self, s):
```

Menggeser posisi byte pada setiap baris untuk mengubah susunan data.

**4. MixColumns**

```python
def _mix_columns(self, s):
```

Melakukan transformasi matematis pada setiap kolom untuk mencampurkan nilai byte.

**5. Encryption Rounds**

```python
for r in range(1, 10):
    state = self._sub_bytes(state)
    state = self._shift_rows(state)
    state = self._mix_columns(state)
    state = self._add_round_key(
        state,
        self.key_schedule[r*16:(r+1)*16]
    )
```

Keempat tahapan tersebut dijalankan sebanyak sembilan putaran.

Pada putaran terakhir, MixColumns tidak digunakan.

## 4.5. AES Decryption Process

Dekripsi merupakan kebalikan dari proses enkripsi.

Program menggunakan beberapa fungsi:
- `inv_shift_rows()`
- `inv_sub_bytes()`
- `add_round_key()`
- `inv_mix_columns()`

Tujuannya adalah mengembalikan ciphertext menjadi plaintext menggunakan kunci yang sesuai.

## 4.6. AES-CBC Mode

Program menggunakan mode CBC (Cipher Block Chaining).

```python
def encrypt_cbc(self, plaintext: str, iv: bytes):
```

Pada mode CBC, setiap blok plaintext terlebih dahulu digabungkan dengan blok ciphertext sebelumnya menggunakan XOR sebelum dienkripsi.

Untuk blok pertama, digunakan Initialization Vector (IV).

Gambaran proses:

```text
Plaintext Block
       |
       XOR <--- IV
       |
    AES-128
       |
   Ciphertext
       |
       XOR <--- Next Plaintext Block
       |
    AES-128
       |
   Next Ciphertext
```

Pada proses dekripsi, ciphertext diproses menggunakan AES decryption, kemudian hasilnya di-XOR dengan IV atau ciphertext blok sebelumnya.

Program juga menggunakan PKCS7 Padding agar panjang plaintext sesuai dengan ukuran blok AES sebesar 16 byte.

---

# 5. Overall Communication Flow

Berikut merupakan alur komunikasi ketika Alice mengirimkan pesan kepada Bob.

```text
             ALICE
               |
               | 1. Request Session Key
               |
               v
              KDC
               |
               | 2. Generate Session Key
               |
               | 3. Encrypt Ticket
               |    Using Bob Master Key
               |
               | 4. Encrypt Wrapped Data
               |    Using Alice Master Key
               |
               v
             ALICE
               |
               | 5. Decrypt Wrapped Data
               |    Using Alice Master Key
               |
               | 6. Get Session Key
               |
               | 7. Encrypt Message
               |    Using Session Key
               |
               v
              BOB
               |
               | 8. Decrypt Ticket
               |    Using Bob Master Key
               |
               | 9. Get Session Key
               |
               | 10. Decrypt Message
               |     Using Session Key
               |
               v
         ORIGINAL MESSAGE
```

## 5.1. Step-by-Step Explanation

| Step | Process | Description |
|---|---|---|
| 1 | Request | Alice meminta Session Key kepada KDC. |
| 2 | Key Generation | KDC membuat Session Key secara acak. |
| 3 | Ticket Encryption | KDC mengenkripsi Ticket menggunakan Bob Master Key. |
| 4 | Wrapped Encryption | KDC mengenkripsi distribusi Session Key menggunakan Alice Master Key. |
| 5 | Decryption | Alice membuka Wrapped Data menggunakan Alice Master Key. |
| 6 | Key Extraction | Alice mendapatkan Session Key dan Ticket untuk Bob. |
| 7 | Message Encryption | Alice mengenkripsi pesan menggunakan Session Key. |
| 8 | Ticket Decryption | Bob membuka Ticket menggunakan Bob Master Key. |
| 9 | Key Extraction | Bob mendapatkan Session Key yang sama. |
| 10 | Message Decryption | Bob mendekripsi pesan menggunakan Session Key. |

**Catatan:** Apabila Bob mengirim pesan kepada Alice, prosesnya berlaku sebaliknya. Bob menjadi requester dan Alice menjadi recipient.

---

# 6. Important Concepts to Remember

| Term | Simple Explanation |
|---|---|
| Master Key | Kunci rahasia jangka panjang antara client dan KDC. |
| Session Key | Kunci sementara untuk komunikasi antara Alice dan Bob. |
| KDC | Pusat distribusi kunci. |
| Ticket | Informasi Session Key yang dienkripsi untuk penerima. |
| Wrapped Data | Data Session Key yang dienkripsi untuk pihak peminta. |
| AES-128 | Algoritma enkripsi simetris dengan kunci 128 bit. |
| IV | Nilai awal yang digunakan pada mode AES-CBC. |
| Plaintext | Pesan asli sebelum dienkripsi. |
| Ciphertext | Pesan setelah dienkripsi. |
| Decryption | Proses mengembalikan ciphertext menjadi plaintext. |

---

# 7. Important Notes and Limitations

Implementasi ini merupakan simulasi pembelajaran konsep KDC dan AES-128, bukan implementasi yang direkomendasikan untuk sistem produksi.

Beberapa hal yang perlu diperhatikan:

1. AES-CBC dalam program ini belum menggunakan mekanisme autentikasi seperti HMAC atau authenticated encryption. Oleh karena itu, integritas dan keaslian pesan belum terjamin.
2. IV yang sama digunakan untuk beberapa proses enkripsi dalam satu distribusi. Pada implementasi keamanan sebenarnya, IV CBC harus unik dan dihasilkan secara aman untuk setiap enkripsi dengan kunci yang sama.
3. Master Key disimpan langsung di dalam kode program.
4. Program Alice dan Bob belum menyinkronkan perubahan Master Key ke KDC melalui fitur `CHANGE_KEYS`.
5. AES manual digunakan untuk tujuan pembelajaran. Implementasi produksi sebaiknya menggunakan library kriptografi yang telah teruji.

---

# 8. Conclusion

Dalam sistem ini, Alice dan Bob dapat melakukan komunikasi menggunakan Session Key yang didistribusikan oleh KDC.

KDC bertindak sebagai pusat pengelolaan kunci yang memastikan Session Key diberikan kepada pihak yang tepat melalui enkripsi menggunakan Master Key masing-masing.

Sementara itu, AES-128 berfungsi sebagai algoritma kriptografi yang melindungi Ticket, distribusi Session Key, dan pesan selama proses komunikasi.

**The main idea is that KDC manages the key distribution, Alice and Bob exchange encrypted messages, and AES-128 protects the information using shared secret keys.**