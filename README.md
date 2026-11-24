# NetAuction Live

Hub lelang waktu nyata berbasis Java. Banyak peserta menawar bersamaan, barang dibuka dan ditutup otomatis sesuai jadwal, dan pemenang diumumkan seketika.

**Mata Kuliah:** Pemrograman Jaringan — Universitas Riau  
**Kelompok:** Kelompok 7 

## Tim Pengembang

| Nama | NIM |
| :--- | :--- |
| Habil Rahman Soleh | 2407110968
| Raja Muhammad Faridz Talbi | 2407111265 
| Siti Nazira |  2407110007

**Dosen Pengampu:** Linna Oktaviana Sari, S.T., M.T. & Radinal Dwiki Novendra, S.T., M.T.

## Fitur

- **Lelang terjadwal:** peserta mengajukan barang (kategori Smartphone, PC & Laptop, Aksesoris) dengan waktu mulai dan durasi. Server membuka dan menutup lelang otomatis, lalu mengumumkan pemenang.
- **Penawaran tersinkronisasi:** kelipatan tawaran minimum, anti-sniping, dan notifikasi *outbid* real-time.
- **Chat:** pesan ke semua peserta atau pribadi.
- **Katalog berkas:** unduh dan unggah berkas biner (PNG, JPG, PDF).
- **Konversi valas** dari REST API dan **live ticker harga** lewat UDP.
- **Tiga klien:** CLI, dan web (HTML).

## Lima Pilar Teknis

| Pilar | Implementasi | Berkas utama |
|---|---|---|
| 1. Concurrency | `newFixedThreadPool(32)` untuk klien, `newCachedThreadPool()` untuk transfer berkas; `ConcurrentHashMap`, `AtomicInteger`, `synchronized` per barang | `AuctionServer`, `AuctionManager`, `Auction` |
| 2. Protokol kustom dan chat (M11) | Protokol teks (CRLF) dengan 16 perintah dan pesan asinkron `ASYNC`; *background receiver thread* di klien | `ClientHandler`, `AuctionClient` |
| 3. Transfer biner | `DataInput/OutputStream`, chunk 8KB, header metadata, `shutdownOutput()` | `FileTransferHandler`, `FileRegistry` |
| 4. REST dan Gson  | `HttpURLConnection` GET ke open.er-api.com, DTO dengan `@SerializedName`, `fromJson` | `CurrencyService`, `ExchangeRateResponse` |
| 5. UDP | Ticker per detik, heartbeat PING/PONG, `setSoTimeout`, `TICK_ACK` dan retransmisi | `UdpTickerBroadcaster` |

## Arsitektur dan Port

Browser tidak dapat membuka soket TCP/UDP mentah, sehingga klien web terhubung lewat `WebGateway` (HTTP) yang meneruskan ke server memakai protokol soket yang sama dengan CLI dan Swing.

| Layanan | Protokol | Port |
|---|---|---|
| Perintah dan pesan asinkron | TCP | 5000 |
| Transfer berkas biner | TCP | 5001 |
| Live ticker dan heartbeat | UDP | 5002 |
| WebGateway (klien web) | HTTP | 8080 |

## Struktur Proyek

```text
netauction/
├── pom.xml
├── README.md
├── lib/gson-2.13.1.jar
├── catalog_files/                  # berkas katalog di server
├── downloads/                      # tujuan unduhan klien
└── src/main/
    ├── java/
    │   ├── server/   AuctionServer, ClientHandler, AuctionManager, Auction, AuctionInfo,
    │   │             FileTransferHandler, FileRegistry, UdpTickerBroadcaster
    │   ├── client/   AuctionClient (CLI), AuctionClientGUI (Swing)
    │   ├── util/     CurrencyService, ExchangeRateResponse, AppLogger
    │   └── web/      WebGateway
    └── resources/web/  index.html, style.css, app.js
```

## Cara Menjalankan

**Prasyarat:** JDK 17 atau lebih baru (diuji pada OpenJDK 21). Maven bersifat opsional.

Jalankan seluruh perintah dari folder `netauction/`. Pada Windows, ganti pemisah classpath `:` dengan `;`.

**1. Kompilasi**

```bash
# Linux / macOS
mkdir -p out
javac -cp lib/gson-2.13.1.jar -d out $(find src/main/java -name "*.java")
```

```powershell
# Windows
mkdir out
dir /s /b src\main\java\*.java > sources.txt
javac -cp "lib/gson-2.13.1.jar" -d out @sources.txt
```

**2. Server**

```bash
java -cp "out:lib/gson-2.13.1.jar" server.AuctionServer
```

**3. Klien** 
```bash
# Web: jalankan gateway, lalu buka http://localhost:8080
java -cp "out:src/main/resources:lib/gson-2.13.1.jar" web.WebGateway

# CLI (tambahkan IP server bila beda komputer)
java -cp "out:lib/gson-2.13.1.jar" client.AuctionClient [ip_server]

```

**IntelliJ IDEA:** buka folder `netauction` sebagai proyek Maven, lalu buat konfigurasi Run untuk setiap kelas dengan *working directory* `$MODULE_WORKING_DIR$` agar `catalog_files/` terbaca.

## Protokol Singkat (TCP 5000)

Satu perintah per baris, berakhiran CRLF. Balasan berformat `<kode> SUCCESS|ERROR <pesan>`.

| Perintah | Fungsi |
|---|---|
| `LOGIN <nama>` | Masuk (3-15 karakter: huruf, angka, garis bawah) |
| `LIST`, `USERS`, `STATUS <id>`, `HISTORY <id>` | Daftar barang, peserta aktif, detail barang, riwayat tawaran |
| `BID <id> <nominal>` | Menawar |
| `SELL <nama> <harga> [berkas] [@kategori] [+S<menit>] [+D<menit>]` | Mengajukan barang |
| `DELLOT <id>` | Menghapus barang milik sendiri (sebelum ada tawaran) |
| `BROADCAST <pesan>`, `PRIVATE <nama> <pesan>` | Chat |
| `CATALOG`, `DOWNLOAD <nama>`, `UPTICKET`, `DELFILE <nama>` | Katalog berkas |
| `CONVERT <dari> <ke> <nominal>` | Konversi valas |
| `EXIT` | Keluar |

Contoh: `SELL Kamera 450000 foto.png @aksesoris +S5 +D30` (kategori `@smartphone`, `@pclaptop`, `@aksesoris`; mulai `+S` 0/1/5/15/30/60 menit lagi; durasi `+D` 1/5/15/30/60 menit).

**Pesan asinkron dari server:** `OUTBID`, `BID_UPDATE`, `NEW_LOT`, `LOT_REMOVED`, `FILE_REMOVED`, `AUCTION_START`, `AUCTION_END`, `WINNER`, `SOLD`, `UNSOLD`, `BROADCAST`, `PRIVATE`, `SYSTEM`.

**Kode respons:** 200 berhasil, 400 format salah, 401 belum login, 403 tidak berwenang, 404 tidak ditemukan, 408 tidak aktif, 409 konflik aturan, 503 server penuh atau layanan kurs tidak tersedia.

**Transfer biner (TCP 5001):** mode `DOWNLOAD`, `UPLOAD`, dan `UPLOAD_AUTH` (dengan tiket dari `UPTICKET` sehingga pengunggah tercatat sebagai pemilik). Unggahan dibatasi pada PNG, JPG, PDF, maksimal 5 MB, dan isi berkas harus cocok dengan tipenya.

## Aturan Lelang

- **Siklus barang:** menunggu, berjalan, selesai. Penjadwal server membuka dan menutup barang otomatis.
- **Tawaran:** pertama minimal sama dengan harga awal; berikutnya minimal tawaran tertinggi + kelipatan (Rp 10.000 bila harga di bawah Rp 1 juta, Rp 50.000 bila di atasnya).
- **Anti-sniping:** tawaran di 30 detik terakhir memperpanjang waktu menjadi 30 detik lagi (maksimal 10 kali per barang).
- **Pemenang:** penawar tertinggi saat waktu habis; tanpa tawaran berarti tidak terjual.
- **Penjual** tidak boleh menawar barangnya sendiri.
- **Kuota:** maksimal 20 barang aktif, 3 barang aktif per penjual.
- **Barang bawaan** langsung berjalan selama 30 menit sejak server menyala.

## Antarmuka Web

| Alamat | Isi |
|---|---|
| `#/` | Daftar barang dengan filter kategori dan status, hitung mundur |
| `#/barang/<id>` | Halaman penawaran: harga tertinggi, kolom penawaran dengan tombol `-10x`, `-`, `+`, `+10x`, riwayat tawaran |
| `#/jual` | Halaman ajukan barang dengan jadwal mulai dan durasi |

## Catatan

- Server melayani maksimal 32 klien bersamaan; koneksi ke-33 ditolak dengan `503`.
- `auction.log` dan `file_owners.properties` dibuat otomatis oleh server saat berjalan.

