---
title: "Linux下安装FoundationDB"
date: 2018-04-23T21:20:03+08:00
draft: false
categories: ["db", "foundationdb"]
keywords: ["db", "nosql", "foundationdb"]
tags: ["db", "nosql", "foundationdb"]
---


<!-- # Linux下安装FoundationDB -->

<a href="https://apple.github.io/foundationdb/downloads.html" target="_blank"> [下载地址]

`系统要求: `

- 64位操作系统
- RHEL/CentOS 6.x或者7.x
- Ubuntu 12.04或更高版本
- 不支持的Linux发行版包含：
内核版本介于2.6.33和3.0.x之间（含）或3.7或更高,适用于.deb或.rpm包 或者，macOS 10.7或更高版本


>      警告： macOS下FoundationDB仅适用于本地访问。


### Ubuntu
>      google@localhost$ sudo dpkg -i foundationdb-clients_5.1.5-1_amd64.deb foundationdb-server_5.1.5-1_amd64.deb

### RHEL/CentOS 6
>      google@localhost$ sudo rpm -Uvh foundationdb-clients-5.1.5-1.el7.x86_64.rpm foundationdb-server-5.1.5-1.el7.x86_64.rpm

### RHEL/CentOS 7
>      google@localhost$ sudo rpm -Uvh foundationdb-clients-5.1.5-1.el7.x86_64.rpm  foundationdb-server-5.1.5-1.el7.x86_64.rpm

### 测试

``` sql
google@localhost$ fdbcli
Using cluster file `/etc/foundationdb/fdb.cluster'.

The database is available.

Welcome to the fdbcli. For help, type `help'.
fdb> status

Configuration:
  Redundancy mode        - single
  Storage engine         - memory
  Coordinators           - 1

Cluster:
  FoundationDB processes - 1
  Machines               - 1
  Memory availability    - 4.1 GB per process on machine with least available
  Fault Tolerance        - 0 machines
  Server time            - Thu Mar 15 14:41:34 2018

Data:
  Replication health     - Healthy
  Moving data            - 0.000 GB
  Sum of key-value sizes - 8 MB
  Disk space used        - 103 MB

Operating space:
  Storage server         - 1.0 GB free on most full server
  Transaction log        - 1.0 GB free on most full server

Workload:
  Read rate              - 2 Hz
  Write rate             - 0 Hz
  Transactions started   - 2 Hz
  Transactions committed - 0 Hz
  Conflict rate          - 0 Hz

Backup and DR:
  Running backups        - 0
  Running DRs            - 0

Client time: Thu Mar 15 14:41:34 2018
```
如果显示上述代表成功。
