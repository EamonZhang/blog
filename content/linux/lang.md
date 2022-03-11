---
title: "Linux 字符集"
date: 2022-03-11T15:25:50+08:00
draft: false
toc: false
categories: ["linux"]
tags: []
---

## Centos 字符集

```
yum -y install glibc-locale-source glibc-langpack-en

localedef -f UTF-8 -i en_US en_US.UTF-8

```
