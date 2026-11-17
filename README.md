# DVWA Lab dengan Docker Compose

Dokumentasi ini menjelaskan cara menyiapkan, menjalankan, menggunakan,
memelihara, dan melakukan troubleshooting **Damn Vulnerable Web
Application (DVWA)** menggunakan Docker Compose.

DVWA adalah aplikasi web yang sengaja dibuat rentan untuk membantu
pembelajaran keamanan aplikasi web. Lab ini cocok untuk demonstrasi
instruktur dan praktikum terkontrol, misalnya SQL Injection, Cross-Site
Scripting (XSS), Command Injection, dan kelemahan upload file.

> **Peringatan keamanan:** DVWA memang dirancang memiliki kerentanan.
> Jalankan hanya di lingkungan lab terisolasi. Jangan mengeksposnya ke
> internet, jangan gunakan data asli, dan jangan memasangnya pada server
> produksi.

------------------------------------------------------------------------

## Daftar Isi

-   [1. Gambaran Arsitektur](#1-gambaran-arsitektur)
-   [2. Prasyarat](#2-prasyarat)
-   [3. Struktur Direktori](#3-struktur-direktori)
-   [4. Konfigurasi Docker Compose](#4-konfigurasi-docker-compose)
-   [5. Menjalankan Lab](#5-menjalankan-lab)
-   [6. Login dan Inisialisasi DVWA](#6-login-dan-inisialisasi-dvwa)
-   [7. Konfigurasi Tingkat Keamanan](#7-konfigurasi-tingkat-keamanan)
-   [8. Modul Praktikum](#8-modul-praktikum)
-   [9. Pengelolaan Container](#9-pengelolaan-container)
-   [10. Backup dan Pemulihan
    Database](#10-backup-dan-pemulihan-database)
-   [11. Troubleshooting](#11-troubleshooting)
-   [12. Penggunaan untuk Kelas](#12-penggunaan-untuk-kelas)
-   [13. Batasan dan Catatan](#13-batasan-dan-catatan)

------------------------------------------------------------------------

## 1. Gambaran Arsitektur

Lab terdiri dari dua container utama:

  -----------------------------------------------------------------------
  Komponen                            Fungsi
  ----------------------------------- -----------------------------------
  `dvwa`                              Menjalankan aplikasi DVWA melalui
                                      web server dan PHP

  `db`                                Menyimpan data DVWA menggunakan
                                      MariaDB

  `dvwa-db-data`                      Named volume Docker untuk menyimpan
                                      data database secara persisten

  `dvwa-network`                      Jaringan bridge internal untuk
                                      komunikasi antarkontainer
  -----------------------------------------------------------------------

Alur komunikasi:

``` text
Browser
   |
   | http://localhost:8085
   v
Host port 8085
   |
   v
DVWA container (port 80)
   |
   | Docker network: dvwa-network
   | DB host: db
   v
MariaDB container (port 3306)
   |
   v
Named volume: dvwa-db-data
```

Port `8085` pada host diteruskan ke port `80` di container DVWA. MariaDB
tidak dipublikasikan ke host karena hanya perlu diakses oleh DVWA
melalui jaringan Docker.

## 2. Prasyarat

Pastikan perangkat sudah memiliki:

-   Docker Engine atau Docker Desktop.
-   Docker Compose plugin (perintah `docker compose`).
-   Browser modern.
-   Terminal atau command line.

Periksa instalasi:

``` bash
docker --version
docker compose version
```

Jika perintah `docker compose` tidak tersedia, pasang atau aktifkan
Docker Compose sesuai sistem operasi dan distribusi Docker yang
digunakan.

## 3. Struktur Direktori

Buat direktori proyek:

``` bash
mkdir -p dvwa-lab
cd dvwa-lab
```

Struktur yang disarankan:

``` text
dvwa-lab/
├── docker-compose.yml
└── README.md
```

## 4. Konfigurasi Docker Compose

Buat file `docker-compose.yml`:

``` yaml
services:
  dvwa:
    image: ghcr.io/digininja/dvwa:latest
    container_name: dvwa
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "8085:80"
    environment:
      DB_SERVER: db
      DB_USER: dvwa
      DB_PASSWORD: dvwa_password
      DB_NAME: dvwa
    networks:
      - dvwa-network

  db:
    image: mariadb:10.11
    container_name: dvwa-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: dvwa
      MYSQL_USER: dvwa
      MYSQL_PASSWORD: dvwa_password
    volumes:
      - dvwa-db-data:/var/lib/mysql
    networks:
      - dvwa-network

volumes:
  dvwa-db-data:

networks:
  dvwa-network:
    driver: bridge
```

### Penjelasan konfigurasi

#### Service `dvwa`

-   `image`: image DVWA yang digunakan.
-   `container_name`: memberi nama container `dvwa` agar mudah
    diperiksa.
-   `restart: unless-stopped`: container akan dijalankan kembali ketika
    Docker aktif, kecuali dihentikan secara manual.
-   `depends_on`: mengatur urutan start agar service database dimulai
    sebelum DVWA. Ini **tidak menjamin** database sudah siap menerima
    koneksi.
-   `ports`: memetakan port host `8085` ke port container `80`.
-   `DB_SERVER`: hostname database pada jaringan Docker. Nama service
    `db` dapat digunakan sebagai hostname.
-   `DB_USER`, `DB_PASSWORD`, `DB_NAME`: kredensial dan nama database
    yang digunakan DVWA.

#### Service `db`

-   `image: mariadb:10.11`: menggunakan MariaDB seri 10.11.
-   `MYSQL_ROOT_PASSWORD`: password akun administratif database.
-   `MYSQL_DATABASE`: database yang dibuat saat inisialisasi pertama.
-   `MYSQL_USER` dan `MYSQL_PASSWORD`: akun aplikasi yang dibuat saat
    inisialisasi pertama.
-   `volumes`: menyimpan file database di named volume agar data tetap
    ada meskipun container dibuat ulang.

#### Network dan volume

-   `dvwa-network` memungkinkan DVWA menghubungi database melalui nama
    service `db`.
-   `dvwa-db-data` menyimpan data database di luar filesystem container.

### Catatan kredensial

Ada dua jenis kredensial yang berbeda:

  Kegunaan                   Username   Password
  -------------------------- ---------- -----------------
  Login aplikasi DVWA        `admin`    `password`
  Login MariaDB untuk DVWA   `dvwa`     `dvwa_password`
  Administrator MariaDB      `root`     `root_password`

Kredensial di atas adalah contoh untuk lab lokal. Ganti sebelum
digunakan pada lingkungan bersama, simpan rahasia dengan aman, dan
jangan commit password nyata ke repositori publik.

**Penting tentang volume:** variabel `MYSQL_DATABASE`, `MYSQL_USER`, dan
password inisialisasi MariaDB hanya diterapkan ketika direktori data
database masih kosong. Mengubah nilai environment setelah volume sudah
berisi data tidak otomatis mengubah akun atau password yang tersimpan.

## 5. Menjalankan Lab

Dari direktori proyek, jalankan:

``` bash
docker compose up -d
```

Perintah tersebut akan mengunduh image yang diperlukan jika belum
tersedia, membuat network dan volume, lalu menjalankan container di
background.

Periksa status:

``` bash
docker compose ps
```

Pantau log kedua service:

``` bash
docker compose logs -f
```

Atau per service:

``` bash
docker compose logs -f dvwa
docker compose logs -f db
```

Jika container berjalan, buka:

**http://localhost:8085**

Jika port `8085` sudah digunakan aplikasi lain, ubah bagian `ports`,
misalnya:

``` yaml
ports:
  - "8086:80"
```

Kemudian terapkan konfigurasi:

``` bash
docker compose up -d
```

Akses aplikasi melalui `http://localhost:8086`.

## 6. Login dan Inisialisasi DVWA

### 6.1 Login awal

Gunakan kredensial default DVWA:

``` text
Username: admin
Password: password
```

Jangan tertukar dengan kredensial MariaDB. Username dan password
database bukan kredensial untuk halaman login DVWA.

### 6.2 Buat atau reset database DVWA

Setelah masuk ke antarmuka DVWA:

1.  Buka menu **Setup**.
2.  Temukan bagian **Create / Reset Database**.
3.  Klik tombol **Create / Reset Database**.
4.  Tunggu proses selesai dan perhatikan apakah ada pesan error.
5.  Kembali ke halaman login jika diperlukan, lalu coba login dengan
    kredensial default.

Tindakan reset dapat menghapus atau membangun ulang data aplikasi.
Gunakan hanya pada database lab.

### 6.3 Jika login gagal setelah reset

Lakukan pemeriksaan berikut secara berurutan:

**A. Periksa status container**

``` bash
docker compose ps
```

Pastikan container `dvwa` dan `dvwa-db` berstatus berjalan (`Up`).

**B. Periksa log aplikasi**

``` bash
docker logs dvwa --tail 100
```

Cari pesan terkait koneksi database, konfigurasi PHP, session, atau
error aplikasi.

**C. Periksa log database**

``` bash
docker logs dvwa-db --tail 100
```

Pastikan MariaDB berhasil melakukan inisialisasi dan tidak mengalami
error.

**D. Periksa environment DVWA**

``` bash
docker exec dvwa env | grep '^DB_'
```

Nilai yang diharapkan:

``` text
DB_SERVER=db
DB_USER=dvwa
DB_PASSWORD=dvwa_password
DB_NAME=dvwa
```

**E. Periksa tabel database**

``` bash
docker exec -it dvwa-db mariadb -u dvwa -pdvwa_password dvwa -e "SHOW TABLES;"
```

Setelah database DVWA berhasil dibuat, seharusnya tabel aplikasi
terlihat. Salah satu tabel yang diharapkan adalah `users`.

**F. Periksa akun aplikasi**

``` bash
docker exec -it dvwa-db mariadb -u dvwa -pdvwa_password dvwa -e "SELECT user FROM users;"
```

Pada instalasi default, akun `admin` seharusnya tersedia setelah
database DVWA berhasil diinisialisasi.

Hindari menampilkan atau membagikan hash password dalam laporan
diagnostik yang tidak memerlukannya.

**G. Bersihkan cookie/session browser**

-   Buka alamat DVWA yang benar.
-   Hapus cookie untuk `localhost` jika session bermasalah.
-   Coba melalui jendela Incognito/Private.
-   Pastikan browser tidak membuka alamat atau port lama.

Jika masih gagal, kumpulkan output `docker compose ps`, log DVWA, log
MariaDB, hasil `SHOW TABLES`, dan pesan error yang tampil di browser.
Jangan langsung menghapus volume karena data diagnostik mungkin masih
diperlukan.

## 7. Konfigurasi Tingkat Keamanan

DVWA menyediakan beberapa tingkat keamanan untuk memperlihatkan
perbedaan antara implementasi rentan dan implementasi yang lebih aman.

Setelah login:

1.  Buka menu **DVWA Security**.
2.  Pilih tingkat keamanan yang ingin digunakan.
3.  Simpan pengaturan melalui tombol yang tersedia.

Gambaran umum:

  Level        Tujuan pembelajaran
  ------------ ------------------------------------------------------
  Low          Memahami perilaku aplikasi dengan proteksi minimal
  Medium       Mengamati proteksi tambahan yang belum tentu memadai
  High         Mempelajari validasi dan kontrol yang lebih ketat
  Impossible   Membandingkan implementasi yang dirancang lebih aman

Nama dan perilaku mekanisme proteksi dapat berbeda antarversi DVWA dan
antarfitur. Gunakan source code versi yang dipasang sebagai acuan saat
menjelaskan detail implementasi.

Urutan pembelajaran yang disarankan: mulai dari **Low**, amati
dampaknya, lalu ulangi pada **Medium**, **High**, dan **Impossible**
untuk membandingkan hasil.

## 8. Modul Praktikum

Berikut contoh materi yang dapat digunakan dalam pembelajaran. Lakukan
pengujian hanya pada instance DVWA milik sendiri.

  -----------------------------------------------------------------------
  Modul                   Konsep yang dipelajari  Fokus pembelajaran
  ----------------------- ----------------------- -----------------------
  SQL Injection           Input tidak aman dalam  Parameterisasi query
                          query SQL               dan validasi input

  Blind SQL Injection     Inferensi melalui       Perbedaan respons dan
                          respons aplikasi        risiko kebocoran
                                                  informasi

  Reflected XSS           Input dipantulkan ke    Output encoding dan
                          halaman                 konteks HTML/JavaScript

  Stored XSS              Input tersimpan dan     Sanitasi, encoding, dan
                          ditampilkan kembali     penyimpanan data

  DOM XSS                 Manipulasi DOM pada     Sumber data tidak
                          sisi klien              tepercaya dan sink yang
                                                  berbahaya

  Command Injection       Input memengaruhi       Menghindari pemanggilan
                          perintah sistem         shell dan membatasi hak
                                                  proses

  File Upload             Validasi file yang      Validasi konten,
                          diunggah                ekstensi, MIME, dan
                                                  lokasi penyimpanan

  CSRF                    Permintaan yang dipicu  Token anti-CSRF dan
                          tanpa otorisasi memadai validasi origin

  File Inclusion          Pemilihan file          Allowlist dan
                          berdasarkan input       pembatasan path

  Brute Force             Percobaan autentikasi   Rate limiting, MFA, dan
                          berulang                pemantauan
  -----------------------------------------------------------------------

### Format kegiatan praktikum

Untuk setiap modul, mahasiswa dapat diminta untuk:

1.  Menjelaskan fungsi fitur yang diuji.
2.  Mengidentifikasi input yang dikendalikan pengguna.
3.  Mengamati perilaku aplikasi pada level Low.
4.  Mencatat bukti pengujian yang tidak mengandung data sensitif.
5.  Mengulang pengamatan pada level keamanan lain.
6.  Menjelaskan akar penyebab kerentanan.
7.  Mengusulkan mitigasi dan cara menguji perbaikannya.

Gunakan Burp Suite atau browser Developer Tools untuk mengamati request
dan response HTTP. Jangan melakukan pengujian terhadap sistem di luar
lab tanpa izin tertulis.

## 9. Pengelolaan Container

### Melihat container

``` bash
docker compose ps
```

### Melihat log

``` bash
docker compose logs -f
```

### Menghentikan lab

``` bash
docker compose stop
```

Container berhenti tetapi tetap ada. Data database tetap tersimpan.

### Menjalankan kembali lab

``` bash
docker compose start
```

### Menghentikan dan menghapus container serta network

``` bash
docker compose down
```

Named volume database tidak dihapus oleh perintah tersebut.

### Mengunduh image terbaru dan membuat ulang container

``` bash
docker compose pull
docker compose up -d
```

Perlu diingat bahwa tag `latest` dapat berubah. Untuk kelas yang
membutuhkan hasil konsisten, gunakan versi image yang telah diuji dan
pin versinya atau digest image. Uji pembaruan terlebih dahulu sebelum
digunakan dalam sesi pengajaran.

## 10. Backup dan Pemulihan Database

### Backup

Buat backup database menggunakan:

``` bash
docker exec dvwa-db mariadb-dump -u root -proot_password dvwa > dvwa-backup.sql
```

Simpan file backup di tempat yang aman. File tersebut berisi data
database dan harus diperlakukan sebagai data sensitif.

### Pemulihan

Pastikan container database berjalan, kemudian impor:

``` bash
cat dvwa-backup.sql | docker exec -i dvwa-db mariadb -u root -proot_password dvwa
```

Pemulihan akan mengubah isi database tujuan. Pastikan target database
benar dan buat backup sebelum menimpa data yang masih diperlukan.

## 11. Troubleshooting

### Port sudah digunakan

Gejala: Docker gagal memulai container atau menampilkan pesan
`port is already allocated`.

Periksa:

``` bash
docker ps
```

Ubah port host pada Compose:

``` yaml
ports:
  - "8086:80"
```

Terapkan:

``` bash
docker compose up -d
```

### Container DVWA tidak dapat terhubung ke database

Periksa:

``` bash
docker compose ps
docker compose logs db
docker compose logs dvwa
```

Pastikan:

-   Kedua service berada di `dvwa-network`.
-   `DB_SERVER` bernilai `db`, bukan `localhost`.
-   Nama database dan kredensial aplikasi sama dengan konfigurasi
    MariaDB.
-   MariaDB sudah siap menerima koneksi.

`depends_on` biasa hanya mengatur urutan start, bukan readiness. Jika
database membutuhkan waktu untuk inisialisasi, tunggu beberapa saat dan
periksa log sebelum mencoba lagi.

### Password database berubah tetapi koneksi tetap gagal

Jika volume database sudah pernah dibuat, perubahan `MYSQL_PASSWORD` di
Compose tidak otomatis mengubah password akun yang ada. Gunakan
kredensial yang memang tersimpan di database atau lakukan perubahan akun
secara terencana. Jangan menghapus volume sebagai langkah pertama.

### Login DVWA gagal

Periksa bahwa:

-   Kredensial yang digunakan adalah `admin` / `password`.
-   Database DVWA sudah dibuat melalui halaman Setup.
-   Tabel `users` tersedia.
-   Container aplikasi dan database berjalan.
-   Browser tidak memakai cookie/session lama.
-   Log aplikasi tidak menunjukkan error database atau PHP.

### Halaman kosong atau error PHP

Periksa:

``` bash
docker logs dvwa --tail 100
```

Coba muat ulang halaman dan periksa Developer Tools browser. Catat pesan
error yang terlihat. Jangan menonaktifkan kontrol keamanan host untuk
mengatasi error aplikasi.

### Reset total lab

**PERINGATAN: perintah berikut menghapus seluruh data database dalam
named volume proyek.**

Jika memang ingin memulai dari nol:

``` bash
docker compose down -v
docker compose up -d
```

Setelah container siap, buka DVWA dan lakukan **Create / Reset
Database** lagi.

## 12. Penggunaan untuk Kelas

### Akses dari komputer instruktur

Secara default, mapping `8085:80` dapat membuat aplikasi tersedia pada
interface jaringan host, bergantung pada konfigurasi Docker. Untuk
penggunaan lokal satu perangkat, lebih aman membatasi binding ke
loopback:

``` yaml
ports:
  - "127.0.0.1:8085:80"
```

Dengan konfigurasi tersebut, DVWA hanya dapat diakses melalui host itu
sendiri.

### Akses dari komputer mahasiswa melalui LAN

Jika mahasiswa perlu mengakses satu instance instruktur:

-   Gunakan jaringan kelas yang terisolasi.
-   Pastikan firewall hanya mengizinkan subnet kelas menuju port DVWA.
-   Jangan lakukan port forwarding dari router ke internet.
-   Gunakan data dummy.
-   Matikan lab setelah sesi selesai.

Untuk membatasi akses, jangan mengandalkan konfigurasi Docker saja.
Periksa aturan firewall host dan kebijakan jaringan karena perilaku
publikasi port Docker dapat berbeda antarplatform.

### Satu instance per kelompok

Untuk kelas yang lebih besar, pertimbangkan satu stack per kelompok
dengan port host yang berbeda, misalnya `8085`, `8086`, dan `8087`,
serta project name dan volume yang terpisah. Jangan menggunakan nama
container atau volume yang sama untuk beberapa stack jika ingin isolasi
data.

## 13. Batasan dan Catatan

-   DVWA adalah aplikasi latihan, bukan aplikasi produksi.
-   Level keamanan DVWA merupakan simulasi pembelajaran, bukan standar
    penilaian keamanan sistem nyata.
-   Keberhasilan suatu demonstrasi tidak otomatis membuktikan dampak
    pada sistem lain.
-   Gunakan image yang telah diverifikasi dan versi yang konsisten untuk
    materi kelas.
-   Jangan memasukkan data pribadi, kredensial nyata, atau file penting
    ke dalam lab.
-   Jangan mengekspos DVWA ke jaringan publik.
-   Simpan backup sebelum melakukan reset atau perubahan database.

------------------------------------------------------------------------

## Referensi

-   DVWA --- proyek resmi: https://github.com/digininja/DVWA
-   Docker Compose documentation: https://docs.docker.com/compose/
-   MariaDB documentation: https://mariadb.com/kb/en/documentation/

**Selesai.** Setelah mengikuti panduan ini, lab DVWA seharusnya dapat
dijalankan melalui Docker Compose dan digunakan sebagai lingkungan
praktikum keamanan aplikasi web yang terkontrol.

