---
title: "Deploy BMKG Rest API Menggunakan Proxy Server Nginx"
date: "2023-11-07T16:24:05"
template: "post"
draft: false
slug: "/posts/deploy-simple-cloud-run"
category: "Technology"
tags:
  - "Cloud Run"
  - "Web Development"
  - "Google Cloud Platform"
description: "Membuat Backend service Rest API menggunakan Node.js dan framework Express.js. Menggunakan Nginx sebagai Proxy server."
socialImage: "./media/nginx.png"
---

## Open API 

Bertujuan untuk membuka API google ke cloud run.

```
gcloud services enable run.googleapis.com
```

## Konfigurasi Aplikasi

### Membuat Aplikasi Sederhana

Buat file dengan nama package.json :

```json
{
  "name": "helloworld",
  "description": "Simple hello world sample in Node",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "author": "Google LLC",
  "license": "Apache-2.0",
  "dependencies": {
    "express": "^4.17.1"
  }
}
```

Buat file dengan nama index.js :

```js
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  const name = process.env.NAME || 'World';
  res.send(`Hello ${name}!`);
});

app.listen(port, () => {
  console.log(`helloworld: listening on port ${port}`);
});
```

Server akan berjalan di ```Port 8080``` memakai Express JS.

Buat file dengan nama Dockerfile :

```dockerfile
FROM node:12-slim

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install --only=production

COPY . ./

CMD [ "npm", "start" ]
```

**Keterangan :**

```FROM node:12-slim``` : Menggunakan official lightweight Node.js 12 image

```WORKDIR /usr/src/app``` : Membuat dan mengganti ke folder applikasi

```COPY package*.json ./``` : Memindahkan package.json ke container

```RUN npm install --only=production``` : Perintah untuk Insallasi aplikasi

```COPY . ./``` : Memindahkan semua file di local ke container

```CMD [ "npm", "start" ]``` : Perintah untuk menjalankan aplikasi

## Deploy Container Image

```
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

Kita membuild container image dan akan memberi tag atau nama dengan nama helloworld.

Lakukan pengecekan Artifact Registry :
```
gcloud container images list
```

Lakukan pengecekan autentikasi dan konfigurasi docker :
```
gcloud auth configure-docker
```

Lakukan testing di lokal untuk memastikan container dan container image sudah benar

```
docker run -d -p 8080:8080 gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

```Membuka di port 8080:8080```

## Deploy to Cloud Run

```
gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld --allow-unauthenticated --region=$LOCATION
```

Kalian bisa membuat location terlebih dahulu atau langsung mengetikannya nanti pada saat akan melakukan deployment. ```LOCATION="Region"```


### Menghapus Image dan Container

```
gcloud container images delete gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld
```

```
gcloud run services delete helloworld --region=$LOCATION
```

## Selesai