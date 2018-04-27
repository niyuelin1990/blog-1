---
title: "Golang使用FoundationDB计数器"
date: 2018-04-27T13:49:54+08:00
draft: false
categories: ["foundationdb"]
keywords: ["db", "nosql", "foundationdb"]
tags: ["db", "nosql", "foundationdb"]
---

工作中利用foundationDB实现了一套自己的IM服务，有针对于消息的统计计数，
官方对golang使用计数器的文档对负数的处理说的含糊不清，废话不多说直接上代码，坑都写在里面了，请欣赏：

``` go

package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"time"

	"github.com/apple/foundationdb/bindings/go/src/fdb"
)

func main() {
	fdb.MustAPIVersion(510)
	db := fdb.MustOpenDefault()
	start := time.Now()
	_, err := db.Transact(func(tr fdb.Transaction) (ret interface{}, e error) {
		for index := 0; index < 20; index++ {
			value := make([]byte, binary.MaxVarintLen64)
			var v int64 = -1
			// golang 大小端接口都是无符号的，所以要进行强转(被坑了很久)
			binary.LittleEndian.PutUint64(value, uint64(v))
			// Add参数的value要求必须是小端存储
			tr.Add(fdb.Key("hello"), value)
		}
		return
	})
	if err != nil {
		log.Fatalf("Unable to set FDB database value (%v)", err)
	}

	log.Println("入库耗时", time.Now().Sub(start).Seconds())

	// 查询
	ret, err := db.Transact(func(tr fdb.Transaction) (ret interface{}, e error) {
		ret = tr.Get(fdb.Key("hello")).MustGet()
		return
	})
	if err != nil {
		log.Fatalf("Unable to read FDB database value (%v)", err)
	}

	v := ret.([]byte)
	// 记得小端和符号问题
	fmt.Println("hello, ", int64(binary.LittleEndian.Uint64(v)))
}

```

OK！ See you！
