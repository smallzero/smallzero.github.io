---
layout: post
title:  "Import Public Certificate Into Java Truststore"
date:   2021-10-11 21:00:00 +0700
categories: Tips
excerpt: Sebenarnya ada 2 cara agar kita bisa akses https request di Java, yaitu dengan menambahkan source code di class TrustManager dengan X509TrustManager, dan menambahkann secara manual kedalam local Java truststore. Untuk kali ini kita coba cara ke-2 yaitu menambahkan public certificate kedalam Java truststore.
---
Sebenarnya ada 2 cara agar kita bisa akses https request menggunakan Java, yaitu: 
1. Dengan menambahkan source code di class TrustManager dengan X509TrustManager. 
2. Menambahkann secara manual kedalam local Java truststore. 

Untuk kali ini kita coba cara ke-2 yaitu menambahkan public certificate kedalam Java truststore.

Cara menampilkan Root Certificate Authority(CA) di dalam Java trutstore yang kita miliki:
```
$JAVA_HOME/bin/keytool -keystore $JAVA_HOME/jre/lib/security/cacerts -list

default password truststore adalah : changeit
```

Cara mendapatkan public certificate bisa menggunakan browser. Contoh di chrome kita bisa copy to file public certificate-nya. Simpan file dengan tipe format .cer

Cara import certificate(.cer) kedalam Java truststore:
```
$JAVA_HOME/bin/keytool -import -alias localImportServer01 -keystore $JAVA_HOME/jre/lib/security/cacerts -file file-import-browser.cer
```
