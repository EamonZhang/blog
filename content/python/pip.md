---
title: "python pip"
date: 2023-04-26T09:03:03+08:00
draft: false
toc: true
categories: ['python']
tags: []
---

## 介绍
pip python第三方库管理工具,从 Python 3.4 开始，pip 已经内置在 Python。

## 升级
```
pip install --upgrade pip 或 pip install -U pip
```

## 安装第三方库

```
pip install package_name==包版本
```

## 批量安装
```
pip install -r requirements.txt
```

```
cat requirements.txt
tensorflow==2.3.1
uvicorn==0.12.2
fastapi==0.63.0
```

## 卸载
```
pip uninstall package_name
```

## 升级
```
pip install -U package_name
```

## 冻结当前环境
有时您想输出当前环境中所有已安装的包，或生成一个需求文件
```
pip freeze
```

## 查看需要升级的库
```
pip list -o
```

## 检查兼容性问题
```
pip check
```
