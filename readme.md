# Odoo 20 Community Edition — Docker Development Environment

Lingkungan pengembangan **Odoo 20 CE** berbasis Docker Compose yang terisolasi penuh per project. Seluruh container, network, dan volume memakai prefix nama project sehingga tidak bentrok dengan project Odoo lain di mesin yang sama.

## Daftar Isi

- [Deskripsi](#deskripsi)
- [Stack](#stack)
- [Prasyarat](#prasyarat)
- [Struktur Folder](#struktur-folder)
- [Instalasi](#instalasi)
- [Cara Menggunakan](#cara-menggunakan)
- [Perintah Harian](#perintah-harian)
- [Backup dan Restore](#backup-dan-restore)
- [Referensi Konfigurasi](#referensi-konfigurasi)
- [Troubleshooting](#troubleshooting)
- [Catatan Produksi](#catatan-produksi)

---

## Deskripsi

Repositori ini menyediakan environment development Odoo 20 Community Edition yang siap pakai:

- Image resmi `odoo:20.0` (edisi Community).
- Database PostgreSQL 16 (memenuhi syarat minimum Odoo 20).
- Mailpit untuk menangkap seluruh email keluar saat development (tidak ada email yang benar-benar terkirim).
- Tiga folder addons terpisah: modul buatan sendiri, dan modul pihak ketiga.
- **Isolasi project**: nama container, network, dan volume otomatis diberi prefix dari `COMPOSE_PROJECT_NAME`, dan port host dapat diatur lewat `.env`.

> Environment ini ditujukan untuk **development**. Lihat bagian [Catatan Produksi](#catatan-produksi) sebelum dipakai di server.

## Stack

| Komponen | Image | Fungsi |
|---|---|---|
| `odoo` | `odoo:20.0` | Aplikasi Odoo 20 CE |
| `db` | `postgres:16` | Database PostgreSQL |
| `mail` | `axllent/mailpit:latest` | Penangkap email development (SMTP + Web UI) |

Data disimpan di named volume Docker (`odoo_data` untuk filestore, `pg_data` untuk database), bukan di folder project.

## Prasyarat

- Docker Engine dan Docker Compose v2 (`docker compose`, bukan `docker-compose`).
- Minimal 4 GB RAM yang tersedia untuk Docker (disarankan 8 GB).
- Port host yang dipakai pada `.env` dalam keadaan bebas (default: `18069` dan `18025`).

Cek versi:

```bash
docker --version
docker compose version
```

## Struktur Folder

```
odoo20-<nama-project>/
├── .env                  # rahasia & port (JANGAN di-commit)
├── .env.example          # template untuk tim
├── .gitignore
├── docker-compose.yml
├── README.md
├── config/
│   └── odoo.conf         # konfigurasi Odoo
├── addons/
│   ├── custom/           # modul buatan sendiri
│   ├── third_party/      # modul OCA / pihak ketiga
│   └── bwa/              # modul khusus BWA
└── backups/              # hasil dump database (di-ignore git)
```

## Instalasi

### 1. Siapkan folder project

```bash
mkdir -p odoo20-<nama-project>/{config,backups,addons/{custom,third_party,bwa}}
cd odoo20-<nama-project>
```

Letakkan `docker-compose.yml`, `.env.example`, `.gitignore`, dan `config/odoo.conf` ke dalam folder tersebut.

### 2. Buat file `.env`

```bash
cp .env.example .env
```

Edit `.env` dan **ubah nilai berikut agar unik untuk project ini**:

```env
# WAJIB unik per project. Jadi prefix container, network, dan volume.
COMPOSE_PROJECT_NAME=acme_odoo20

# Port di host (hanya 127.0.0.1). Ubah bila bentrok.
ODOO_PORT=18069
MAIL_UI_PORT=18025

# Database khusus project ini
POSTGRES_USER=odoo_acme
POSTGRES_PASSWORD=ganti_dengan_password_acak
```

### 3. Atur `config/odoo.conf`

Pastikan minimal `admin_passwd` sudah diganti (master password untuk database manager). Nilai ini tidak dibaca dari `.env`, jadi isi langsung di file konfigurasi.

```ini
[options]
admin_passwd = ganti_master_password_acak

addons_path = /mnt/extra-addons/custom,/mnt/extra-addons/third_party,/mnt/extra-addons/bwa
data_dir = /var/lib/odoo

db_host = db
db_port = 5432

workers = 0
list_db = True
log_level = info
; dev_mode = reload,qweb,xml

smtp_server = mail
smtp_port = 1025
smtp_ssl = False
email_from = noreply@acme.local
```

> Core addons Odoo otomatis ditambahkan, jadi `addons_path` hanya perlu berisi folder addons tambahan.

### 4. Validasi dan jalankan

```bash
docker compose config      # cek hasil substitusi variabel .env
docker compose pull        # unduh image
docker compose up -d       # jalankan di background
docker compose ps          # pastikan semua service berstatus running/healthy
```

### 5. Isi `.gitignore`

```gitignore
.env
backups/
__pycache__/
*.pyc
```

## Cara Menggunakan

### Akses layanan

| Layanan | URL |
|---|---|
| Odoo | http://localhost:18069 |
| Mailpit (inbox email dev) | http://localhost:18025 |

Ganti port sesuai `ODOO_PORT` dan `MAIL_UI_PORT` di `.env` Anda.

### Membuat database pertama

1. Buka `http://localhost:18069`. Halaman database manager akan tampil karena belum ada database.
2. Isi **Master Password** dengan nilai `admin_passwd` dari `odoo.conf`.
3. Isi nama database yang unik (contoh: `acme_dev`), email, dan password admin.
4. Centang **Demo data** hanya jika diperlukan untuk eksperimen.
5. Klik **Create database** dan tunggu proses inisialisasi selesai.

### Menambah modul custom

1. Letakkan folder modul di salah satu folder addons:
   - `addons/custom/` untuk modul buatan sendiri
   - `addons/third_party/` untuk modul OCA / pihak ketiga
   - `addons/bwa/` untuk modul khusus BWA
2. Restart Odoo agar modul Python terbaca:

   ```bash
   docker compose restart odoo
   ```
3. Di Odoo, aktifkan **Developer Mode**, buka **Apps**, lalu klik **Update Apps List**.
4. Cari modul dan klik **Install**.

### Mengubah kode modul yang sudah terpasang

- Perubahan file Python: `docker compose restart odoo`, lalu upgrade modul.
- Perubahan view/XML/data: upgrade modul lewat UI (Apps, lalu Upgrade) atau lewat CLI:

  ```bash
  docker compose run --rm odoo odoo -d <nama_db> -u <nama_modul> --stop-after-init
  ```
- Opsional: aktifkan `dev_mode = reload,qweb,xml` di `odoo.conf` untuk auto-reload saat development.

### Mengetes pengiriman email

Konfigurasi SMTP di `odoo.conf` sudah mengarah ke Mailpit. Email apa pun yang dikirim Odoo (reset password, notifikasi, dll.) tidak keluar ke internet, tetapi bisa dilihat di Web UI Mailpit.

## Perintah Harian

| Tujuan | Perintah |
|---|---|
| Cek konfigurasi | `docker compose config` |
| Start | `docker compose up -d` |
| Status service | `docker compose ps` |
| Log Odoo (live) | `docker compose logs -f odoo` |
| Log semua service | `docker compose logs -f` |
| Restart Odoo (setelah ubah kode Python) | `docker compose restart odoo` |
| Update modul via CLI | `docker compose run --rm odoo odoo -d <db> -u <modul> --stop-after-init` |
| Install modul via CLI | `docker compose run --rm odoo odoo -d <db> -i <modul> --stop-after-init` |
| Shell Odoo (Python) | `docker compose run --rm odoo odoo shell -d <db>` |
| Shell DB (psql) | `docker compose exec db sh -c 'psql -U "$POSTGRES_USER" postgres'` |
| Masuk ke container Odoo | `docker compose exec odoo bash` |
| Update image ke rilis terbaru 20.0 | `docker compose pull && docker compose up -d` |
| Stop (data tetap aman) | `docker compose down` |
| **Hapus total project ini (data ikut hilang)** | `docker compose down -v` |

> `docker compose down -v` hanya menghapus volume milik project ini (ber-prefix `COMPOSE_PROJECT_NAME`), tidak menyentuh project lain.

## Backup dan Restore

Data Odoo terbagi dua: **database** (volume `pg_data`) dan **filestore** (volume `odoo_data`). Backup database saja tidak cukup untuk restore penuh.

### Opsi 1 — Backup penuh lewat Database Manager (disarankan)

1. Buka `http://localhost:18069/web/database/manager`.
2. Pilih database, klik **Backup**, format **zip (includes filestore)**.
3. Simpan hasilnya ke folder `backups/`.

Restore dengan menu **Restore Database** pada halaman yang sama.

### Opsi 2 — Dump database via CLI (hanya DB, tanpa filestore)

Backup:

```bash
docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" -Fc <nama_db>' > backups/<nama_db>_$(date +%F).dump
```

Restore (database tujuan dibuat kosong terlebih dahulu):

```bash
docker compose exec db sh -c 'createdb -U "$POSTGRES_USER" <nama_db_baru>'
docker compose exec -T db sh -c 'pg_restore -U "$POSTGRES_USER" -d <nama_db_baru> --no-owner' < backups/<file>.dump
```

### Backup filestore (opsional, melengkapi Opsi 2)

```bash
docker run --rm \
  -v ${COMPOSE_PROJECT_NAME}_odoo_data:/data:ro \
  -v "$(pwd)/backups":/backup \
  alpine tar czf /backup/filestore_$(date +%F).tar.gz -C /data .
```

> Perintah di atas membaca `COMPOSE_PROJECT_NAME` dari shell. Jalankan `set -a; source .env; set +a` terlebih dahulu, atau ganti manual dengan nama project Anda.

## Referensi Konfigurasi

### Variabel `.env`

| Variabel | Fungsi | Contoh |
|---|---|---|
| `COMPOSE_PROJECT_NAME` | Prefix unik untuk container, network, dan volume | `acme_odoo20` |
| `ODOO_PORT` | Port host untuk Odoo (bind ke 127.0.0.1) | `18069` |
| `MAIL_UI_PORT` | Port host untuk Web UI Mailpit | `18025` |
| `POSTGRES_USER` | User database | `odoo_acme` |
| `POSTGRES_PASSWORD` | Password database | (acak, kuat) |

### Opsi penting `odoo.conf`

| Opsi | Keterangan |
|---|---|
| `admin_passwd` | Master password database manager |
| `addons_path` | Daftar folder addons tambahan |
| `workers` | `0` untuk development (single process) |
| `list_db` | `True` untuk dev, `False` untuk produksi |
| `dev_mode` | Contoh: `reload,qweb,xml` untuk auto-reload saat development |
| `smtp_server`, `smtp_port` | Diarahkan ke service `mail` (Mailpit) |

### Resource yang dibuat Docker

Dengan `COMPOSE_PROJECT_NAME=acme_odoo20`:

| Jenis | Nama |
|---|---|
| Network | `acme_odoo20_net` |
| Volume | `acme_odoo20_odoo_data`, `acme_odoo20_pg_data` |
| Container | `acme_odoo20-odoo-1`, `acme_odoo20-db-1`, `acme_odoo20-mail-1` |

## Troubleshooting

**Port sudah dipakai / `bind: address already in use`**
Ubah `ODOO_PORT` atau `MAIL_UI_PORT` di `.env`, lalu `docker compose up -d`.

**Variabel `.env` tidak terbaca / muncul pesan `isi COMPOSE_PROJECT_NAME di .env`**
Pastikan file `.env` berada satu folder dengan `docker-compose.yml` dan perintah dijalankan dari folder tersebut.

**Odoo gagal start karena tidak bisa konek database**
Cek `docker compose ps` dan `docker compose logs db`. Service `odoo` baru berjalan setelah `db` berstatus *healthy*. Pastikan `POSTGRES_USER` dan `POSTGRES_PASSWORD` tidak diubah setelah volume `pg_data` pertama kali dibuat (kredensial tersimpan di volume). Jika perlu mengulang dari nol: `docker compose down -v`.

**Modul tidak muncul di Apps**
Pastikan folder modul berisi `__manifest__.py`, restart Odoo, aktifkan Developer Mode, lalu **Update Apps List**.

**Perubahan `odoo.conf` tidak berlaku**
Jalankan `docker compose restart odoo`. File ini di-mount read-only dan hanya dibaca saat Odoo start.

**Email tidak muncul di Mailpit**
Pastikan `smtp_server = mail` dan `smtp_port = 1025` di `odoo.conf`, lalu restart Odoo. Port `1025` adalah port internal container, bukan port host.

## Catatan Produksi

Konfigurasi ini dirancang untuk development. Sebelum dipakai di server produksi:

- Ganti `admin_passwd` dan `POSTGRES_PASSWORD` dengan nilai kuat dan unik.
- Set `list_db = False`.
- Set `workers` ke 2 atau lebih sesuai CPU/RAM. Dengan `workers > 0`, koneksi websocket berjalan di port 8072 sehingga perlu reverse proxy (Nginx/Traefik) yang meneruskan `/websocket` ke port tersebut.
- Set `proxy_mode = True` bila berada di belakang reverse proxy, dan pasang HTTPS.
- Hapus service `mail` (Mailpit) dan konfigurasikan SMTP sungguhan.
- Pin tag image ke versi spesifik dan jadwalkan backup otomatis (database + filestore).
- Jangan expose port PostgreSQL ke jaringan publik.

> **Catatan upgrade:** database Odoo 20 tidak kompatibel dengan versi major lain. Perpindahan antar-versi major memerlukan proses migrasi, bukan sekadar mengganti tag image.