---
title: "Minio存储类别"
date: 2022-08-22T11:37:58+08:00
draft: false
toc: true 
categories: ['minio']
tags: []
---

# minio 存储类别

## 副本机制与纠删码方式对比 

  在分布式数据存储服务中，通常使用的方式有副本机制，与纠删码机制。

**原理对比**

副本技术的原理比较简单，通过副本机制，数据的冗余写来保证数据的可靠性

纠删码类似于raid5、raid6类似。通过引入校验数据块保障数据冗余，从而获得更多的存储空间

EC ( Erasure Coding) 相对 RAID 技术更加灵活，条带由 K 个数据块和 M 个校验数据块组成，而且 K 和 M 是可以调整的。

**空间利用率**

副本机制空间利用率为只有两个副本时，使用率为50%。副本数越多可靠性越高，空间利用率越低。计算比较简单。

纠删码通过调整K 数据块,M 校验块比例来设置可靠性与空间利用率。空间利用率 K / (K+M)。 K 越大空间利用率越高，M越大 可靠性越高。

**可靠性**

容忍节点或磁盘故障数量 副本数 (N-1 ) ,N为节点的数量。纠删码 M，为校验块的比例。损失达到M后变为只读

**纠删码代价**

纠删码通过CPU代价补偿硬盘的空间利用率。CPU的开销会更高

## Minio 中的存储类别

minio采用纠删码的方式保证数据的可靠性。不同于raid5 、minio的纠删码使用在对象级别。

- **[ Erasure Sets](https://docs.min.io/minio/baremetal/concepts/erasure-coding.html#erasure-sets).**

 Erasure set size is automatically calculated based on the number of disks. MinIO  supports unlimited number of disks but each erasure set can be upto 16  disks and a minimum of 2 disks

首先需要了解一个概念， Erasure Sets。 他是一集合，就是前面纠删码中提到的k+M，minio会根据集群中的磁盘总数，将磁盘划分为多个组，每个组就是一个单独的集合（Erasure Sets), 每个集合拥有的磁盘数为4-16个。

参考关系表格见 https://github.com/minio/minio/blob/master/docs/distributed/SIZING.md

[具体计算ES](https://min.io/product/erasure-code-calculator)

- minio中存储类别有两个级别，**STANDARD**、**REDUCED_REDUNDANCY**

STANDARD 的N 值大于 REDUCED_REDUNDANCY 中的N 值。 并且小于 磁盘总数的1/2。

- **设置纠删码中的N 值**

​	通过在启动集群时设置环境变量的方式

```
# 设置环境变量
export MINIO_STORAGE_CLASS_STANDARD=EC:3
export MINIO_STORAGE_CLASS_RRS=EC:2
```

​    通过命令方式 mc admin config get/set

```
# 命令修改EC:N

$mc admin  config get  myplay/ storage_class
storage_class standard=EC:3 rrs=EC:1

$mc admin  config set  myplay/ storage_class standard=EC:2 rrs=EC:1
Successfully applied new settings.
```

```
查看修改 ec 值对总容量的影响
$ mc admin  decommission status myplay/
┌─────┬───────────────────────────────────────────────┬─────────────────────────────────┬────────┐
│ ID  │ Pools                                         │ Capacity                        │ Status │
│ 1st │ http://***** │ 1.1 TiB (used) / 44 TiB (total) │ Active │
└─────┴───────────────────────────────────────────────┴─────────────────────────────────┴────────┘
```

- **默认值**

如果在启动minio集群时没有设置EC值，将采用默认值。

​       通过以下代码与实现测试环境观察。与 [Erasure Coding ](https://docs.min.io/minio/baremetal/concepts/erasure-coding.html#) 文档中的描述 略有不符。

```
 MinIO sets the `REDUCED_REDUNDANCY` parity to `EC:2` by default
```

```
代码位置 https://github.com/minio/minio/blob/master/internal/config/storageclass/storage-class.go

// Default RRS parity is always minimum parity. 默认RRS parity 值 1
	defaultRRSParity = 1

// DefaultParityBlocks returns default parity blocks for 'drive' count 默认 Standy parity 值与dirve 个数的对应关系
func DefaultParityBlocks(drive int) int {
	switch drive {
	case 1:
		return 0
	case 3, 2:
		return 1
	case 4, 5:
		return 2
	case 6, 7:
		return 3
	default:
		return 4
	}
}

// DefaultKVS - default storage class config 配置文件中的默认描述  storage_class standard= rrs=EC:1 # 通过命令 `mc admin config get myplay/ storage_class`
var (
	DefaultKVS = config.KVS{
		config.KV{
			Key:   ClassStandard,
			Value: "",
		},
		config.KV{
			Key:   ClassRRS,
			Value: "EC:1",
		},
	}
)

// Standard constats for config info storage class
const (
	ClassStandard = "standard"
	ClassRRS      = "rrs"

	// Reduced redundancy storage class environment variable
	RRSEnv = "MINIO_STORAGE_CLASS_RRS"
	// Standard storage class environment variable
	StandardEnv = "MINIO_STORAGE_CLASS_STANDARD"

	// Supported storage class scheme is EC
	schemePrefix = "EC"

	// Min parity disks
	minParityDisks = 0

	// Default RRS parity is always minimum parity.
	defaultRRSParity = 1
)
```

## 终端指定类别

   通过 StorageClass设定使用存储类型

```
# minio-client.go
s3Client, err := minio.New("localhost:9000", "YOUR-ACCESSKEYID", "YOUR-SECRETACCESSKEY", true)
if err != nil {
 log.Fatalln(err)
}

object, err := os.Open("my-testfile")
if err != nil {
 log.Fatalln(err)
}
defer object.Close()
objectStat, err := object.Stat()
if err != nil {
 log.Fatalln(err)
}

n, err := s3Client.PutObject("my-bucketname", "my-objectname", object, objectStat.Size(), minio.PutObjectOptions{ContentType: "application/octet-stream", StorageClass: "REDUCED_REDUNDANCY"})
if err != nil {
 log.Fatalln(err)
}
log.Println("Uploaded", "my-objectname", " of size: ", n, "Successfully.")
```

没有指定Storage Class的情况

```
    If STANDARD storage class is set via environment variables or mc admin config get/set commands, and x-amz-storage-class is not present in request metadata, MinIO server will apply STANDARD storage class to the object. This means the data and parity disks will be used as set in STANDARD storage class.
    
    如果集群中设置了STANDARD 而在终端代码中没有使用 storage-class类别，则默认使用STANDARD

    If storage class is not defined before starting MinIO server, and subsequent PutObject metadata field has x-amz-storage-class present with values REDUCED_REDUNDANCY or STANDARD, MinIO server uses default parity values.
    
    如果集群中没有指定 storage class 类别。终端代码中指定的storage-class类别的值将采用集群的默认值
```

参考

https://github.com/minio/minio/blob/9c605ad153fec94df496906bb94ec8733b5620df/docs/erasure/storage-class/README.md



## 思考

**minio 分布式部署方案为什么推荐只是需要4块磁盘，而不是3块**

如果只有3块磁盘，EC最大值为 小于等于3/2 即EC 1， 当其中任何一块磁盘坏掉的情况下，整个集群降为只读模式。

如果有4块磁盘，EC为2，当坏掉其中一块磁盘的时候，整个集群依然具有读写能力。



理解minio中EC 对集群如何做到高可用及集群的扩容将非常有帮助




