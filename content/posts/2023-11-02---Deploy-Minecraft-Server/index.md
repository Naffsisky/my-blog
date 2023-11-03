---
title: "Deploy Minecraft Server + Auto Backup"
date: "2023-11-03T00:09:16"
template: "post"
draft: false
slug: "/posts/deploy-minecraft-server"
category: "Technology"
tags:
  - "Minecraft"
  - "Games"
  - "Google Cloud Platform"
description: "Deploy server minecraft menggunakan VM Instance di Google Cloud Platform dan menggunakan Rclone + Cronjob untuk membackup seluruh file ke dalam Cloud Storage. Cloud Storage bisa berupa File Storage dari Google Cloud Platform, Google Drive, Microsoft One Drive, atau juga Cloud Storage as a Service lainnya."
socialImage: "./media/intro.png"
---

## DAFTAR ISI

- [Membuat Firewall](#1-membuat-firewall)

- [Membuat VM Instance](#2-membuat-vm-instance)

- [Rclone](#3-konfigurasi-rclone)

- [Minecraft Server](#4-membuat-server-minecraft)

- [Cronjob](#5-membuat-cronjob)

- [Monitoring](#6-monitoring-cronjob)

## Intro

Kira-kira untuk Infrastrukturnya seperti ini.

![infra](./media/infra.png)

Dari VM Instance utama akan dibagi menjadi 3 screen, yaitu untuk Minecraft Server, Rclone, dan untuk Monitoring Cronjob.

Agar rclone auto backup, kita menggunakan bantuan Cronjob.

Server ini sederhana saja, hanya membutuhkan VM Instance dari Compute Engine dan Port 25565 pada firewall.

## 1. Membuat Firewall

VPC Network -> Firewall -> Create New Rule

1. Konfigurasikan:

    Nama Firewall : <nama_firewall> (ini bebas)

    Network : <nama_network> (bebas/default)

    Direction of traffic : Ingress
    
    Targets : Specified target tags
    
    Tags : <nama_tag> (bebas)
    
    Port : TCP 25565

![firewall](./media/firewall.png)

Silahkan create firewall.

## 2. Membuat VM Instance

Konfigurasikan sesuka kalian dengan spek yang sesuai dengan kebutuhan. Disini saya menggunakan seri N2-Standard-2 dengan 2vcpu + 1core dan ram 8gb.

Lalu jangan lupa tambahkan tag yang sudah kita buat di awal pada bagian Network. Hal ini bertujuan untuk membuka port server minecraft.

Jika kalian masih kebingungan untuk membuat VM Instance, mungkin bisa mengunjungi artikel [ini](https://blog.napuniverse.me/posts/make-website-https) pada bagian pembuatan VM Instance.

![VM Instance 1](./media/vm-1.png)

Untuk bagian Network Interface, saya sarankan menggunakan IP STATIC EXTERNAL. Karena agar IP tersebut tidak berubah-rubah. Namun, saya disini tidak akan memberikan tutorial untuk membuat atau reserve IP STATIC EXTERNAL. Mungkin bisa lihat [disini](https://blog.napuniverse.me/posts/make-website-https) pada bagian Reserve IP.

![VM Instance 2](./media/vm-2.png)

Pada bagian Management -> Startup Script. Kita akan buat seperti ini:

```
sudo apt-get update
sudo apt install openjdk-17-jre-headless -y
sudo apt-get install rclone
sudo apt-get install fuse3 -y
```

Kita akan menginstall Java, Rclone, dan Fuse3 untuk berjaga-jaga dalam mounting nanti. Setelah itu bisa di Create, lalu tunggu hingga pembuatan VM selesai.

## 3. Konfigurasi Rclone

### Setup Config

Mungkin untuk konfigurasi Rclone bisa dilihat [disini](https://blog.napuniverse.me/posts/rclone-auto-backup). Tetapi hanya sampai menjalankan server saja.

Saya membuat dengan nama minecraft-backup dan untuk cloud storage Microsoft Onedrive.

![rclone 1](./media/rclone-1.png)

### Setup Mount / File

Pindah screen untuk aktifkan rclone (kita akan membuat screen baru) dengan perintah:

```
screen -S <nama_screen> 
```

Misal:

```
screen -S rclone-backup
```

Selanjutnya kita akan membuat file untuk rclone (sebagai wadah untuk rclonenya). 

Simplenya gini, semua file yang ada di cloud kita akan tersedia di file local ini.

![infra rclone](./media/rclone.png)

Setelah itu kita Mounting:

Mounting rclone dengan perintah:

```
rclone mount <nama_drive>: ~/<nama folder lokal>/ --vfs-cache-mode writes
```

![rclone 2](./media/rclone-2.png)

Setelah itu tidak akan ada respon apa-apa, kalian keluar screen saja dengan menekan ```ALT + A + D```. Kalian akan diarahkan ke screen utama.

Lalu cek folder kalian di local, apakah sudah ada file-file atau folder yang di dalam cloud (pindah ke local).

## 4. Membuat Server Minecraft

Pindah screen untuk menginstall server minecraft (kita akan membuat screen baru) dengan perintah:

```
screen -S <nama_screen> 
```

Misal:

```
screen -S minecraft
```

Download server minecraft

```
wget https://piston-data.mojang.com/v1/objects/5b868151bd02b41319f54c8d4061b8cae84e665c/server.jar
```

Silahkan kunjungi website minecraft server untuk mendapatkan file terbaru [disini](https://www.minecraft.net/en-us/download/server).

Kalian buat folder dan hasil download yang tadi bisa dimasukan kedalamnya. Seperti ini

![mc-server-1](./media/mc-server-1.png)

Jika sudah, Install minecraft server

```
java -Xms4G -Xmx8G -jar server.jar nogui
```

Pasti nanti akan error, kalian setujui eula saja dengan perintah ```sudo nano eula.txt```. Ganti false menjadi true.

Sekalian juga ganti pada server.properties menggunakan perintah ```sudo nano server.properties```

Kalian sesuaikan saja dengan kebutuhan, yang utama adalah ganti ```online-mode=false```.

Untuk mengganti MOTD atau Server Description bisa menggunakan [situs ini](https://mctools.org/motd-creator).

Lalu jalankan server kembali dengan perintah 
```
java -Xms4G -Xmx8G -jar server.jar nogui
```

Setelah itu kalian keluar screen saja dengan menekan ```ALT + A + D```. Kalian akan diarahkan ke screen utama.

Untuk sampai step ini sebenarnya server telah aktif dan bisa langsung dimainkan. Namun kita akan menggunakan backup agar file tersebut tidak hilang.

## 5. Membuat Cronjob

Pindah screen untuk membuat cronjob (kita akan membuat screen baru) dengan perintah:

```
screen -S <nama_screen> 
```

Setelah membuat screen baru, silahkan ketikan

```
crontab -e
```

dan masukan perintah:

```
*/10 * * * * cp -rf /home/nap/<namafile> /home/nap/rclone-backup/minecraft/
```

misal:

```
*/10 * * * * cp -rf /home/nap/minecraft/ /home/nap/backup/minecraft/
```

Untuk ```*/10``` itu berarti kita membackup setiap 10 menit sekali.

## 6. Monitoring Cronjob

Kalian bisa membuat screen baru atau langsung di screen utama. Untuk monitoring ini menggunakan script sederhana saja.

Silahkan ketikan dan simpan kedalam file bash. Misal ```cronjob.sh```. Jangan lupa untuk merubah permission dengan perintah ```chmod +x cronjob.sh```. Dan jalankan dengan perintah ```./cronjob.sh```

```
#!/bin/bash

while true
do
    clear
    grep CRON /var/log/syslog
    sleep 3
done
```

![files](./media/file.png)

Mungkin cukup sekian tutorial dari saya, terima kasih. :)

## Selesai