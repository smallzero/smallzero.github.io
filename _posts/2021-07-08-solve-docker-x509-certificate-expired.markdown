---
layout: post
title:  "Solve Docker x509: certificate has expired or is not yet valid"
date:   2021-07-08 21:00:00 +0700
categories: Problem Solving
excerpt: Sudah lama tidak membuka docker desktop versi pertama di windows dan menemukan error x509 certificate has expired or is not yet valid
---
Sudah lama tidak membuka docker desktop versi pertama di windows dan menemukan error x509 certificate has expired or is not yet valid
```
$ docker ps -a
error during connect: Get https://192.168.99.100:2376/v1.37/containers/json?all=1: x509: certificate has expired or is not yet valid
```

Saat menjalankan "docker-machine start" normal tidak ada masalah.

Solusi dari error di atas adalah sebagai berikut :
```
$ docker-machine.exe regenerate-certs --client-certs -f default
```

Output
```
Regenerating TLS certificates
Regenerating local certificates
CA certificate is outdated and needs to be regenerated
Creating CA: C:\Users\XXXX\.docker\machine\certs\ca.pem
Client certificate is outdated and needs to be regenerated
Creating client certificate: C:\Users\XXXX\.docker\machine\certs\cert.pem
Waiting for SSH to be available...
Detecting the provisioner...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
```