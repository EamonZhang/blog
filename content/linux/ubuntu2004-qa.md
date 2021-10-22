---
title: "Ubuntu20.04 装机后"
date: 2021-10-20T15:04:04+08:00
draft: false
toc: false
categories: ["linux"]
tags: [""]
---

## 关闭cloud init

```
systemctl stop cloud-init-local cloud-init cloud-config cloud-final

systemctl disable  cloud-init-local cloud-init cloud-config cloud-final

```
