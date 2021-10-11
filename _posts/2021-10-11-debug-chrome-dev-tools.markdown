---
layout: post
title:  "Debug Network Request using Chrome Dev Tool"
date:   2021-10-11 21:00:00 +0700
categories: Problem Solving
excerpt: Cara debug network request menggunakan chrome Dev tool.
---
Terkadang perlu kebutuhan yang cepat untuk debug request dari sebuah web. Solusi yang paling gampang dan cepat yang saya tahu menggunakan F12 dan memasukan script di bawah ini, yaitu menggunkan event function *"beforeunload"*.
```
window.addEventListener("beforeunload", function() { debugger; }, false)
```

Ketika kita request akan muncul pop up sebelum page redirect ke halaman lain. Nah kita bisa download XHR dahulu sebelum melanjutkan request. 

#Happy Debug :)
