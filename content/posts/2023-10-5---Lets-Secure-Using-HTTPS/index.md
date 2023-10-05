---
title: "Securing Website with HTTPS (Domain) using Certbot and Google Compute Engine"
date: "2023-10-05T22:10:52"
template: "post"
draft: false
slug: "/posts/make-website-https"
category: "Technology"
tags:
  - "Web Development"
  - "Cloud Computing"
  - "Google Cloud Platform"
description: "Let's Encrypt adalah Otoritas Sertifikat (CA) yang memfasilitasi sertifikat TLS/SSL gratis, yang memungkinkan HTTPS terenkripsi pada server web. Certbot pada juga akan secara otomatis untuk mendapatkan sertifikat. Proses mendapatkan dan menginstal sertifikat sepenuhnya diotomatisasi pada Apache dan Nginx. Dalam tutorial kali ini, kita akan menggunakan Certbot untuk sertifikat SSL gratis Apache di Debian 11 dan membuat pembaruan sertifikat secara otomatis."
socialImage: "./media/certbot.png"
---

## Persiapan

Pada tutorial kali ini kita membutuhkan:

- Google Cloud Platform (Trial/Berbayar)
- Domain (Bebas mau pake ekstensi apa, disini kita pake .my.id)

### Membuat External IP Static 

Tujuannya agar pada saat dimatikan atau mengalami kejadian yang tidak terduga, maka IP address tetap sama dan tidak perlu di konfigurasi ulang.

Pergi ke:

VPC Network -> IP Addresses -> RESERVE EXTERNAL IP ADDRESSES.

![vpc-1](./media/vpc-1.png)

_Abaikan kedua tersebut, karena saya sudah mempunyai 2 VM_.

Kemudian setting:

- Nama: (Bebas)

Disini saya memakai nama _ip-static-jkt_, karena saya akan membuat static IP di Jakarta.

- Network Service Tier: (Bebas juga)

Perbedaan Premium dan Standard adalah kalo premium terbuka semua IP (IPv4 dan IPv6). Lalu untuk region dan global juga terbuka semua. 

- IP version: IPv4

Sebenernya ini opsional, karena yang paling sering menggunakan IPv4.

- Type: Regional

Ini juga opsional tapi kita akan memakai regional agar cepat (target client orang indo misalnya).

- Region: (Bebas)

Kita pilih asia-southeast2 karena ini server Jakarta, tapi bebas aja jika ingin region lain.

- Attached: None

Isi none dulu, karena kita belum membuat instance/VM.

Gasss, langsung create/reserve.


![vpc-2](./media/vpc-2.png)

Kurang lebih jika sudah selesai akan seperti ini:

![vpc-3](./media/vpc-3.png)

### Membuat Instance / Virtual Machine dan Konfigurasinya

Seperti biasa, langsung saja ke Compute Engine -> VM Instance -> Create Instance.

Konfigurasikan:

- Nama: (Bebas)
- Region: (Sesuaikan dengan yang tadi, disini kita memakai asia-southeast2 atau Jakarta)
- Zone: (Bebas)
- Machine Type: (bebas ambil yang murah aja kalo buat coba-coba)
- Boot Disk: (Bebas, tapi harus pake debian. Karena ini tutor pake debian eheheh)
- Firewall:

![vm-1](./media/vm-1.png)

Lalu kita ke Advanced Option, lalu ke Networking dan Network Interfaces.

Biarkan semua default, kecuali pada External IPv4 Addresses. Kita ganti menjadi IPv4 yang tadi kita sudah buat diawal (Disini dengan nama ip-static-jkt).

![vm-2](./media/vm-2.png)

_Untuk IP Internal kita biarkan random saja, bisa juga dicustom, caranya tidak jauh berbeda dengan ini_.

Lalu pencet done dan create. Tunggu hingga Instance/VM selesai dibuat.

Jika sudah selesai, kita masuk memakai SSH. Bebas sih sebenernya mau pake apa juga, yang penting masuk VM dulu.

Setelah masuk, kita install apache2 untuk web servernya.

```
sudo apt-get update
sudo apt-get install apache2 -y
```

![vm-3](./media/vm-3.png)

Oke, lalu kita coba pastikan apache2 sudah aktif dengan cara kita bisa mengunjungi External IP Address yang ada atau bisa juga dengan perintah:

```
sudo systemctl status apache2
```

![vm-4](./media/vm-4.png)

setelah itu kita minimize/tutup sementara terminalnya.

### Setting DNS

Kita masuk ke Network Services -> Cloud DNS -> Create Zone.

Lakukan setting:

- Zone Type: Public
- Zone Name: (Bebas)
- DNS Name: <Nama_Domain>

Sisanya biarkan default. Lalu Create.

![dns-1](./media/dns-1.png)

Lalu kita sesuaikan dengan cara tekan nama yang tadi kita sudah buat.

![dns-2](./media/dns-2.png)

Kita akan membuat konfigurasi baru, dengan tekan Add Standard.

Masukan IPv4 Address External dari VM yang tadi sudah kita buat, untuk DNS Name kita kosongkan saja. Dan tekan Create.

![dns-3](./media/dns-3.png)

Lalu kita buat Record baru lagi seperti tadi.

Untuk DNS Name kita isi dengan www, untuk Resource ganti dengan CNAME, dan tambahkan domain kalian.

![dns-4](./media/dns-4.png)

Tekan tanda panah, untuk menampilkan Name server

![dns-5](./media/dns-5.png)

Lalu kita pergi ke manage domain, disini saya memakai rumah web. (Menu untuk Name Server pasti ada disemua penyedia layanan domain, cuma letaknya saja yang berbeda)

Kita pergi ke menu Name Server

![dns-6](./media/dns-6.png)

Isikan Domain Server tadi, lalu change name server.

![dns-7](./media/dns-7.png)

_Pastikan mengisi domain server tanpa tanda apapun dibelakangnya, misalnya titik (pada contoh gambar)_.

![dns-8](./media/dns-8.png)

Setelah sukses, tunggu beberapa menit atau max 24 jam. Selagi menunggu kita lanjut ke step berikutnya.


### Install Certbot untuk membuat HTTPS

Lakukan perintah:

```
sudo apt install certbot python3-certbot-apache
```

Lalu kita membuka file konfigurasi dari apache2 dengan perintah:

```
cd /etc/apache2/sites-available/
```

Lalu kita lihat disana ada file apa saja dengan ```ls```, karena kita default maka kita pake _000-default.conf_. Atau jika kalian sudah mempunyai configuration sendiri bisa memakainya. Lalu kita edit

```
sudo nano 000-default.conf
```

![vm-5](./media/vm-5.png)

Lalu tambahkan dengan:

```
ServerName <nama_domain>
ServerAlias <www.nama_domain>
```

![vm-6](./media/vm-6.png)

Setelah itu save.

Lalu masukan perintah:

```
sudo apache2ctl configtest
```

pastikan respon Syntax OK, dan lakukan reload server

```
sudo systemctl reload apache2
```

![vm-7](./media/vm-7.png)

Lalu masukan perintah untuk membuat SSL:

```
sudo certbot --apache
```

- Pada bagian email bisa kalian isikan email kalian untuk mendapatkan notifikasi.

- Pada Term of Service bisa kita YES

- Pada Sponsorship bisa kita NO (Atau YES juga bisa)

- Pada menu pilih domain kita biarkan kosong (Langsung Enter) itu berarti semuanya kita pasang sertifikat.

![cb-1](./media/certbot-1.png)

![cb-2](./media/certbot-2.png)

Seharusnya sampai disini, kita sudah SELESAI karena sudah tersambung dengan domain dan terpasang Sertifikat SSL.

![cb-3](./media/certbot-3.png)

### Membuat Auto Renew Sertifikat / SSL

Lakukan perintah untuk memastikan ssl terpasang dengan baik.

```
sudo certbot renew --dry-run
```

![cb-4](./media/certbot-4.png)

Lalu masukan perintah untuk mengatur jadwal pembaruan otomatis.

```
sudo crontab -e
```

dan masukan kode, untuk mengatur 1x 12jam, atau 2x dalam 24jam. Ini opsional bisa diganti sesuai keperluan.

```javascript
0 */12 * * * /usr/bin/certbot renew --quiet
```

![certbot-6](./media/certbot-6.png)

Kemudian save dan exit.

### Selesai