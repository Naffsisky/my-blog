---
title: "RCLONE Auto Backup File to Cloud Storage"
date: "2023-10-01T22:33:04"
template: "post"
draft: false
slug: "/posts/rclone-auto-backup"
category: "Technology"
tags:
  - "Development"
  - "Cloud Computing"
  - "Rclone"
description: "Tutorial cara membuat auto backup file maupun folder (Linux) ke cloud storage, Google Drive, Microsoft Onedrive, dan lainnya."
socialImage: "./media/RcloneLogo.svg"
---

Cara membuat auto backup file maupun folder ke cloud storage, Google Drive, Microsoft Onedrive, dan lainnya. Dalam hal ini kita menggunakan Linux (Debian 11).

**Perhatikan tulisan yang di bold, agar tidak membingungkan.**

## Install Rclone & Fuse

1. Lakukan install Rclone dan Fuse untuk mounting.

```
sudo apt-get install rclone
sudo apt-get install fuse3 -y
```

## Konfigurasi Rclone

1. Ketikan pada terminal:

```
rclone config
```

2. Kemudian buat baru atau pencet (n).
3. Kasih nama (**drive**).
4. Pilih storage (**cloud storage**). _Ini bebas silahkan masukan yang ingin anda sambungkan, disini kita akan memakai Onedrive_.
5. Kemudian isikan kosong (default). _Isikan kosong atau langsung tekan enter_.

6. Verifikasi token (OTP). _Anda akan diminta memasukan pilihan untuk mengauthentikasi, jika ingin authentikasi di mesin/komputer yang sama maka tekan Y atau menggunakan mesin/komputer yang berbeda untuk verifikasi maka tekan N. Lalu anda akan diminta mengcopy link ke mesin lain (ikuti saja instruksinya)_.

## Jalankan Server Rclone

1. Ketikan pada terminal fusermount untuk mounting.
```
which fusermount3
```

2. Membuat folder baru untuk tempat rclonenya (**folder lokal**). Bisa menggunakan perintah:
```
mkdir <nama folder lokal>
```

Misal: 
```
mkdir fileRclone
```

3. Mounting rclone dengan perintah:
```
rclone mount <nama_drive>: ~/<nama folder lokal>/ --vfs-cache-mode writes
```

Misal:
```
rclone mount rclone: ~/fileRclone/ --vfs-cache-mode writes
```

_Nama drive, yaitu nama awal yang dimasukan saat pertama konfigurasi_.

Saat berhasil dinyalakan, tidak akan muncul respon apa-apa. Diamkan saja.

## Copy File atau Folder

1. Setelah menyalakan server Rclone, buka terminal baru (jangan di berhentikan terminal sebelumnya).

2. Mengcopy file/folder dengan perintah:

```
rclone copy /path/to/local/<nama folder / file yang mau di copy> <nama_drive>:<folder di cloud>
```

Misal:
```
rclone copy /path/to/local/fileRclone rclone:namaFileCloud/
```

Jika bingung path nya ada dimana, bisa menggunakan perintah ```pwd```.

## Selesai