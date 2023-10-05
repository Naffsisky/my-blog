---
title: "Deploy Aplikasi dengan Global External HTTP(S) Load Balancing - Google Cloud Load Balancer"
date: "2023-10-02T10:33:12"
template: "post"
draft: false
slug: "/posts/deploy-load-balancing-gcp"
category: "Technology"
tags:
  - "Development"
  - "Cloud Computing"
  - "Load Balancing"
description: "Deploy aplikasi dengan menggunakan Global External HTTP(S) Load Balancing. Konfigurasi Firewall, Membuat Instance Template dan Instance Group, Konfigurasi Load Balancer (Backend & Frontend), Membuat Health Check, dan Stress Test terhadap Load Balancer."
socialImage: "./media/global-ext-https-lb.svg"
---

## Intro

- [Arsitektur](#arsitektur-load-balancer)
- [Firewall](#konfigurasi-firewall)
- [Instance Template](#membuat-instance-template)
- [Instance Group](#membuat-instance-group)
- [Load Balancer](#konfigurasi-load-balancer)

### Arsitektur Load Balancer
![Arsitektur](media/global-ext-https-lb.svg)

Sekilas tentang perbedaan Load Balancer Google Cloud.

|    **Fitur**   |                        **Load balancing eksternal**                        |                         **Load balancing internal**                        |
|:--------------:|:--------------------------------------------------------------------------:|:--------------------------------------------------------------------------:|
| Sumber traffic |                                  Internet                                  |                          Jaringan VPC Google Cloud                         |
|    Keamanan    |                Membutuhkan SSL/TLS untuk mengamankan traffic               |                          Tidak membutuhkan SSL/TLS                         |
|      Fitur     | Mendukung berbagai jenis load balancing, seperti HTTP, HTTPS, TCP, dan UDP | Mendukung berbagai jenis load balancing, seperti HTTP, HTTPS, TCP, dan UDP |
|  Skalabilitas  |            Dapat menskalakan secara otomatis berdasarkan traffic           |                       Dapat menskalakan secara manual                      |


## Konfigurasi Firewall

Langkah pertama yang perlu kita lakukan adalah menambahkan Firewall rule untuk mengizinkan akses HTTP dan Health Check.
![create-firewall](media/create-firewall.png)

Atau pada GCP yang terbaru, kita sudah di fasilitasi firewall default. Seperti dibawah ini:
![](media/default-firewall.png)

Sebenarnya bisa saja tidak perlu mengkonfigurasi firewall, karena sudah ada dari bawaan GCP yang terbaru.

Tapi pada kasus kali ini kita akan membuat firewall sendiri.

1. Buka halaman Firewall rule dengan cara masuk ke Navigation menu -> VPC network -> Firewall.

2. Di bagian atas halaman, klik tombol Create Firewall Rule.

3. Isikan form seperti berikut untuk membuat HTTP firewall rule:

    - Name: (bebas)
    - Network: default
    - Targets: Specified target tags
    - Target tags: http-server
    - Direction of traffic: Ingress
    - Source filter: IPv4 ranges
    - Source IP ranges: 0.0.0.0/0
    - Protocols and ports: Pilih Specified protocols and ports, centang tcp, isikan 80.

![fw1](media/fw-1.png)

![fw2](media/fw-2.png)

**Note**: 

- Targets: _Specified target tags_. Karena kita akan menargetkan pada VM instance dengan tags (_http-server_).

- Direction of traffic: _Ingress_. Karena kita akan mengakses ke dalam. Ingress dari luar ke dalam, jika Egress dari dalam ke luar.

- Source IP ranges: _0.0.0.0/0_. Kita mengizinkan semua IP address untuk mengakses VM instance kita, jadi IP berapapun bisa mengakses website.

- Protocols and ports: _TCP: 80_. Karena kita akan membuat website dengan HTTP maka aktifkan port 80 untuk HTTP.

4. Setelah itu, Create.

5. Selanjutnya, buat satu firewall rule lagi untuk health check.

6. Di bagian atas halaman, klik tombol Create Firewall Rule.

7. Isikan form seperti berikut untuk membuat HTTP firewall rule:

    - Name: (bebas)
    - Network: default
    - Targets: Specified target tags
    - Target tags: http-server
    - Direction of traffic: Ingress
    - Source filter: IPv4 ranges
    - Source IP ranges: 130.211.0.0/22, 35.191.0.0/16
    - Protocols and ports: Pilih Specified protocols and ports, centang tcp.

8. Setelah itu, Create.

![fw-health-check](media/fw-health-check.png)

**Note**:

Kurang lebih sama seperti sebelumnya. Tetapi yang berbeda adalah:

- Source IP ranges: _130.211.0.0/22, 35.191.0.0/16_. Ini adalah bawaan IP instance dari GCP.

- Protocols and ports: _TCP_. Hanya mencentang TCP karena itu yang mendasari untuk Health Check.

- Sebagai informasi, health check pada load balancer menggunakan IP address range 130.211.0.0/22 dan 35.191.0.0/16 sehingga kita cukup mengizinkan IP range tersebut.

    Baca untuk: [Dokumentasi Health Check](https://cloud.google.com/load-balancing/docs/health-checks)

## Membuat Instance Template

Untuk membuat load balancer, kita perlu menggunakan Managed Instance Group. Untuk membuatnya, kita memerlukan Instance Template untuk Autoscaling. 

Pada kasus ini, kita akan membuat 2 instance groups di 2 region berbeda, yaitu asia-southeast2 dan europe-west1. Ikuti langkah-langkah berikut:

1. Masuk ke Navigation menu > Compute Engine > Instance templates.

2. Di bagian atas halaman, klik Create Instance Template.

3. Isi nama instance template, misalnya asia-southeast2-template. Biarkan konfigurasi lain tetap sesuai default.

![instance-template-1](media/instance-template-1.png)

4. Setting Machine configuration ke yang kecil saja, biar murah meriah.

![instance-template-2](media/instance-template-2.png)

5. Buka panah pada Networking, Disks, Security, Management, Sole-Tenancy. 

6. Pada bagian Networking kemudian pada Network tags, isikan http-server. _Tags inilah yang digunakan pada Firewall rule_. Buka panah Network interfaces. Lalu, pada Subnetwork, pilih subnet asia-southeast2. 

![instance-template-3](media/instance-template-3.png)

![instance-template-4](media/instance-template-4.png)

7. Pada bagian Management -> Automation, isikan kode berikut untuk Startup script:

```
sudo apt-get update
sudo apt-get install apache2 -y
echo '<!doctype html><html><body><h1>Hello from Jakarta!<h1></body></html>' | sudo tee /var/www/html/index.html
```

![instance-template-5](media/instance-template-5.png)

8. Setelah itu, Create.

9. Ulangi langkah seperti tadi dan ubah bagian:

- Nama: europe-west1-template
- Tags: http-server
- Subnet: europe-west1
- Script:

```
sudo apt-get update
sudo apt-get install apache2 -y
echo '<!doctype html><html><body><h1>Hello from Belgium!<h1></body></html>' | sudo tee /var/www/html/index.html
```

10. Setelah itu, Create lagi.

Kita sudah mempunyai 2 Instance Template, 1 untuk region southeast2 (Jakarta) dan 1 untuk region europe-west1 (Belgium).

![instance-template-6](media/instance-template-6.png)

Baca untuk: [Dokumentasi Regions dan Zones](https://cloud.google.com/compute/docs/regions-zones)

Setelah membuat 2 buah instance template, selanjutnya kita akan buat instance group yang bersumber dari instance template tersebut.

## Membuat Instance Group

1. Buka halaman Instance groups yang ada di menu sebelah kiri.

2. Klik Create Instance Group.

![instance-group-1](media/instance-group-1.png)

3. Lalu, isi form seperti berikut:
    - Name: asia-southeast2-group
    - Location: Multiple zones
    - Region: asia-southeast2
    - Instance template: asia-southeast2-template
    - Autoscaling policy: Klik ikon pensil, pilih CPU utilization pada Metric type, dan isi 80 pada Target CPU utilization. Klik Done.
    - Minimum number of instances: 1
    - Maximum number of instances: 5

![instance-group-2](media/instance-group-2.png)

![instance-group-3](media/instance-group-3.png)


_Pada contoh di atas kita membuat sebuah Managed Instance Group bernama asia-southeast2-group di beberapa zone yang berada di region asia-southeast2. Managed Instance Group adalah kumpulan instance yang memiliki fitur untuk autoscaling. Kita bisa menentukan jumlah instance yang digunakan untuk autoscaling dan kapan instance harus melakukan scaling. Kali ini kita menentukan ketika penggunaan CPU mencapai 80% dan bisa melakukan scaling hingga 5 instance_.

**Note**:

Jangan setting Minimum ke 0 jika tidak ingin server anda lambat untuk pertama kali diakses setelah tidak ada orang yang mengaksesnya. Ini juga biasa disebut dengan Cold Start.

4. Oke, setelah selesai, klik tombol Create.

5. Tunggu hingga instance group selesai dibuat. Lalu, masuklah ke halaman VM Instances di menu sebelah kiri. Anda akan melihat instance baru dengan nama namainstancegroup-xxxx.

6. Selanjutnya, buat satu instance group lagi untuk region europe-west1.

7. Masuk lagi ke menu Instance groups, lalu klik Create Instance Group.

8. Kurang lebih sama seperti sebelumnya, tetapi kita mengganti nama dan region:
    - Name: europe-west1-group
    - Location: Multiple zones
    - Region: europe-west1
    - Instance template: europe-west1-template
    - Autoscaling policy: Klik ikon pensil, pilih CPU utilization pada Metric type, dan isi 80 pada Target CPU utilization. Klik Done.
    - Minimum number of instances: 1
    - Maximum number of instances: 5

![instance-group-4](media/instance-group-4.png)

9. Oke, setelah selesai, klik tombol Create.

10. Tunggu hingga instance group selesai dibuat. Lalu, masuklah ke halaman VM Instances di menu sebelah kiri. Anda akan melihat instance baru dengan nama namainstancegroup-xxxx.

![instance-group-5](media/instance-group-5.png)

Kita sudah mempunya 2 instance di VM Instance dan ini akan bertambah dan berkurang secara otomatis tergantung dengan penggunaan CPU tadi yang sudah kita set ke 80%.

## Konfigurasi Load Balancer

Setelah kedua Managed Instance Group siap, selanjutnya kita akan membuat Global External HTTP(S) Load Balancing dan mengatur konfigurasinya.

1. Buka halaman load balancing melalui Navigation Menu -> Network services -> Load balancing, kemudian klik Create Load Balancer.

2. Pilih Start configuration pada HTTP(S) Load Balancing. Saat muncul opsi, biarkan saja sesuai default dengan mengeklik tombol Continue karena kita akan menggunakan external load balancer (dari internet ke VM) dengan cakupan global.

![loadbalancer-1](media/loadbalancer-1.png)

![loadbalancer-2](media/loadbalancer-2.png)

3. Pilih Frontend configuration. Pilih IPv4 pada IP version, pilih Ephemeral pada IP address, dan pilih 80 pada Port. Lalu, klik Done.

    _Untuk nama bebas saja_

4. Tambahkan lagi Frontend configuration dengan klik Add Frontend IP and Port. Pilih IPv6 pada IP version, pilih Auto-allocate pada IP address, dan pilih 80 pada Port. Lalu, klik Done.

![loadbalancer-3](media/loadbalancer-3.png)

5. Pada bagian Backend configuration, klik kolom Backend services & backend buckets, lalu pilih Create a Backend Service.

![loadbalancer-4](media/loadbalancer-4.png)

6. Pada dialog yang muncul, berikan nama untuk backend service, sebagai contoh lb-http-backend.

![loadbalancer-5](media/loadbalancer-5.png)

7. Tambahkan backend baru menggunakan instance group yang tadi Anda buat. Pilih asia-southeast2-group pada kolom Instance group dan isikan 80 pada kolom Port numbers.
Pada Balancing mode, kita bisa mengatur bagaimana pembagian beban untuk backend instance tersebut. Pada backend asia-southeast2 ini, kita akan menggunakan mode Rate dengan Maximum RPS (request per second) sebesar 50 per instance. Artinya, load akan tetap dijalankan di backend ini selama traffic masih di bawah 50 RPS. Klik Done.

![loadbalancer-6](media/loadbalancer-6.png)

8. Lalu, klik Add Backend untuk membuat backend baru. Pilih europe-west1-group untuk Instance group dan isi 80 untuk Port numbers.

9. Biarkan konfigurasi Balancing mode sesuai defaut. Klik Done.

10. Selanjutnya, pada field Health check, pilih Create a Health Check. Health check ini berperan untuk memantau kondisi ketersediaan dari instance.

11. Berikan nama untuk health check dengan http-health-check, lalu ubah Protocol menjadi HTTP. Klik Save.

12. Kemudian, klik Create untuk menyimpan konfigurasi dari backend service. Lalu, pilih OK.

![loadbalancer-7](media/loadbalancer-7.png)

![loadbalancer-8](media/loadbalancer-8.png)

13. Masuk ke bagian Review and finalize untuk mengulas kembali konfigurasi dari load balancer yang akan kita buat. Jika sudah benar, klik tombol Create.

![loadbalancer-9](media/loadbalancer-9.png)

14. Tunggu beberapa menit. GCP sedang membuat load balancer dan health check sedang melakukan pengecekan terhadap instance.

15. Selang beberapa menit, klik pada nama load balancer, lalu salin IP:Port untuk IPv4 di bagian Frontend, kemudian jalankan pada tab browser Anda.

_Anda akan diarahkan ke instance terdekat dari lokasi Anda. Anda juga bisa coba akses IP tersebut menggunakan VPN agar menerima respons dari Europe_.

![loadbalancer-10](media/loadbalancer-10.png)

Contoh Proxy EU :

![loadbalancer-11](media/loadbalancer-11.png)

## Stress Test terhadap Load Balancer

Setelah berhasil membuat load balancer untuk mengakses instance yang berada di lokasi terdekat dari pelanggan, langkah selanjutnya kita akan melakukan stress test guna menguji pembagian beban pada load balancer.

1. Install siege

```
sudo apt-get install siege
```

2. Jalankan perintah berikut untuk memulai stress test:

```
siege -c 250 http://load_balancer_ip
```

3. Buka halaman Load balancing, lalu klik pada nama load balancer Anda. Buka tab Monitoring, Anda akan melihat alur dari traffic yang dikirim dari Siege. Karena kita telah mengatur backend di asia-southeast2 hanya bisa menerima 50 RPS, traffic pun akan dialihkan ke europe-west1 meskipun VM tempat kita mengakses berada di Asia.


Setelah melakukan stress test, walaupun kita sedang berada di dekat Jakarta tetapi kita di pindahkan ke server Belgium. karena traffic di Jakarta sedang full.

_Gambar kiri yang baru, dan Gambar kanan yang lama_.

![loadbalancer-12](media/loadbalancer-12.png)

## Selesai