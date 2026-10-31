# README – Key Distribution Center (KDC)

## 1. Project Overview

### 1.1. What is KDC?

Key Distribution Center (KDC) merupakan server pusat yang bertugas mengelola dan mendistribusikan kunci enkripsi kepada client yang ingin melakukan komunikasi secara aman.

Dalam program ini, KDC berperan sebagai pihak ketiga yang dipercaya oleh Alice dan Bob.

KDC memiliki beberapa tugas utama:

1. Menyimpan Master Key milik Alice dan Bob.
2. Menerima permintaan Session Key dari Alice atau Bob.
3. Menghasilkan Session Key secara acak.
4. Membuat Ticket untuk penerima.
5. Mengenkripsi informasi Session Key menggunakan Master Key masing-masing pihak.
6. Mengirimkan hasil distribusi kunci kepada client yang meminta.

### 1.2. How KDC Works

Contoh ketika Alice ingin mengirim pesan kepada Bob:

- Alice meminta Session Key kepada KDC.
- KDC menghasilkan Session Key baru.
- KDC membuat Ticket yang dienkripsi menggunakan Bob Master Key.
- KDC membungkus Session Key dan Ticket menggunakan Alice Master Key.
- KDC mengirimkan hasilnya kepada Alice.
- Alice meneruskan Ticket kepada Bob.

Dengan mekanisme tersebut, Alice dan Bob dapat memperoleh Session Key yang sama tanpa harus membagikan Master Key mereka secara langsung.

---

# 2. Import Library

```python
import os
import json
import socket
import secrets
from aes_manual import AES128Manual
```

Bagian ini mengimpor library yang dibutuhkan oleh KDC.

| Library | Function |
|---|---|
| `os` | Mengambil konfigurasi jaringan melalui environment variables. |
| `json` | Mengelola data komunikasi dalam format JSON. |
| `socket` | Membuat komunikasi jaringan antara KDC dan client. |
| `secrets` | Menghasilkan nilai acak yang sesuai untuk kebutuhan kriptografi. |
| `AES128Manual` | Melakukan enkripsi dan dekripsi menggunakan AES-128 manual. |

**Penjelasan penting:**

Library `secrets` digunakan untuk menghasilkan Session Key dan IV secara acak, sedangkan `AES128Manual` digunakan untuk mengamankan data yang didistribusikan oleh KDC.

---

# 3. Network Configuration

```python
HOST = os.getenv("KDC_HOST", "0.0.0.0")
PORT = int(os.getenv("KDC_PORT", "5000"))
```

Bagian ini digunakan untuk menentukan alamat jaringan dan port server KDC.

### Penjelasan:

**1. HOST**

```python
HOST = os.getenv("KDC_HOST", "0.0.0.0")
```

Menentukan alamat interface jaringan yang digunakan KDC.

Nilai `0.0.0.0` berarti server menerima koneksi melalui seluruh interface IPv4 yang tersedia.

**2. PORT**

```python
PORT = int(os.getenv("KDC_PORT", "5000"))
```

Menentukan port komunikasi server.

Dalam program ini, port default yang digunakan adalah `5000`.

**Kesimpulan:** KDC menggunakan alamat dan port tersebut untuk menerima permintaan dari Alice maupun Bob.

---

# 4. Master Key Configuration

```python
ALICE_MASTER_KEY = b"AliceMasterKey12"
BOB_MASTER_KEY = b"BobMasterKey1234"
```

Bagian ini menyimpan Master Key milik Alice dan Bob di dalam server KDC.

### 4.1. What is Master Key?

Master Key merupakan kunci rahasia yang digunakan untuk mengamankan distribusi Session Key antara client dan KDC.

Setiap client memiliki Master Key yang berbeda.

| Master Key | Owner | Function |
|---|---|---|
| Alice Master Key | Alice dan KDC | Mengamankan distribusi Session Key kepada Alice. |
| Bob Master Key | Bob dan KDC | Mengamankan Ticket untuk Bob. |

Kedua Master Key memiliki panjang 16 byte sehingga sesuai dengan kebutuhan AES-128.

### 4.2. Why Does KDC Store Both Keys?

Karena KDC bertugas mendistribusikan Session Key kepada Alice dan Bob.

KDC harus mengetahui Master Key masing-masing pihak agar dapat mengenkripsi informasi untuk pihak yang tepat.

Contohnya:

Jika Alice ingin berkomunikasi dengan Bob:
- KDC menggunakan Alice Master Key untuk mengamankan informasi yang dikirim kepada Alice.
- KDC menggunakan Bob Master Key untuk mengamankan Ticket yang nantinya diterima Bob.

---

# 5. Send JSON and Receive JSON

KDC menggunakan dua fungsi untuk melakukan pertukaran data dengan client.

## 5.1. Send JSON

```python
def send_json(conn, data):
    conn.sendall(json.dumps(data).encode("utf-8") + b"\n")
```

Fungsi ini digunakan untuk mengirimkan data dari KDC kepada client.

Penjelasan setiap bagian:

- `json.dumps(data)`: Mengubah dictionary Python menjadi JSON.
- `.encode("utf-8")`: Mengubah teks JSON menjadi bytes.
- `conn.sendall()`: Mengirim seluruh data melalui koneksi TCP.
- `b"\n"`: Menambahkan newline sebagai penanda akhir pesan.

Contoh data yang dikirim:

```python
{
    "status": "ok",
    "iv_hex": "example",
    "wrapped": "encrypted_data"
}
```

## 5.2. Receive JSON

```python
def recv_json(conn):
    data = bytearray()

    while not data.endswith(b"\n"):
        chunk = conn.recv(4096)

        if not chunk:
            break

        data.extend(chunk)

        if len(data) > 1_000_000:
            raise ValueError("Request too large")

    if not data:
        raise ValueError("Empty request")

    return json.loads(data.decode("utf-8"))
```

Fungsi ini digunakan untuk menerima request dari Alice atau Bob.

Penjelasan:

**1. Menyiapkan tempat penyimpanan data**

```python
data = bytearray()
```

Membuat tempat untuk menampung data yang diterima.

**2. Menerima data**

```python
chunk = conn.recv(4096)
```

Menerima data dari koneksi TCP dengan ukuran maksimal 4096 byte dalam satu pembacaan.

**3. Memeriksa koneksi**

```python
if not chunk:
    break
```

Jika tidak ada data yang diterima karena koneksi terputus, perulangan dihentikan.

**4. Menambahkan data**

```python
data.extend(chunk)
```

Memasukkan data yang diterima ke dalam buffer.

**5. Membatasi ukuran request**

```python
if len(data) > 1_000_000:
    raise ValueError("Request too large")
```

Mencegah request yang terlalu besar diproses oleh KDC.

**6. Memeriksa request kosong**

```python
if not data:
    raise ValueError("Empty request")
```

Jika tidak ada data, program menghasilkan error.

**7. Mengubah JSON menjadi dictionary**

```python
return json.loads(data.decode("utf-8"))
```

Data yang diterima dikembalikan menjadi objek Python agar dapat diproses oleh KDC.

---

# 6. Make Distribution – Core Function of KDC

```python
def make_distribution(sender: str, target: str):
```

Fungsi `make_distribution()` merupakan bagian paling penting dalam program KDC.

Fungsi ini bertanggung jawab untuk membuat Session Key dan mendistribusikannya secara terenkripsi kepada pihak yang meminta.

Prosesnya terdiri dari lima tahapan utama:

1. Menentukan Master Key pengirim dan penerima.
2. Membuat Session Key dan IV.
3. Membuat Ticket untuk penerima.
4. Mengenkripsi distribusi Session Key untuk pengirim.
5. Mengembalikan hasil distribusi kepada client.

## 6.1. Determine Requester Key and Recipient Key

```python
if sender == "Alice" and target == "Bob":
    requester_key = ALICE_MASTER_KEY
    recipient_key = BOB_MASTER_KEY

elif sender == "Bob" and target == "Alice":
    requester_key = BOB_MASTER_KEY
    recipient_key = ALICE_MASTER_KEY

else:
    raise ValueError(
        "Only Alice->Bob or Bob->Alice is supported"
    )
```

Bagian ini digunakan untuk menentukan Master Key berdasarkan identitas pengirim dan penerima.

Ada dua kondisi yang didukung:

**Condition 1: Alice to Bob**

Jika Alice ingin mengirim pesan kepada Bob:

```python
requester_key = ALICE_MASTER_KEY
recipient_key = BOB_MASTER_KEY
```

Artinya:
- Alice merupakan pihak yang meminta Session Key.
- Bob merupakan pihak yang menerima pesan.

**Condition 2: Bob to Alice**

Jika Bob ingin mengirim pesan kepada Alice:

```python
requester_key = BOB_MASTER_KEY
recipient_key = ALICE_MASTER_KEY
```

Artinya:
- Bob merupakan pihak yang meminta Session Key.
- Alice merupakan pihak yang menerima pesan.

Jika identitas pengirim dan penerima tidak sesuai dengan dua kondisi tersebut, KDC akan menghasilkan error.

**Kesimpulan:** KDC harus mengetahui siapa yang meminta dan siapa yang menerima agar dapat memilih Master Key yang sesuai.

## 6.2. Generate Session Key and IV

```python
session_key = secrets.token_bytes(16)

iv = secrets.token_bytes(16)

iv_hex = iv.hex()
```

Bagian ini digunakan untuk menghasilkan Session Key dan Initialization Vector.

### A. Session Key

```python
session_key = secrets.token_bytes(16)
```

KDC menghasilkan Session Key secara acak sepanjang 16 byte atau 128 bit.

Session Key akan digunakan oleh Alice dan Bob untuk mengenkripsi serta mendekripsi pesan.

Mengapa menggunakan `secrets`?

Karena `secrets` dirancang untuk menghasilkan nilai acak yang sesuai untuk kebutuhan keamanan kriptografi.

### B. Initialization Vector (IV)

```python
iv = secrets.token_bytes(16)
```

KDC juga menghasilkan IV sepanjang 16 byte.

IV digunakan dalam mode AES-CBC untuk membantu proses enkripsi setiap blok data.

IV bukan merupakan Secret Key, sehingga tidak harus dirahasiakan seperti Master Key atau Session Key.

### C. Hexadecimal Conversion

```python
iv_hex = iv.hex()
```

Mengubah IV dari bytes menjadi format hexadecimal.

Tujuannya agar lebih mudah disimpan dan dikirim dalam format teks.

**Kesimpulan:** Pada tahap ini KDC sudah memiliki Session Key dan IV yang akan digunakan dalam proses komunikasi.

## 6.3. Create Ticket for Recipient

```python
ticket_payload = f"{sender}|{target}|{session_key.hex()}|{iv_hex}"

ticket_for_target = AES128Manual(recipient_key).encrypt_cbc(
    ticket_payload,
    iv
)
```

Bagian ini digunakan untuk membuat Ticket yang ditujukan kepada penerima.

### A. Create Ticket Payload

```python
ticket_payload = f"{sender}|{target}|{session_key.hex()}|{iv_hex}"
```

KDC menyusun informasi yang akan dimasukkan ke dalam Ticket.

Isi Ticket:

| Data | Function |
|---|---|
| `sender` | Identitas pengirim. |
| `target` | Identitas penerima. |
| `session_key.hex()` | Session Key dalam format hexadecimal. |
| `iv_hex` | Initialization Vector dalam format hexadecimal. |

Pemisah `|` digunakan agar informasi dapat dipisahkan kembali ketika Ticket didekripsi.

Contoh ilustrasi:

```text
Alice|Bob|SESSION_KEY|IV
```

### B. Encrypt Ticket

```python
ticket_for_target = AES128Manual(recipient_key).encrypt_cbc(
    ticket_payload,
    iv
)
```

KDC mengenkripsi Ticket menggunakan Master Key milik penerima.

Contoh ketika Alice ingin mengirim pesan kepada Bob:

- Recipient Key = Bob Master Key.
- Ticket dienkripsi menggunakan Bob Master Key.
- Alice tidak perlu membuka isi Ticket.
- Bob dapat membuka Ticket menggunakan Master Key miliknya.

Mengapa Ticket harus dienkripsi menggunakan Master Key penerima?

Agar informasi Session Key yang terdapat dalam Ticket hanya dapat dibaca oleh pihak yang memiliki Master Key penerima.

**Kesimpulan:** Ticket merupakan informasi yang disiapkan KDC untuk diberikan kepada penerima agar penerima dapat memperoleh Session Key.

## 6.4. Wrap Session Key for Requester

```python
requester_payload = f"{session_key.hex()}|{iv_hex}|{ticket_for_target}"

wrapped_for_requester = AES128Manual(requester_key).encrypt_cbc(
    requester_payload,
    iv
)
```

Bagian ini digunakan untuk mengamankan informasi Session Key yang dikirimkan kepada pihak peminta.

### A. Create Requester Payload

```python
requester_payload = f"{session_key.hex()}|{iv_hex}|{ticket_for_target}"
```

KDC menggabungkan tiga informasi:

1. Session Key.
2. IV.
3. Ticket untuk penerima.

Contoh ilustrasi:

```text
SESSION_KEY|IV|TICKET_FOR_BOB
```

### B. Encrypt Requester Payload

```python
wrapped_for_requester = AES128Manual(requester_key).encrypt_cbc(
    requester_payload,
    iv
)
```

KDC mengenkripsi seluruh informasi tersebut menggunakan Requester Master Key.

Contoh Alice mengirim pesan kepada Bob:

- Requester Key = Alice Master Key.
- Payload dienkripsi menggunakan Alice Master Key.
- Alice dapat membuka informasi tersebut menggunakan Master Key miliknya.

Setelah berhasil membuka Wrapped Data, Alice mendapatkan Session Key dan Ticket untuk Bob.

**Mengapa disebut Wrapped Data?**

Karena Session Key dan informasi pendukungnya dibungkus dalam bentuk ciphertext menggunakan Master Key milik pihak peminta.

## 6.5. Return Distribution Result

```python
return {
    "status": "ok",
    "iv_hex": iv_hex,
    "wrapped": wrapped_for_requester
}
```

Bagian ini mengembalikan hasil distribusi kepada fungsi yang memanggil `make_distribution()`.

Informasi yang dikembalikan:

| Field | Description |
|---|---|
| `status` | Menunjukkan status permintaan. |
| `iv_hex` | IV dalam format hexadecimal. |
| `wrapped` | Hasil enkripsi Requester Payload. |

Respons tersebut kemudian dikirimkan kepada Alice atau Bob.

### Summary of Make Distribution

Ketika Alice meminta Session Key untuk berkomunikasi dengan Bob, KDC melakukan:

1. Memilih Alice Master Key dan Bob Master Key.
2. Menghasilkan Session Key dan IV.
3. Membuat Ticket yang berisi Session Key.
4. Mengenkripsi Ticket menggunakan Bob Master Key.
5. Membungkus Session Key, IV, dan Ticket.
6. Mengenkripsi Wrapped Data menggunakan Alice Master Key.
7. Mengirimkan hasilnya kepada Alice.

---

# 7. Handle Client – Processing Client Requests

```python
def handle_client(conn, address):
```

Fungsi ini bertugas menangani permintaan yang masuk ke server KDC.

Setiap kali Alice atau Bob terhubung, KDC akan membaca request dan menentukan tindakan yang harus dilakukan.

## 7.1. Initialize Request Processing

```python
global ALICE_MASTER_KEY, BOB_MASTER_KEY

with conn:
    try:
        req = recv_json(conn)
```

Penjelasan:

- `global`: Memungkinkan fungsi mengakses dan mengubah variabel Master Key global.
- `with conn`: Mengelola koneksi client.
- `try`: Menjalankan proses dan menangani kemungkinan error.
- `recv_json(conn)`: Membaca request yang dikirimkan client.

## 7.2. Change Master Keys

```python
if req.get("type") == "CHANGE_KEYS":
    alice_key = req.get("alice_key", "").encode(
        "ascii", errors="strict"
    )

    bob_key = req.get("bob_key", "").encode(
        "ascii", errors="strict"
    )
```

Bagian ini digunakan jika KDC menerima permintaan untuk mengganti Master Key.

KDC mengambil Master Key baru milik Alice dan Bob dari request.

### Validate Key Length

```python
if len(alice_key) != 16 or len(bob_key) != 16:
    raise ValueError(
        "Both master keys must be exactly 16 ASCII bytes"
    )
```

KDC memeriksa apakah kedua Master Key memiliki panjang tepat 16 byte.

Jika tidak sesuai, program menghasilkan error.

### Update Master Keys

```python
ALICE_MASTER_KEY, BOB_MASTER_KEY = alice_key, bob_key
```

Jika validasi berhasil, KDC memperbarui Master Key yang disimpan.

### Send Confirmation

```python
send_json(conn, {
    "status": "ok",
    "msg": "Master keys updated"
})
```

KDC mengirimkan konfirmasi bahwa Master Key berhasil diperbarui.

**Catatan:** Fitur `CHANGE_KEYS` sudah tersedia pada KDC, tetapi kode Alice dan Bob yang digunakan saat ini belum mengirimkan request perubahan Master Key ke KDC.

## 7.3. Validate Request Type

```python
if req.get("type") != "REQUEST_SESSION_KEY":
    raise ValueError("Unsupported request type")
```

Jika request bukan `REQUEST_SESSION_KEY` dan bukan `CHANGE_KEYS`, KDC menolak permintaan tersebut.

Tujuannya adalah membatasi jenis request yang didukung program.

## 7.4. Process Session Key Request

```python
sender, target = req.get("from"), req.get("to")

response = make_distribution(sender, target)

send_json(conn, response)
```

Bagian ini merupakan penghubung antara request client dengan fungsi distribusi kunci.

Tahapannya:

1. KDC membaca identitas pengirim.
2. KDC membaca identitas penerima.
3. KDC memanggil `make_distribution()`.
4. KDC mendapatkan hasil distribusi.
5. KDC mengirimkan respons kepada client.

Contoh request dari Alice:

```python
{
    "type": "REQUEST_SESSION_KEY",
    "from": "Alice",
    "to": "Bob"
}
```

KDC akan memproses permintaan tersebut dan membuat Session Key untuk komunikasi Alice dengan Bob.

## 7.5. Error Handling

```python
except Exception as exc:
    try:
        send_json(conn, {
            "status": "error",
            "message": str(exc)
        })
    except Exception:
        pass

    print(f"[KDC] Error from {address}: {exc}")
```

Bagian ini menangani error yang terjadi selama proses komunikasi.

Jika terjadi kesalahan:
- KDC mencoba mengirimkan informasi error kepada client.
- KDC menampilkan pesan error pada terminal.
- Program tidak perlu langsung berhenti hanya karena satu request mengalami kesalahan.

---

# 8. Main Program – Starting KDC Server

```python
def main():
    with socket.socket(
        socket.AF_INET,
        socket.SOCK_STREAM
    ) as server:
```

Bagian ini merupakan program utama yang menjalankan KDC sebagai server.

### 8.1. Create Server Socket

```python
socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

Membuat socket menggunakan:
- `AF_INET`: IPv4.
- `SOCK_STREAM`: TCP.

### 8.2. Set Socket Option

```python
server.setsockopt(
    socket.SOL_SOCKET,
    socket.SO_REUSEADDR,
    1
)
```

Mengizinkan penggunaan kembali alamat socket setelah server sebelumnya berhenti.

Hal ini membantu ketika server dijalankan kembali menggunakan alamat dan port yang sama.

### 8.3. Bind Server Address

```python
server.bind((HOST, PORT))
```

Menghubungkan socket dengan alamat IP dan port KDC.

### 8.4. Start Listening

```python
server.listen()
```

Membuat server mulai menunggu koneksi dari client.

### 8.5. Accept Client Connection

```python
while True:
    conn, address = server.accept()
    handle_client(conn, address)
```

KDC terus menunggu koneksi masuk.

Ketika Alice atau Bob menghubungkan diri:
1. KDC menerima koneksi.
2. KDC mendapatkan alamat client.
3. KDC menjalankan `handle_client()`.
4. Request client diproses.

---

# 9. Complete KDC Communication Flow

Berikut gambaran proses distribusi Session Key ketika Alice ingin berkomunikasi dengan Bob.

```text
              ALICE
                |
                |
       REQUEST_SESSION_KEY
                |
                v
          +-----------+
          |    KDC    |
          +-----------+
                |
                v
       Determine Master Keys
                |
                v
       Generate Session Key
                |
                v
           Generate IV
                |
                v
        Create Ticket Payload
                |
                v
      Encrypt Ticket Using
         Bob Master Key
                |
                v
      Create Requester Payload
                |
                v
      Encrypt Using Alice
           Master Key
                |
                v
       Return Wrapped Data
                |
                v
              ALICE
                |
                v
       Decrypt Using Alice
           Master Key
                |
                v
       Obtain Session Key
                |
                v
       Send Ticket to Bob
                |
                v
               BOB
                |
                v
       Decrypt Ticket Using
          Bob Master Key
                |
                v
       Obtain Session Key
```

## 9.1. Communication Data Comparison

| Information | Encryption Key | Destination |
|---|---|---|
| Ticket | Bob Master Key | Bob melalui Alice |
| Wrapped Data | Alice Master Key | Alice |
| Message | Session Key | Bob |

Perlu diperhatikan bahwa Ticket dan Wrapped Data memiliki tujuan yang berbeda.

- Ticket dibuat agar penerima dapat memperoleh Session Key.
- Wrapped Data dibuat agar pihak peminta dapat memperoleh Session Key dan Ticket.

---

# 10. Important Concepts to Remember

| Term | Explanation |
|---|---|
| KDC | Server pusat untuk distribusi kunci. |
| Master Key | Kunci rahasia antara client dan KDC. |
| Session Key | Kunci sementara untuk komunikasi Alice dan Bob. |
| Requester | Pihak yang meminta Session Key. |
| Recipient | Pihak yang menerima pesan. |
| Ticket | Informasi terenkripsi yang disiapkan untuk penerima. |
| Wrapped Data | Informasi Session Key yang dienkripsi untuk requester. |
| IV | Initialization Vector yang digunakan AES-CBC. |
| AES-128 | Algoritma enkripsi simetris dengan kunci 128 bit. |
| TCP Socket | Media komunikasi antara KDC dan client. |

---

# 11. Important Notes and Limitations

Program ini merupakan simulasi pembelajaran konsep Key Distribution Center dan symmetric key cryptography.

Beberapa keterbatasan implementasi yang perlu diketahui:

1. Master Key masih disimpan langsung di dalam kode.
2. Program belum menerapkan autentikasi client yang kuat.
3. Komunikasi TCP belum dilindungi TLS.
4. Ticket belum memiliki mekanisme expiry time.
5. Session Key belum memiliki mekanisme penghapusan atau pembatasan masa berlaku secara khusus.
6. AES-CBC belum menyediakan autentikasi dan integritas pesan.
7. Fitur perubahan Master Key di KDC belum terhubung dengan menu perubahan Master Key pada Alice dan Bob.

Untuk sistem produksi, diperlukan mekanisme autentikasi, perlindungan integritas, pengelolaan kunci yang aman, serta penggunaan protokol dan library kriptografi yang telah teruji.

---

# 12. Conclusion

Key Distribution Center merupakan komponen utama dalam sistem komunikasi ini karena bertugas mengelola dan mendistribusikan Session Key kepada Alice dan Bob.

Melalui fungsi `make_distribution()`, KDC menghasilkan Session Key, membuat Ticket untuk penerima, dan mengamankan informasi distribusi menggunakan Master Key masing-masing pihak.

AES-128 digunakan untuk melindungi informasi tersebut sehingga Session Key tidak dikirimkan sebagai plaintext melalui jaringan.

**The main idea is that KDC acts as a trusted third party that generates and securely distributes session keys, allowing Alice and Bob to communicate using AES-128 encryption.**
