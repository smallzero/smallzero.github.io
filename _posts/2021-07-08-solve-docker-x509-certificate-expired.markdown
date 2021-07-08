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

Solusi dari error di atas adalah pertma kita harus tahu nama docker-machine kita apa, dengan perintah sebagai berikut:
```
$ docker-machine.exe ls
```
Contoh punya saya nama docker-machine adalah "default"
```
$ docker-machine.exe ls
NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
default   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.06.0-ce
```

Lalu perbarui certificate dengan cara di bawah ini:
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

Dan hasilnya sudah tidak ada error lagi
```
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                               NAMES
368d9baa26c4        adminer                "entrypoint.sh docke…"   5 weeks ago         Up 1 second         0.0.0.0:8036->8080/tcp              docker_adminer_1
ddc8fb5f3ea7        docker_mysql-service   "docker-entrypoint.s…"   5 weeks ago         Up 1 second         0.0.0.0:3306->3306/tcp, 33060/tcp   docker_mysql-service_1
```


Good luck :)