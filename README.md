# Konfigurasi DNS Server Lokal menggunakan BIND9 (Proxmox LXC)

Dokumentasi langkah demi langkah instalasi dan konfigurasi **BIND9 DNS Server** pada sistem operasi Linux (Ubuntu/Debian Container di Proxmox VE).

> **Catatan Penyesuaian (Variabel):**
> Jika Anda menggunakan konfigurasi ini untuk jaringan atau tugas Anda sendiri, sesuaikan bagian berikut:
> * **Domain Utama:** `fatihmahardika.xyz` → ganti dengan domain Anda.
> * **Alamat IP Server:** `192.168.10.23` → ganti dengan IP server Anda.
> * **Segmen Reverse IP:** `10.168.192` → ganti dengan 3 oktet pertama IP Anda yang dibalik.
> * **Host ID / Oktet Terakhir:** `23` → ganti dengan angka terakhir IP Anda (`1`–`254`).

---

## 📋 Daftar File yang Dikonfigurasi

1. `/etc/bind/named.conf.local` — Mendaftarkan zona *forward* dan *reverse*.
2. `/etc/bind/named.conf.options` — Mengatur *DNS forwarders* dan validasi DNSSEC.
3. `/etc/bind/db.fatihmahardika` — Database *forward zone* (Domain ke IP), disalin dari `db.local`.
4. `/etc/bind/db.192` — Database *reverse zone* (IP ke Domain), disalin dari `db.127`.
5. `/etc/resolv.conf` — Mengarahkan DNS resolver lokal ke server BIND9.

---

## 🚀 Langkah-Langkah Konfigurasi

### 1. Instalasi Paket BIND9
Perbarui repositori dan instal paket BIND9 beserta utilitas pengujian DNS:

```bash
apt update
apt install bind9 bind9utils bind9-doc dnsutils -y
```

---

### 2. Daftarkan Zona di `named.conf.local`
Buka file `/etc/bind/named.conf.local` dan tambahkan konfigurasi zona berikut:

```bash
nano /etc/bind/named.conf.local
```

Isi konfigurasi:
```text
zone "fatihmahardika.xyz" {
    type master;
    file "/etc/bind/db.fatihmahardika";
};

zone "10.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
};
```

---

### 3. Konfigurasi Forward Zone (`db.fatihmahardika`)
Salin template bawaan `db.local`:
```bash
cp /etc/bind/db.local /etc/bind/db.fatihmahardika
nano /etc/bind/db.fatihmahardika
```

Ubah isinya menjadi:
```text
;
; BIND data file for fatihmahardika.xyz
;
$TTL    604800
@       IN      SOA     ns1.fatihmahardika.xyz. root.fatihmahardika.xyz. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.fatihmahardika.xyz.
@       IN      A       192.168.10.23
ns1     IN      A       192.168.10.23
www     IN      A       192.168.10.23
```

---

### 4. Konfigurasi Reverse Zone (`db.192`)
Salin template bawaan `db.127`:
```bash
cp /etc/bind/db.127 /etc/bind/db.192
nano /etc/bind/db.192
```

Ubah isinya menjadi:
```text
;
; BIND reverse data file for local 192.168.10.0 network
;
$TTL    604800
@       IN      SOA     ns1.fatihmahardika.xyz. root.fatihmahardika.xyz. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.fatihmahardika.xyz.
23      IN      PTR     fatihmahardika.xyz.
23      IN      PTR     www.fatihmahardika.xyz.
```

---

### 5. Atur Forwarders di `named.conf.options`
Buka file `/etc/bind/named.conf.options` agar server dapat meneruskan permintaan domain internet ke DNS publik dan mencegah error `FORMERR`:

```bash
nano /etc/bind/named.conf.options
```

Sesuaikan bagian `options`:
```text
options {
        directory "/var/cache/bind";

        forwarders {
                8.8.8.8;
                1.1.1.1;
        };

        dnssec-validation no;

        listen-on-v6 { any; };
};
```

---

### 6. Atur DNS Resolver (`resolv.conf`)
Pastikan server mengarah ke IP DNS lokalnya sendiri. *(Catatan: Pada Proxmox LXC, ubah juga pengaturan pada tab **DNS** container agar tidak tertimpa saat reboot)*.

```bash
nano /etc/resolv.conf
```

Isi konfigurasi:
```text
search fatihmahardika.xyz
nameserver 192.168.10.23
nameserver 8.8.8.8
```

---

### 7. Validasi & Restart Service
Cek apakah ada kesalahan sintaks pada konfigurasi zona sebelum me-restart layanan BIND9:

```bash
named-checkconf -z
named-checkzone fatihmahardika.xyz /etc/bind/db.fatihmahardika
named-checkzone 10.168.192.in-addr.arpa /etc/bind/db.192
systemctl restart bind9
```

---

## ✅ Pengujian (Testing)

Jalankan perintah berikut untuk memastikan resolusi *Forward* dan *Reverse* berjalan normal:

```bash
nslookup fatihmahardika.xyz
nslookup 192.168.10.23
```

**Contoh Output Berhasil:**
```text
Server:         192.168.10.23
Address:        192.168.10.23#53

Name:   fatihmahardika.xyz
Address: 192.168.10.23
```
