---
layout: post
title:  编译Python源码
date:   2020-07-07 18:35:08 +0800
tags:   Python CPython
---
```sh
curl "https://www.python.org/ftp/python/3.8.3/Python-3.8.3.tgz" -o Python-3.8.3.tgz
tar -zxvf Python-3.8.3.tgz
cd Python-3.8.3
./configure --prefix=$PWD/build
make
make install
$PWD/build/bin/python3 -V
```
