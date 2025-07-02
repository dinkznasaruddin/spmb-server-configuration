# Web Server Optimization - SPMB Kota Makassar

Konfigurasi ini digunakan untuk mengoptimalkan performa web server berbasis **Nginx + PHP-FPM** dalam menangani trafik tinggi saat pendaftaran SPMB (Sistem Penerimaan Mahasiswa Baru) Kota Makassar.

---

## 📌 Tujuan

* Mencegah overload & downtime saat lonjakan pengunjung
* Membatasi jumlah koneksi dan permintaan per IP
* Menjaga kestabilan PHP-FPM
* Memberikan edukasi kepada pengguna saat server sibuk

---

## 🧰 Teknologi yang Digunakan

* Ubuntu Server
* RAM 175
* Core 32
* VM Sangfor 
* Nginx 1.18
* PHP 7.4 + PHP-FPM
* Mode Single Web Server (belum Load Balancer)

  

---

## 🔧 Konfigurasi Nginx

### 1. Tambahkan di blok `http` (biasanya di `/etc/nginx/nginx.conf`)

```nginx
limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;
limit_req_zone $binary_remote_addr zone=req_limit_per_ip:10m rate=10r/s;
```

### 2. Konfigurasi di dalam blok `server`

```nginx
server {
    server_name spmb.makassarkota.go.id;
    client_max_body_size 100M;

    root /var/www/html/ppdb/;
    index index.html index.php index.htm;

    limit_conn conn_limit_per_ip 5;
    limit_req zone=req_limit_per_ip burst=10 nodelay;

    location / {
        alias /var/www/html/ppdb/;
        index index.htm index.php;
        limit_req zone=req_limit_per_ip burst=10 nodelay;
        limit_conn conn_limit_per_ip 5;
        try_files $uri $uri/ /index.php;
    }

    # Endpoint login - dibatasi lebih ketat
    location = /login {
        limit_req zone=req_limit_per_ip burst=5 nodelay;
        limit_conn conn_limit_per_ip 2;
        try_files $uri $uri/ /index.php;
    }

    location ~ \.php$ {
        limit_req zone=req_limit_per_ip burst=10 nodelay;
        limit_conn conn_limit_per_ip 5;

        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 120s;
        fastcgi_read_timeout 120s;
        fastcgi_buffering on;
    }

    error_page 503 /full.html;
    location = /full.html {
        internal;
        root /var/www/html;
    }
}
```

---

## 🧠 Edukasi Pengguna (`/full.html`)

```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8" />
  <title>Sedang Padat</title>
</head>
<body>
  <h2>Mohon Maaf</h2>
  <p>Saat ini server sedang sangat padat. Silakan coba beberapa saat lagi.</p>
  <p>Tips: Gunakan jaringan seluler untuk akses yang lebih lancar.</p>
</body>
</html>
```

---

## ⚙️ Tuning PHP-FPM

Edit `/etc/php/7.4/fpm/pool.d/www.conf`

```ini
pm = dynamic
pm.max_children = 1200
pm.start_servers = 60
pm.min_spare_servers = 30
pm.max_spare_servers = 120
pm.max_requests = 500
pm.process_idle_timeout = 10s
```

---

## ✅ Hasil

* Website tetap stabil dengan lebih dari **20.000 pengguna aktif harian**
* Trafik tinggi pada endpoint `/login` terkendali
* PHP-FPM tidak lagi overload
* Server tetap responsif (200 OK) pada akses normal

---

## 📈 Rekomendasi Selanjutnya

* Implementasi **Load Balancer (Nginx/HAProxy)** jika menggunakan 2+ server web
* Monitoring sistem dengan **Grafana / Netdata**
* Evaluasi konfigurasi pasca-event untuk perbaikan

---

**Disusun oleh:**
Tim IT Support SPMB Kota Makassar
2025

