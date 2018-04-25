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

``` go
package main

import (
	"fmt"
	"log"

	"github.com/apple/foundationdb/bindings/go/src/fdb"
	"github.com/apple/foundationdb/bindings/go/src/fdb/directory"
	"github.com/apple/foundationdb/bindings/go/src/fdb/tuple"
)

func main() {
	// 设置版本
	fdb.MustAPIVersion(510)
	// 打开配置文件
	db := fdb.MustOpen("/usr/local/etc/foundationdb/fdb.cluster", []byte("DB"))

	// 默认配置
	// db := fdb.MustOpenDefault()
	nmqDir, err := directory.CreateOrOpen(db, []string{"Ali"}, nil)
	if err != nil {
		log.Fatal(err)
	}
	// 创建子空间
	app := nmqDir.Sub("Taobao")
	subtopic := "Account"
	image := "images"
	// 存储消息
	_, err = db.Transact(func(tr fdb.Transaction) (ret interface{}, err error) {
		for i := 1; i < 70; i++ {
			// 创建Key，可以根据业务类型增加Key的数量进行分类
			// 如:  公司,产品,会员
			//      公司,产品,会员,头像
			//      公司,产品,会员,简介

			// foundationdb 是用key来排序的，有排序需求的同学必须要对key进行特殊处理
			SCKey := app.Pack(tuple.Tuple{subtopic, image, fmt.Sprintf("%02d", i), fmt.Sprintf("%02d", i-1)})
			log.Println(string(SCKey))
			tr.Set(SCKey, []byte("imag_id/url"))
		}
		return
	})

	// 范围查询消息
	var keys []string
	_, err = db.Transact(func(tr fdb.Transaction) (ret interface{}, err error) {
		pr, _ := fdb.PrefixRange([]byte("Taobao"))
		// 指定Key的开始和结尾
		pr.Begin = app.Pack(tuple.Tuple{subtopic, image, fmt.Sprintf("%02d", 0)})
		pr.End = app.Pack(tuple.Tuple{subtopic, image, fmt.Sprintf("%02d", 69)})

		// 也可以调用  tr.Get(key) 查询单条
		// 指定limit
		ir := tr.GetRange(pr, fdb.RangeOptions{Limit: 10}).Iterator()
		count := 0
		for ir.Advance() {
			// 这里一定要注意将MustGet()结果拿到，然后用这个结果去获得key和value
			v := ir.MustGet()
			count++
			log.Println(string(v.Key), string(v.Value), count)
			keys = append(keys, string(v.Key))
		}
		return
	})

	fmt.Println(keys)

	// 删除消息
	_, err = db.Transact(func(tr fdb.Transaction) (ret interface{}, err error) {
		tr.ClearRange(app.Sub(subtopic))

		// 也可以调用 tr.Clear(key)删除单条

		return
	})

	// 验证删除
	_, err = db.Transact(func(tr fdb.Transaction) (ret interface{}, err error) {
		pr, _ := fdb.PrefixRange([]byte("Taobao"))
		// 指定Key的开始和结尾
		pr.Begin = app.Pack(tuple.Tuple{subtopic, image, fmt.Sprintf("%02d", 0)})
		pr.End = app.Pack(tuple.Tuple{subtopic, image, fmt.Sprintf("%02d", 69)})

		// 指定limit
		ir := tr.GetRange(pr, fdb.RangeOptions{Limit: 10}).Iterator()
		count := 0
		for ir.Advance() {
			// 这里一定要注意将MustGet()结果拿到，然后用这个结果去获得key和value
			v := ir.MustGet()
			count++
			log.Println(string(v.Key), string(v.Value), count)
			keys = append(keys, string(v.Key))
		}
		return
	})
}


```
是不是非常简单，还有很多强大的功能等着你去发掘，自己动手丰衣足食。

最后奉上官方golang API文档
[api] (https://godoc.org/github.com/apple/foundationdb/bindings/go/src/fdb)
