---
title: "golang调用foundationDB"
date: 2018-04-25T19:37:24+08:00
draft: false

categories: ["foundationdb"]
keywords: ["golang", "go", "db", "nosql", "foundationdb"]
tags: ["golang", "db", "nosql", "foundationdb"]
---

FoundationDB是苹果苹果公司早起闭源又重新开源的一款KV数据库（一开源就是5.x的版本6666），并且支持分布式事务（赞）。
下面为大家介绍一下go是如何使用foundationDB，废话不多说直接上代码！


### 1. 简单的增、删、查（事务）
``` go
package main

import (
	"fmt"
	"log"

	"github.com/apple/foundationdb/bindings/go/src/fdb"
)

func main() {
	// 在使用Api之前需要指定Api的版本，这样做的好处是如果以后修改了api也可以很好的向下兼容
	fdb.MustAPIVersion(510)

	// 打开一个foundationDB数据库，连接道默认的集群配置文件
	// 也可以使用fdb.Open(clusterFile string, dbName []byte)来打开一个指定的集群配置文件
	db := fdb.MustOpenDefault()

	// ok !上述我们已经准备好了一个database, 下面让我们尝试写入一组k-v数据happy一下。
	// 下面是一次单一事务操作，如果没有错误返回，那么数据将会100%插入。
	_, err = db.Transact(func(tr fdb.Transaction) (ret interface{}, e error) {
		tr.Set(fdb.Key("hello"), []byte("world"))
		return
	})
	if err != nil {
		log.Fatalf("Unable to set FDB database value (%v)", err)
	}

	// 既然已经插入了，那么必须要查一次来验证一下吧！
	ret, err := db.Transact(func(tr fdb.Transaction) (ret interface{}, e error) {
		ret = tr.Get(fdb.Key("hello")).MustGet()
		return
	})
	if err != nil {
		log.Fatalf("Unable to read FDB database value (%v)", err)
	}

	v := ret.([]byte)
	fmt.Printf("hello, %s\n", string(v))

	// 增、查都已经操作了，那么删除肯定也是必须的了。
	db.Transact(func(tr fdb.Transaction) (ret interface{}, e error) {
		tr.Clear(fdb.Key("hello"))
		return
	})

	// 验证一把删除
	ret, err = db.Transact(func(tr fdb.Transaction) (ret interface{}, e error) {
		ret = tr.Get(fdb.Key("hello")).MustGet()
		return
	})
	if err != nil {
		log.Fatalf("Unable to read FDB database value (%v)", err)
	}

	v = ret.([]byte)
	fmt.Printf("hello, %s\n", string(v))
}


```
是不是非常简单，还有很多强大的功能等着你去发掘，自己动手丰衣足食。

最后奉上官方golang API文档
[api] (https://godoc.org/github.com/apple/foundationdb/bindings/go/src/fdb)
