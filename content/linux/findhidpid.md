---
title: "Linux 找出隐藏进程"
date: 2021-11-03T13:45:36+08:00
draft: false
toc: false
categories: []
tags: []
---

## 原理

在top ps 等命令被改写时，利用Linux一切皆文件。找出被隐藏的进程。

一下为python脚本

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import os

def get_max_pid():
    out = os.popen('cat /proc/sys/kernel/pid_max')
    content = out.readline().strip('\n')
    if content.isdigit():
        return int(content)

def get_ps_proc_list():
    pid_list = []
    out = os.popen('ps -e --no-header')
    lines = out.readlines()
    for line in lines:
        parts = line.split(' ')
        for part in parts:
            if part == '':
                parts.remove(part)

        pid = int(parts[0])
        pid_list.append(pid)

    return pid_list


def get_ps_lwp_list():
    lwp_list = []
    out = os.popen('ps --no-header -eL o lwp')
    lines = out.readlines()
    for line in lines:
        tid = int(line)
        lwp_list.append(tid)

    return lwp_list


def print_badpid_info(pid):
    out = os.popen('ls -l /proc/%d/exe' % pid)
    lines = out.readlines()
    print(lines)


def main():
    max_pid = get_max_pid()
    #print('max pid is %d' % max_pid)
    if max_pid < 0 or max_pid > 50000:
        return

    ps_pid_list = get_ps_proc_list()
    ps_lwp_list = get_ps_lwp_list()

    self_pid = os.getpid()
    for pid in range(2, max_pid):

        #print("handle pid: %d" % pid)

        if pid == self_pid:
            continue

        if pid in ps_pid_list or pid in ps_lwp_list:
            continue

        if not os.path.exists('/proc/' + str(pid)):
            continue

        print("found process not in ps list: %d" % pid)

        print_badpid_info(pid)

if __name__ == '__main__':
    main()
```

[更多参考](https://mp.weixin.qq.com/s?__biz=MzAxODI5ODMwOA==&mid=2666550500&idx=1&sn=9e6cc70e53291b16f7feb5de25882b2b&chksm=80dc904fb7ab19591ccec1bf0bf985f076286545c03a680775a659aaa7e05057c5b8d8e45e11&mpshare=1&scene=23&srcid=0124OSJW32r89rZe9zJf5YKK&sharer_sharetime=1611488968087&sharer_shareid=526a33875b341a963104be96ad05b723#rd)
