# LAPRES Praktikum Kmunikasi Data dan Jaringan Komputer Modul 1 - K-23

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

Untuk setup ada pada node Tirion dan Valmar.

Setup 1 Valmar

A. Ubuntu

```
apt-get update
apt-get install -y bind9 bind9-utils dnsutils

cat > /etc/bind/named.conf.options <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
}
EOF

cat > /etc/bind/named.conf.local <<'EOF'
zone "k23.com" {
    type slave;
    masters { 10.75.3.3; };                  // ns1 (Tirion)
    file "/var/cache/bind/db.k23.com";       // akan terisi via AXFR
};
EOF

named-checkconf
service bind9 restart || /etc/init.d/bind9 restart
```

B. Alpine 
```
apk update
apk add bind bind-tools

cat > /etc/bind/named.conf <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
};

zone "k23.com" {
    type slave;
    masters { 10.75.3.3; };
    file "/var/cache/bind/db.k23.com";
};
EOF

rc-service named restart || /etc/init.d/named restart

```

Setup Tirion

A. Ubuntu

```
# /etc/bind/named.conf.options
cat > /etc/bind/named.conf.options <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
}
EOF

# /etc/bind/named.conf.local
cat > /etc/bind/named.conf.local <<'EOF'
zone "k23.com" {
    type master;
    file "/etc/bind/zones/db.k23.com";
    allow-transfer { 10.75.3.4; };  // Valmar (ns2)
    also-notify   { 10.75.3.4; };
    notify yes;
};
EOF

# file zona
mkdir -p /etc/bind/zones
cat > /etc/bind/zones/db.k23.com <<'EOF'
$TTL 300
@   IN SOA ns1.k23.com. admin.k23.com. (
        2025101301 ; Serial
        3600       ; Refresh
        900        ; Retry
        1209600    ; Expire
        300 )      ; Minimum

    IN NS ns1.k23.com.
    IN NS ns2.k23.com.

ns1 IN A 10.75.3.3     ; Tirion
ns2 IN A 10.75.3.4     ; Valmar
@   IN A 10.75.3.2     ; Sirion (apex)
EOF

# validasi & reload
named-checkzone k23.com /etc/bind/zones/db.k23.com
named-checkconf
service bind9 restart || /etc/init.d/bind9 restart

```

B. Alpine

```
Di Alpine, paling mudah gabungkan options & zone ke satu file named.conf:

cat > /etc/bind/named.conf <<'EOF'
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 192.168.122.1; };
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
};

zone "k23.com" {
    type master;
    file "/etc/bind/db.k23.com";
    allow-transfer { 10.75.3.4; };
    also-notify   { 10.75.3.4; };
    notify yes;
};
EOF

cat > /etc/bind/db.k23.com <<'EOF'
$TTL 300
@   IN SOA ns1.k23.com. admin.k23.com. (
        2025101301 ; Serial
        3600
        900
        1209600
        300 )
    IN NS ns1.k23.com.
    IN NS ns2.k23.com.
ns1 IN A 10.75.3.3
ns2 IN A 10.75.3.4
@   IN A 10.75.3.2
EOF

# restart named
rc-service named restart || /etc/init.d/named restart

```

**Cara memverivikasi**

Pada Valmar

```
dig @10.75.3.4 k23.com SOA +noall +answer +authority
dig @10.75.3.4 NS  k23.com +noall +answer
dig @10.75.3.4 A   k23.com +noall +answer
ls -l /var/cache/bind/db.k23.com
```

Pada Tirion

```
dig @10.75.3.3 k23.com SOA +noall +answer +authority
dig @10.75.3.3 NS  k23.com +noall +answer
dig @10.75.3.3 A   k23.com +noall +answer
```

**Hasil Valmar**
<img width="814" height="137" alt="nomer 4 valmar slave (2)" src="https://github.com/user-attachments/assets/2479a88f-2de0-4300-9e30-39e6e77182e7" />

**Hasil Tirion**
<img width="836" height="124" alt="nomer 4 tirion master (2)" src="https://github.com/user-attachments/assets/494a4ae9-64f1-4b00-845e-5dc31b18c4f1" />




