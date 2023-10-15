---
title: "Deploy Aplikasi Website Memakai NodeJS, Mysql dan CodeIgniter - Google Cloud Compute Engine"
date: "2023-10-15T00:12:16"
template: "post"
draft: false
slug: "/posts/deploy-nodejs-and-codeigniter"
category: "Technology"
tags:
  - "Web Development"
  - "Cloud Computing"
  - "Google Cloud Platform"
description: "Tutorial cara mendeploy aplikasi frontend berbasis website html memakai framework Bootstrap dan CodeIgniter versi 3 dan pada backend memakai nodejs serta untuk database memakai mysql atau mariadb, semua dilakukan pada Google Cloud Platform Compute Engine."
socialImage: "./media/nodejs.jpg"
---

Kita akan membuat sebuah project yang bernama **Duitmu**

- [Storage](#storage)
- [Database](#database)
- [Backend](#backend)
- [Frontend](#frontend)

## Storage

### Membuat Storage Bucket

**Konfigurasi:**

- Nama: duitmu-storage

- Lokasi: asia-southeast2 (Jakarta)

- Tipe Storage: Standard

- Control Access: Default (Tidak ada yang diubah)

- Protect Object: Default (Tidak ada yang diubah)

Lalu Create.

![storage1](./media/storage1.png)

## Database

### Membuat VM Instance

**Konfigurasi:**

- Nama: database-server

- Region: asia-southeast2

- Zone: asia-southeast2-a

- Machine configuration:

  - E2: e2-micro (2vCpu, 1 Core, 1 Memory)

- Boot Disk:

  - Image: Debian 11

  - Storage Type: SSD persistent disk

  - Size: 10GB

![db1](./media/db1.png)

- Firewall

  - Allow HTTP Traffic: ON

  - Allow HTTPS Traffic: ON

  - Allow Load Balancer Health Checks: ON

![db2](./media/db2.png)

- Management

  - Startup script

```sh
sudo apt-get update
sudo apt-get install git -y
sudo apt install mariadb-server -y
```

![db3](./media/db3.png)

_Note: Untuk install git itu opsional._

Lalu Create.

### Membuat User

Aktifkan sebagai Super User dengan perintah `sudo su`.

Lalu ketikan:

```sql
create user '<user>'@'%' identified by '<password>';
```

```sql
grant all privileges on *.* to '<user>'@'%' with grant option;
```

```sql
flush privileges;
```

![db4](./media/db4.png)

Setelah itu, kita akan membuat database beserta tablenya.

```sql
CREATE DATABASE <nama_database> DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
```

Masuk ke dalam database tadi

```sql
use <nama_database>;
```

Lalu, buat table:

```sql
CREATE TABLE records (
    id INT AUTO_INCREMENT PRIMARY KEY NOT NULL,
    name VARCHAR(25) NOT NULL,
    amount DOUBLE NOT NULL,
    date DATETIME NOT NULL,
    notes TEXT,
    attachment VARCHAR(255)
);
```

![db5](./media/db5.png)

### Konfigurasi Firewall

Agar database dapat diakses dari luar Instance/VM kita akan membuka port 3306.

Pergi ke VPC Network -> Firewall -> Create Rule

**Konfigurasi:**

- Name: fw-mysql

- Direction of traffic: Ingress

- Target Tags: fw-mysql

- Source IP Ranges: 0.0.0.0/0

- Protocols and ports: TCP:3306

Lalu Create.

![db6](./media/db6.png)

Setelah selesai, kita balik ke menu Compute Engine, lalu kita edit Instancee/VM database-server tadi. Kita akan menambahkan tag baru yang tadi kita sudah buat dengan nama fw-mysql.

![db7](./media/db7.png)

Lalu Save.

### Konfigurasi Mysql

Kita masuk ke Instance/VM database-server atau nama mesin yang kalian beri tadi.

Lalu ketikan:

```sh
sudo grep -r "bind-address" /etc/mysql/
```

Nanti akan muncul beberapa file, kemudian ubah yang 127.0.0.1 (Localhost) menjadi 0.0.0.0 (Everyone).

Lalu save dan keluar, lalu restart mysql

```sh
 sudo systemctl restart mysql
```

![db8](./media/db8.png)

![db9](./media/db9.png)

### Test Koneksi

Kalian bisa testing port yang terbuka melalui website: [Link](https://www.yougetsignal.com/tools/open-ports/)

Untuk melakukan testing koneksi pada database, disini kita memakai Tools yang bernama TablePlus.

Masukan data-data yang dibutuhkan:

- Nama: Bebas

- Host: IP Eksternal dari VM

- Port: 3306

- User: User yang tadi kita buat

- Password: Password yang tadi kita buat

- Database: Database yang tadi kita buat

![db10](./media/db10.png)

Setelah masuk, akan tampil seperti ini.

![db11](./media/db11.png)

Setting Database Server sudah selesai.

## Backend

### Membuat Service Account

Agar aplikasi kita dapat mengakses Storage Bucket (untuk menyimpan gambar), maka kita perlu membuat service account role.

Pergi ke IAM & Admin -> Service Account -> Create Service Account

**Konfigurasi:**

- Name: Bebas

- Role:

  - Editor

  - Bucket Object Admin

- Sisanya Defaultkan saja

![be1](./media/be1.png)

Setelah itu, kita masuk ke service account dan generate key dengan tipe JSON.

![be2](./media/be2.png)

Secara otomatis akan mendownload file, yang isinya adalah key atau secret token untuk aplikasi kita.

### Membuat VM Instance

Seperti biasa langsung saja ke Compute Engine -> VM Instance -> Create

**Konfigurasi:**

- Nama: backend-server

- Region: asia-southeast2

- Zone: asia-southeast2-a

- Machine configuration:

  - E2: e2-micro (2vCpu, 1 Core, 1 Memory)

- Boot Disk:

  - Image: Debian 11

  - Storage Type: SSD persistent disk

  - Size: 10GB

- Firewall

  - Allow HTTP Traffic: ON

  - Allow HTTPS Traffic: ON

  - Allow Load Balancer Health Checks: ON

- Management

  - Startup script

```sh
sudo apt-get update
sudo apt-get install git -y
sudo apt install nodejs npm -y
```

![be3](./media/be3.png)

Lalu Create. Tunggu beberapa menit untuk menunggu pembuatan mesin dan melakukan penginstallan awal.

Pastikan Nodejs dan NPM sudah terpasang.

```
nap@backend-server:~$ npm -v
7.5.2

nap@backend-server:~$ nodejs -v
v12.22.12
```

### Konfigurasi File

Pada kali ini kita menggunakan github untuk mendownload file.

Silahkan gunakan perintah ```git clone <link repository kalian>```

Pada kasus kali ini, saya memerlukan sedikit konfigurasi pada file.

- serviceaccountkey.json (service account untuk upload image ke Cloud Storage). 

  *Untuk bagian ini, silakan masukan service account yang tadi kita sudah buat diawal dengan hak akses yang memungkinkan aplikasi mengunggah gambar ke Cloud Storage bucket, lalu buka file yang berisi key, kemudian copy paste isi file-nya ke serviceaccountkey.json (timpa isi file sebelumnya).*

- modules -> imgUpload.js (Cloud Storage).

- routes -> record.js (Cloud SQL).

Isi dari service key kurang lebih seperti ini:

```json
  GNU nano 5.4                                 serviceaccountkey.json                                          
{
    "type": "service_account",
    "project_id": "project_id_Anda",
    "private_key_id": "private_key_id_Anda",
    "private_key": "private_key_Anda",
    "client_email": "upload-image@ancient-sunspot-328301.iam.gserviceaccount.com",
    "client_id": "client_id_Anda",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "client_cert_url_Anda"
}
```

Untuk database, kita masukan host IP eksternal pada VM Instance yang bernama database-server.

![be4](./media/be4.png)

### Menginstall PM2

Tujuan menginstall PM2 adalah, jika VM kita crash atau mati. Maka PM2 bisa menghidupkan dan mengkonfigurasi secara otomatis.

Install dengan perintah:

```sh
sudo npm install pm2 -g
```

```sh
sudo pm2 startup
```

Lalu masukan perintah untuk menjalankan server kita. 

```sh
sudo pm2 start <nama file server kita> --name <nama bebas untuk labeling>
```

Contoh:

```sh
sudo pm2 start app.js --name backend-api
```

![be5](./media/be5.png)

Setelah itu kita pastikan server sudah berjalan, dengan perintah:

```sh
curl localhost
```

Jika muncul error atau peringatan connection refused, maka kita harus mengkonfigurasi firewall sesuai dengan port server yang kita gunakan.

![be6](./media/be6.png)

Jika error silahkan ikuti step selanjutnya, jika suda berjalan bisa skip ke bagian frontend.

### Konfigurasi Firewall Backend

Pergi ke VPC Network -> Firewall -> Create Firewall

Setelah itu sesuaikan dengan port pada aplikasi kita, misal disini saya menggunakan port 8000.

- Name: Bebas

- Target tag: Bebas

- Source IP Ranges: 0.0.0.0/0 (ALL atau kalo mau lebih spesifik juga boleh)

- Direction of traffic: Ingress

- Protocols and ports:

  - TCP:8000 (Centang TCP dan masukan port 8000)

Lalu create Firewall.

![be7](./media/be7.png)

Setelah itu pergi ke VM Instance yang bernama backend-server, lalu edit dan masukan tag yang tadi kita sudah buat di firewall.

![be8](./media/be8.png)

Jika sudah, kita cek kembali dengan perintah

```
curl localhost:port
```

![be9](./media/be9.png)

Disini server saya memberikan response sukses. Yang berarti sudah berhasil

## Frontend

### Membuat VM Instance

Seperti biasa langsung saja ke Compute Engine -> VM Instance -> Create

**Konfigurasi:**

- Nama: frontend-server

- Region: asia-southeast2

- Zone: asia-southeast2-a

- Machine configuration:

  - E2: e2-micro (2vCpu, 1 Core, 1 Memory)

- Boot Disk:

  - Image: Debian 11

  - Storage Type: SSD persistent disk

  - Size: 10GB

- Firewall

  - Allow HTTP Traffic: ON

  - Allow HTTPS Traffic: ON

  - Allow Load Balancer Health Checks: ON

- Management

  - Startup script

```sh
sudo apt-get update
sudo apt-get install git -y
sudo apt install php libapache2-mod-php php-mysql php-mbstring php-zip php-gd php-json php-curl -y
```

*Untuk PHP itu opsional ya.*

### Konfigurasi File

Setelah itu kita ambil file source code kita dari github. Seperti biasa gunakan perintah:

```git clone <link repository>```

Pada aplikasi saya membutuhkan sedikit konfigurasi tambahan.

- Pada file application -> config -> config.php (base URL dari Front-End).

- Pada file application -> models -> Record_model.php (base URL dari Back-End).

### Konfigurasi Server

Pada kali ini kita akan menggunakan apache sebagai frontend servernya. Silahkan install apache2:

```sh
sudo apt-get install apache2
```

lalu masuk ke folder frontend kita:

```cd <nama file>```

dan pindahkan semua file yang ada di dalam folder ke /var/www/html. Dengan perintah:

```sudo mv ./* /var/www/html```

Karena default server apache adalah index.html dan file utama saya index.php, maka saya hapus file index.html. Menggunakan perintah:

```sudo rm index.html```

![fe1](./media/fe1.png)

Setelah itu kita akan membuat Routing pada file configurasi apache. Gunakan perintah:

```sh
cd /etc/apache2/sites-available
```

Lalu edit file 000-default.conf, dan masukan kode:

```
<Directory /var/www/html>
  Options Indexes FollowSymLinks
  AllowOverride All
  Require all granted
</Directory>
```

![fe2](./media/fe2.png)

Setelah itu pergi ke ```/var/www/html``` kita akan membuat file .htaccess:

```
RewriteEngine On
#RewriteBase /
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule (.*) index.php/$1
```

Lalu save dan restart server apache2:

```
sudo a2ensite 000-default.conf
sudo a2enmod rewrite
sudo systemctl restart apache2
```

## Test Website Aplikasi

### Testing

Copy IP Eksternal frontend-server kita ke browser (bebas browser apa aja). Lalu coba semua fitur aplikasi kita, pastikan semua berjalan dengan lancar.

## Selesai