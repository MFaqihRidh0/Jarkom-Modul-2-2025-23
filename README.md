# LAPRES Praktikum Komunikasi Data dan Jaringan Komputer Modul 2 - K-23

## Anggota
1. M. Faqih Ridho - 5027241123
2. Kaisar Hanif Pratama - 5027241029


## Pengerjaan

### Nomor 1
<img width="748" height="364" alt="image" src="https://github.com/user-attachments/assets/a4908aa2-db0b-47db-8a5e-c69befb35d0a" />

Nomer 1 membuat topologi sesuai arahan soal

### Nomor 2

<img width="1127" height="311" alt="image" src="https://github.com/user-attachments/assets/eee9a9b1-1b3b-4c48-aa95-ec0eb6d67795" />

Nomer 2 menyuruh untuk node dengan nama eonwe bisa mengakses internet

### Nomor 3

<img width="951" height="559" alt="image" src="https://github.com/user-attachments/assets/bcb2c8c6-865e-4996-b38e-083d6628613c" />

Nomer 3 menyuruh untuk node dari sisi kiri dan kanan bisa saling berkomunikasi

### Nomor 4

1) Bagian yang sama (jalankan di NS1 & NS2)

```
apt-get update
apt-get install -y bind9 bind9utils dnsutils

# /etc/bind/named.conf.options (sama di keduanya)
cat >/etc/bind/named.conf.options <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
};
EOF

chown -R bind:bind /etc/bind /var/cache/bind
```

2) TIRION (NS1/master – 10.75.3.3)

```
# /etc/bind/named.conf.local (zona master)
cat >/etc/bind/named.conf.local <<'EOF'
zone "k23.com" {
    type master;
    file "/etc/bind/zones/db.k23.com";
    allow-transfer { 10.75.3.4; };   // Valmar
    also-notify   { 10.75.3.4; };
    notify yes;
};
EOF

# file zona utama
mkdir -p /etc/bind/zones
cat >/etc/bind/zones/db.k23.com <<'EOF'
$TTL 300
@   IN SOA ns1.k23.com. admin.k23.com. (
        2025101301 ; Serial (naikkan tiap edit)
        3600       ; Refresh
        900        ; Retry
        1209600    ; Expire
        300 )      ; Minimum
    IN NS  ns1.k23.com.
    IN NS  ns2.k23.com.

; alamat NS
ns1     IN A 10.75.3.3
ns2     IN A 10.75.3.4

; apex (front door)
@       IN A 10.75.3.2

; host layanan
sirion   IN A 10.75.3.2
lindon   IN A 10.75.3.5
vingilot IN A 10.75.3.6

; alias (CNAME)
www     IN CNAME sirion.k23.com.
static  IN CNAME lindon.k23.com.
app     IN CNAME vingilot.k23.com.
EOF

# Validasi & jalankan named (tanpa systemd)
named-checkconf
named-checkzone k23.com /etc/bind/zones/db.k23.com
pkill named 2>/dev/null || true
/usr/sbin/named -u bind -c /etc/bind/named.conf
ss -tulpn | grep ':53'   # cek port 53
```

3) VALMAR (NS2/slave – 10.75.3.4)

Di slave jangan bikin file zona manual. Biarkan dia menarik dari NS1.

```
# /etc/bind/named.conf.local (zona slave)
cat >/etc/bind/named.conf.local <<'EOF'
zone "k23.com" {
    type slave;
    masters { 10.75.3.3; };                  // Tirion
    allow-notify { 10.75.3.3; };             // terima NOTIFY dari master
    file "/var/cache/bind/db.k23.com";       // hasil transfer disimpan di sini
};
EOF

# Jalankan named; saat start ia akan AXFR dari NS1
named-checkconf
pkill named 2>/dev/null || true
/usr/sbin/named -u bind -c /etc/bind/named.conf
ss -tulpn | grep ':53'
```
### Verifikasi

Coba di earendil
test tirion

```
dig @10.75.3.3 k23.com
```

Test valmar
```
dig @10.75.3.4 k23.com
```


### Nomor 5

Untuk membuat domain untuk setiap node yang pertama harus configure dari master yaitu Tirion. Berikut konfigurasinya 

```
# 1) Pastikan direktori zona ada
mkdir -p /etc/bind/zones

# 2) named.conf.local – master k23.com dan izinkan transfer ke ns2
cat >/etc/bind/named.conf.local <<'EOF'
zone "k23.com" {
    type master;
    file "/etc/bind/zones/db.k23.com";
    allow-transfer { 10.75.3.4; };   // Valmar
    also-notify   { 10.75.3.4; };
    notify yes;
};
EOF

# 3) named.conf.options (forwarder & listen any)
cat >/etc/bind/named.conf.options <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation no;
    listen-on { any; };
    listen-on-v6 { any; };
};
EOF

# 4) File zona k23.com (lengkap)
cat >/etc/bind/zones/db.k23.com <<'EOF'
$TTL 300
@   IN SOA ns1.k23.com. admin.k23.com. (
        2025101901 ; Serial  (naikkan setiap edit)
        3600       ; Refresh
        900        ; Retry
        1209600    ; Expire
        300 )      ; Minimum

; authoritative NS
    IN  NS  ns1.k23.com.
    IN  NS  ns2.k23.com.

; alamat NS
ns1 IN  A   10.75.3.3
ns2 IN  A   10.75.3.4

; apex (front door)
@   IN  A   10.75.3.2
www IN  CNAME @

; host layanan
sirion   IN  A   10.75.3.2
lindon   IN  A   10.75.3.5
vingilot IN  A   10.75.3.6
static   IN  CNAME lindon.k23.com.
app      IN  CNAME vingilot.k23.com.

; barat (10.75.1.0/24)
earendil IN  A   10.75.1.2
elwing   IN  A   10.75.1.3

; timur (10.75.2.0/24)
cirdan   IN  A   10.75.2.2
elrond   IN  A   10.75.2.3
maglor   IN  A   10.75.2.4
EOF

# 5) Reload named (tanpa systemd)
pkill named 2>/dev/null || true
/usr/sbin/named -u bind -c /etc/bind/named.conf

```

Setelah Tirion , konfigurasi untuk valmar

```
# 1) named.conf.local – tarik zona dari ns1
cat >/etc/bind/named.conf.local <<'EOF'
zone "k23.com" {
    type slave;
    masters { 10.75.3.3; };           // Tirion
    file "/var/cache/bind/db.k23.com";
};
EOF

# 2) named.conf.options (resolver sama dengan ns1)
cat >/etc/bind/named.conf.options <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation no;
    listen-on { any; };
    listen-on-v6 { any; };
};
EOF

# 3) Start/reload
pkill named 2>/dev/null || true
/usr/sbin/named -u bind -c /etc/bind/named.conf

# 4) (opsional) pancing refresh/retransfer
rndc refresh k23.com 2>/dev/null || true
rndc retransfer k23.com 2>/dev/null || true

```

Setelah itu, kita membuat domainnya. untuk membuat domain setiap node perlu mengkonfigurasikan ke masing-masing node dengan tab terminal yang berbeda-beda.Berikut adalah konfigurasinya , dalam pengerjaan menyesuaikan nama node.

```
cat >/etc/resolv.conf <<'EOF'
search k23.com
nameserver 10.75.3.3
nameserver 10.75.3.4
nameserver 192.168.122.1
EOF

```
Berikut salah satu bukti 

<img width="569" height="301" alt="image" src="https://github.com/user-attachments/assets/40cf5faa-882d-478a-8f01-a0b2aa6e439e" />


### Nomor 6

1) NS1 (Tirion) — naikkan serial & kirim NOTIFY

Jalankan di Tirion (10.75.3.3)

Opsi otomatis (tanpa buka editor):

```
DOMAIN=k23.com
ZFILE=/etc/bind/zones/db.$DOMAIN
S=$(dig @127.0.0.1 $DOMAIN SOA +short | awk '{print $3}')
N=$((S+1))
[ -n "$S" ] && [ -n "$N" ] && sed -i "0,/$S/s//$N/" "$ZFILE"
named-checkzone $DOMAIN "$ZFILE"
(rndc reload $DOMAIN || kill -HUP $(pidof named) || { pkill named; /usr/sbin/named -u bind -c /etc/bind/named.conf; })
(rndc notify $DOMAIN 2>/dev/null || true)
dig @127.0.0.1 $DOMAIN SOA +short
```

Kalau mau manual: buka nano /etc/bind/zones/db.k23.com, naikkan angka Serial (+1), simpan, lalu:

```
named-checkzone k23.com /etc/bind/zones/db.k23.com
(rndc reload k23.com || kill -HUP $(pidof named) || { pkill named; /usr/sbin/named -u bind -c /etc/bind/named.conf; })
(rndc notify k23.com 2>/dev/null || true)
dig @10.75.3.3 k23.com SOA +short
```
<img width="907" height="242" alt="image" src="https://github.com/user-attachments/assets/f291e0a2-76cd-45c4-a451-725b3610dab7" />


2) NS2 (Valmar) — tarik salinan terbaru

Jalankan di Valmar (10.75.3.4)

```
DOMAIN=k23.com
(rndc refresh $DOMAIN 2>/dev/null || rndc retransfer $DOMAIN 2>/dev/null || true)
(rndc reload $DOMAIN 2>/dev/null || kill -HUP $(pidof named) || { pkill named; /usr/sbin/named -u bind -c /etc/bind/named.conf; })
dig @127.0.0.1 $DOMAIN SOA +short
ls -l /var/cache/bind/db.$DOMAIN 2>/dev/null || true
```
<img width="947" height="155" alt="soal nomer 6 di valmar naikkan serial (4)" src="https://github.com/user-attachments/assets/5753af58-de69-4199-8878-72f180d31ed3" />


3) Verifikasi (boleh dari node mana saja)
   
```
echo "NS1 serial:"; dig @10.75.3.3 k23.com SOA +short
echo "NS2 serial:"; dig @10.75.3.4 k23.com SOA +short
```


Hasil kedua baris (kolom ke-3) harus sama.
Contoh yang benar:
```
dig @10.75.3.3 k23.com SOA
```
Berikut output yang ditemukan 

<img width="898" height="363" alt="image" src="https://github.com/user-attachments/assets/47049f55-7e5f-48ad-9c9d-5f160b6e530d" />


### Nomor 7

**Tujuan**

Menambahkan:

A record:

sirion.k23.com → 10.75.3.2
lindon.k23.com → 10.75.3.5
vingilot.k23.com → 10.75.3.6

CNAME record:

www.k23.com → sirion.k23.com
static.k23.com → lindon.k23.com
app.k23.com → vingilot.k23.com

**Pengerjaan (NS1 / Tirion – master)**

1. Tambah record ke file zona k23.com (contoh path kami: /etc/bind/zones/db.k23.com):

```
$TTL 300
@   IN SOA ns1.k23.com. admin.k23.com. (
        2025101305 3600 900 1209600 300 )
    IN NS  ns1.k23.com.
    IN NS  ns2.k23.com.

ns1     IN A 10.75.3.3
ns2     IN A 10.75.3.4
@       IN A 10.75.3.2     ; apex mengarah Sirion (opsional)

; host
sirion  IN A 10.75.3.2
lindon  IN A 10.75.3.5
vingilot IN A 10.75.3.6

; alias
www     IN CNAME sirion.k23.com.
static  IN CNAME lindon.k23.com.
app     IN CNAME vingilot.k23.com.
```
**Note** Jangan lupa naikkan serial setiap edit (format YYYYMMDDnn).

<img width="448" height="454" alt="soal 7 konfigurasi tirion bagian 1(5)" src="https://github.com/user-attachments/assets/73300054-68a9-46ce-b8b4-b47356e8d216" />


2. Reload & notify ke slave:

   ```
   named-checkzone k23.com /etc/bind/zones/db.k23.com
rndc reload k23.com
rndc notify k23.com
```

3. Pastikan NS2 menerima perubahan:

```
dig @127.0.0.1 k23.com SOA +short
cat /var/cache/bind/db.k23.com
```

**Verifikasi**

Dari Klient
```dig www.k23.com CNAME +short
dig static.k23.com CNAME +short
dig app.k23.com CNAME +short
dig www.k23.com A +short     # harus 10.75.3.2
dig static.k23.com A +short  # harus 10.75.3.5
dig app.k23.com A +short     # harus 10.75.3.6
```

Dari NS2 (authoritative):

```
dig @10.75.3.4 www.k23.com A +noall +answer +authority +comments
```

<img width="479" height="241" alt="soal 7 hasil (5)" src="https://github.com/user-attachments/assets/70a9b2ff-50bd-4bd4-a3d6-bb6d9573e3e0" />


### Nomor 8

**Tujuan**

Mendeklarasikan reverse zone untuk segmen DMZ 10.75.3.0/24 di NS1 dan menariknya sebagai slave di NS2. Isi PTR:

10.75.3.2 → sirion.k23.com.

10.75.3.5 → lindon.k23.com.

10.75.3.6 → vingilot.k23.com.

**Pengerjaan**

NS1 / Tirion (master)

1. named.conf.local (cuplikan):
   ```
   zone "3.75.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.75.3.rev";
    allow-transfer { 10.75.3.4; };
    notify yes;
};
```

2. File reverse:

```
$TTL 300
@   IN SOA ns1.k23.com. admin.k23.com. (
        2025101301 3600 900 1209600 300 )
    IN NS  ns1.k23.com.
    IN NS  ns2.k23.com.

2   IN PTR sirion.k23.com.
5   IN PTR lindon.k23.com.
6   IN PTR vingilot.k23.com.
```

3. Validasi & reload:

```
named-checkzone 3.75.10.in-addr.arpa /etc/bind/zones/db.10.75.3.rev
rndc reload 3.75.10.in-addr.arpa
rndc notify 3.75.10.in-addr.arpa
```
<img width="362" height="305" alt="Soal 8 konfigurasi tirion (6)" src="https://github.com/user-attachments/assets/aa884a1a-8bd5-42de-a701-effedba57dbe" />


**NS2 / Valmar (slave)**

```
zone "3.75.10.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.10.75.3.rev";
    masters { 10.75.3.3; };
};
```

**Verifikasi**

```# dari klien
dig static.k23.com A +short       # 10.75.3.5
curl -I http://static.k23.com/ | head -n1          # 200 OK
curl -s  http://static.k23.com/annals/ | sed -n '1,5p'  # tampil listing

# akses via IP harus 403
curl -I http://10.75.3.5/ | head -n1               # 403 Forbidden
```

<img width="605" height="325" alt="Soal 8 konfigurasi velmar (6)" src="https://github.com/user-attachments/assets/dddff811-1fe8-4319-976b-b6a24129816f" />

<img width="351" height="191" alt="Soal 8 hasil" src="https://github.com/user-attachments/assets/cbc1051c-ab4f-44a2-b86d-a1e6ddfb81ec" />


### Nomor 9

**Tujuan**

Menjalankan web statis di Lindon dengan:

- Hostname wajib: static.k23.com

- Folder /annals/ autoindex on

- Akses via IP ditolak (403)

**Pengerjaan (Lindon – 10.75.3.5)**

```
apt-get update && apt-get install -y nginx

mkdir -p /var/www/static/public/annals
echo "buldak.txt" > /var/www/static/public/annals/buldak.txt

cat >/etc/nginx/sites-available/static.k23.com <<'EOF'
server {
    listen 80;
    server_name static.k23.com;
    root /var/www/static/public;
    index index.html;

    location /annals/ {
        autoindex on;
    }
}
# akses via IP ditolak
server { listen 80 default_server; return 403; }
EOF

ln -sf /etc/nginx/sites-available/static.k23.com /etc/nginx/sites-enabled/static.k23.com
nginx -t && (pkill nginx 2>/dev/null || true) && nginx
```

<img width="946" height="223" alt="soal 9 konfigurasi lindon bagian kedua" src="https://github.com/user-attachments/assets/7a575309-597f-494c-ab25-ee072755acfb" />

**Verivikasi**

```
# dari klien
dig static.k23.com A +short       # 10.75.3.5
curl -I http://static.k23.com/ | head -n1          # 200 OK
curl -s  http://static.k23.com/annals/ | sed -n '1,5p'  # tampil listing

# akses via IP harus 403
curl -I http://10.75.3.5/ | head -n1               # 403 Forbidden
```

<img width="452" height="326" alt="Soal 9 hasil" src="https://github.com/user-attachments/assets/b62ccd69-6413-4c44-a0a9-0cebd32e3e1d" />


### Nomor 10

**Tujuan**

Menjalankan web dinamis di Vingilot:

- PHP-FPM aktif

- Home (/) dan about tanpa .php

- Akses via hostname wajib

- Akses via IP ditolak

**Pengerjaan (Vingilot – 10.75.3.6)**

1. Install & pastikan PHP-FPM listen TCP :9000

```
apt-get update && apt-get install -y nginx php-fpm
POOL=$(ls /etc/php/*/fpm/pool.d/www.conf | head -n1)
sed -i 's~^listen = .*~listen = 127.0.0.1:9000~' "$POOL"

pkill php-fpm 2>/dev/null || true
PHPFPM=$(command -v php-fpm || ls /usr/sbin/php-fpm* | head -n1)
"$PHPFPM" -D
ss -ltnp | grep ':9000'     # harus muncul
```

2. Dokumen web + vhost Nginx

```mkdir -p /var/www/app/public
cat >/var/www/app/public/index.php <<'PHP'
<?php echo "<h1>Hello from Vingilot (app.k23.com)</h1>"; ?>
PHP

cat >/var/www/app/public/about.php <<'PHP'
<?php echo "<h1>About Vingilot</h1>"; ?>
PHP

cat >/etc/nginx/sites-available/app.k23.com <<'EOF'
server {
    listen 80;
    server_name app.k23.com;
    root /var/www/app/public;
    index index.php;

    location / { try_files $uri $uri/ /index.php?$query_string; }
    location = /about { try_files $uri /about.php; }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass 127.0.0.1:9000;
    }
}
server { listen 80 default_server; return 403; }
EOF

ln -sf /etc/nginx/sites-available/app.k23.com /etc/nginx/sites-enabled/app.k23.com
nginx -t && (pkill nginx 2>/dev/null || true) && nginx

```

<img width="515" height="450" alt="nomer 10 konfigurasi vingilot" src="https://github.com/user-attachments/assets/446e43c0-c3f1-4014-8b31-e0a40a6d61c2" />


**Verivikasi**

```# lokal di Vingilot (pakai Host header)
curl -I -H 'Host: app.k23.com' http://127.0.0.1/ | head -n1      # 200 OK
curl -I -H 'Host: app.k23.com' http://127.0.0.1/about | head -n1 # 200 OK

# dari klien
dig app.k23.com A +short            # 10.75.3.6
curl -I http://app.k23.com/ | head -n1
curl -I http://app.k23.com/about | head -n1

# via IP ditolak
curl -I http://10.75.3.6/ | head -n1     # 403 Forbidden
```

<img width="452" height="94" alt="soal nomer 10 hasil" src="https://github.com/user-attachments/assets/7d59b285-3735-42a4-be5d-0943a29ab1b6" />

