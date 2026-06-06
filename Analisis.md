# Analisis Perbaikan

## Permasalahan 1

### Gejala
`docker-compose.yml` tidak dapat di-parse (error sintaks) saat menjalankan `docker compose up`.

### Penyebab
Key `services` ditulis tanpa tanda titik dua (`services` bukan `services:`), sehingga file Compose invalid.

### Solusi
Tambahkan tanda titik dua setelah `services`.

---

## Permasalahan 2

### Gejala
Beberapa service tidak saling terhubung: Nginx tidak dapat mencapai `web1`/`web3`, dan volume database tidak terpasang.

### Penyebab
- `web1` menggunakan `DB_HOST: mysql` padahal service database bernama `db`.
- `web2` menggunakan `DB_PASS: wrongpassword` (tidak sama dengan `MYSQL_PASSWORD`).
- `web3` build context salah (`./web33`) dan hanya bergabung ke network `backend` (tidak di `frontend`).
- Volume yang didefinisikan di service adalah `db-data` tetapi di bagian `volumes:` root dideklarasikan sebagai `database-data` (nama tidak konsisten).

### Solusi
- Gunakan `DB_HOST: db` di semua web service atau gunakan nama service `db` konsisten.
- Samakan password `DB_PASS` pada web service menjadi `student123` sesuai `MYSQL_PASSWORD`.
- Perbaiki build context `./web33` → `./web3` dan tambahkan `frontend` network ke `web3` agar dapat di-loadbalanced oleh Nginx.
- Samakan nama volume: gunakan `db-data:` di bagian `volumes:` root, atau ubah service untuk menggunakan `database-data:`; pilih salah satu nama dan konsisten.

---

## Permasalahan 3

### Gejala
Nginx tidak melakukan proxy ke web server; container Nginx gagal pada reload konfigurasi.

### Penyebab
File `nginx/nginx.conf` berisi fence Markdown (```nginx ... ```) sehingga Nginx tidak dapat membaca konfigurasi. Selain itu upstream berisi nama host dan port yang salah: `web11:80` (seharusnya `web1:80`) dan `web3:8080` (port seharusnya 80).

### Solusi
- Hapus fence Markdown dari `nginx.conf` sehingga berisi konfigurasi Nginx valid.
- Perbaiki upstream menjadi:

```
upstream backend {
    server web1:80;
    server web2:80;
    server web3:80;
}
```

---

## Permasalahan 4

### Gejala
Database init script tidak membuat tabel pada database `responsi` dan berisi teks yang bukan SQL saat container database diinisialisasi.

### Penyebab
`db/init.sql` mengandung fence Markdown (```sql ... ```) dan tidak menyertakan `USE responsi;` sehingga SQL mungkin tidak dijalankan pada database yang diharapkan.

### Solusi
- Hapus fence Markdown dari `init.sql` dan tambahkan `USE responsi;` atau buat statement `CREATE DATABASE responsi;` jika diperlukan. Contoh:

```
USE responsi;
CREATE TABLE IF NOT EXISTS students (...);
INSERT INTO students ...;
```

---

## Permasalahan 5

### Gejala
Build image untuk `web1` atau `web3` gagal karena base image tidak ditemukan, atau service berjalan di port yang tidak diharapkan.

### Penyebab
- `web1/Dockerfile` menggunakan image dasar `php:8.2-apach` (typo) seharusnya `php:8.2-apache`.
- `web3/Dockerfile` menggunakan `php:8.2-apche` (typo) seharusnya `php:8.2-apache`.
- Semua web Dockerfile mengekspos `8080` padahal Apache bawaan container biasanya berjalan di port `80`.

### Solusi
- Perbaiki base image menjadi `php:8.2-apache` pada semua Dockerfile web.
- Ubah atau hapus `EXPOSE 8080`; jika ingin eksplisit gunakan `EXPOSE 80`.
---

## Permasalahan 6

### Gejala
Container `mysql-db` gagal start dengan error:
```
Attaching to mysql-db
mysql-db | Fatal glibc error: CPU does not support x86-64-v2
```

### Penyebab
Image MySQL `8.0` atau `8.0.28` dibangun dengan optimisasi glibc yang membutuhkan instruction set x86-64-v2, yang tidak didukung oleh host/VM yang digunakan.

### Solusi
Ganti base image pada `db/Dockerfile` ke versi yang lebih kompatibel:
```dockerfile
FROM mysql:8.0.28
```


Rebuild container:
```bash
docker compose down
docker compose build --no-cache db
docker compose up -d
```

---

## Permasalahan 7

### Gejala
Container `nginx-lb` start tetapi Nginx daemon gagal dengan error:
```
[emerg] unknown directive "nginx" in /etc/nginx/nginx.conf:2
nginx: [emerg] unknown directive "nginx" in /etc/nginx/nginx.conf:2
```

### Penyebab
File `nginx/nginx.conf` memiliki baris `nginx` di baris pertama yang tidak valid (artifact atau kesalahan saat copy).

### Solusi
Baris `nginx` di awal file `nginx/nginx.conf` sudah dihapus. Rebuild image Nginx dengan cache clear:
```bash
docker compose down
docker compose build --no-cache nginx
docker compose up -d
```

Verifikasi:
```bash
docker compose logs nginx
docker compose ps
```
