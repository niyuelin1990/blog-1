---
title: "基于foundationDB的课程管理例子"
date: 2018-04-26T13:15:19+08:00
draft: false
bigimg: [{src: "https://res.cloudinary.com/shaocongcong/image/upload/v1524187685/blog/bigimg/internet.jpg", desc: "internet"}]
categories: ["foundationdb"]
keywords: ["golang", "go", "db", "nosql", "foundationdb"]
tags: ["golang", "db", "nosql", "foundationdb"]
---



### 课程管理
课程管理系统主要是帮组学生和管理员安排课程，双手奉上[官方文档](https://apple.github.io/foundationdb/class-scheduling-go.html#class-sched-go-appendix)
，官方的意思是先阅读一下源码，我感觉不必要了，因为我会帮你总结一下知识点的(^_^)。


### 需求
我们需要让用户列出可用的课程并且跟踪哪些学生报哪些课程。 下面是我们需要实现的功能的第一步，定义三个接口：

``` go 
    availableClasses()       // 获取课程列表
    signup(studentID, class) // 为学生报一门课程
    drop(studentID, class)   // 学生停止学习一门课程
```

### 数据模型
首先设计一个数据模型。 数据模型一种在FoundationDB中使用键和值存储数据的方法。 我们需要设计两种类型的数据：

``` go 
    // ("attends", student, class) = ""        // 哪些学生参加哪些课程列表
    // ("class", class_name) = seatsAvailable  // 课程列表
```

### 科普时刻
前面的文章有提到过FoundationDB的核心提供了一个简单的数据模型和强大的事务处理，这种组合允许构建更丰富的数据模型和库
    
    核心数据模型：
        FoundationDB的核心数据模型是一个有序的键值存储。也可以叫做有序关联数组，映射或字典，是一个通用数据结构，由一组键值对组成，其中所有键都是唯一的。从这个简单模型开始，应用程序可以通过将其元素映射到单个键和值来创建更高级别的数据模型。
        在FoundationDB中，键和值都是简单的字节字符串。除了存储和检索之外，数据库关心值的内容。键值在底层字节上来做排序，即字节顺序在底层字节上，其中键按顺序按每个字节排序。例如：
        - “0”在“1”之前排序
        - 'apple'在'banana'之前排序
        - 'apple'在'apple123'之前排序
        - 以'mytable \'开头的键排序在一起（例如'mytable \ row1'，'mytable \ row2'，...）
        密钥的排序可能严重影响结果。应用程序应该将键结构化，以产生一个允许使用范围读取进行高效数据检索的顺序。（The ordering of keys is especially relevant for range operations. An application should structure keys to produce an ordering that allows efficient data retrieval with range reads.）这句话我翻译的不是太好，直接google了，见谅。**/(ㄒoㄒ)/~~**

    数据类型转码：
        由于FoundationDB中的键和值都是是字节字符串，所以在存入数据库之前需要将数据进行序列化（例如，整数，浮点数，数组）。 对于数值，序列化的主要关注点仅仅是CPU和空间效率。 秘钥比较特殊：需要保留它们编码的数据类型的顺序通常很重要。例如：

    整型
        标准元组层提供了保持顺序，有符号，可变长度的编码。
        对于正整数，大端固定长度编码是保序的。
        对于有符号整数，具有最高有效位（符号）位的大端固定长度二进制补码编码是保序的。

    Unicode字符串
        对于unicode代码点按字典顺序排序的unicode字符串，请使用UTF-8编码。 （元组层使用。）
        对于通过特定归类排序的unicode字符串（例如，对特定不区分大小写的排序），需要进行字符串归类转换，然后应用UTF-8编码。在大多数环境和编程语言中，国际化或本地库进行字符串整理转换，例如C，C ++，Python，Ruby，Java和ICU库等。通常输出是unicode字符串，需要进一步用诸如UTF-8之类的代码点排序编码进行编码以获得字节串。
    
    浮点数字
        元组层为基于IEEE big-endian编码的单精度浮点数和双精度浮点数提供了保序，有符号，固定长度的编码，并进行了一些修改以使其正确排序。在这种表示中，-0和+0不相等，并且负NaN值将在所有非NaN值和正NaN值将排序在所有非NaN值之前排序。否则，该表示与数学排序一致。

    复合类型
        应用程序的数据通常使用复合类型来表示，例如具有多个字段的结构或记录。 对于应用程序使用组合键来存储这些数据非常有用。 在FoundationDB中，组合键可以方便直观的映射到单个键进行存储的元组。说白了就是使用组合key来存放value。

好了，话题切回课程管理的例子，我们还需要seatAvailable来记录可用座位的数量。

目录和子空间
    在使用这些功能之前我们要引入一些头文件(写c的人都这么喊 /(ㄒoㄒ)/)

``` go
import (
    "github.com/apple/foundationdb/bindings/go/src/fdb"
    "github.com/apple/foundationdb/bindings/go/src/fdb/directory"
    "github.com/apple/foundationdb/bindings/go/src/fdb/subspace"
    "github.com/apple/foundationdb/bindings/go/src/fdb/tuple"
)
```

创建一个目录
``` go
schedulingDir, err := directory.CreateOrOpen(db, []string{"scheduling"}, nil)
if e != nil {
  log.Fatal(e)
}
```

CreateOrOpen()返回一个子空间，scheduling子空间用来存储我们的应用数据。 每个子空间在定义key时都应该使用固定的前缀。 前缀是元组的第一个元素。 所以我们用"attends"和"class"作为我们的前缀，下面我们来创建这两个子空间。
``` go
courseSS = schedulingDir.Sub("class")
attendSS = schedulingDir.Sub("attends")
```
子空间有专门生成key的Pack()函数。存放数据的时候我们可以这使用：
``` go
attendSS.Pack(tuple.Tuple{studentID, class})
courseSS.Pack(tuple.Tuple{class})
```

### 事务
依赖FoundationDB强大的事务功能保证来修改的正确性，来瞅瞅FoundationDB Go API怎么来使用事务功能。**Transact()**函数。 如，为学生报一门课程：
``` go

// signup  为学生报一门课程
func signup(t fdb.Transactor, studentID, class string) (err error) {
  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    // 向attendSS子空间插入数据
    tr.Set(attendSS.Pack(tuple.Tuple{studentID, class}), []byte{})
    return
  })
  return
}
```
Transactor()函数会自动创建一个事务，在内部实现一个循环来确保事务最终提交。如果不使用Transactor()函数又要保证最终提交可以参考下面的例子:
``` go
func signup(db fdb.Database, studentID, class string) (err error) {
  tr, err := d.CreateTransaction()
  if err != nil {
    return
  }

  wrapped := func() {
    defer func() {
      if r := recover(); r != nil {
        e, ok := r.(Error)
        if ok {
          err = e
        } else {
          panic(r)
        }
      }
    }()

    tr.Set(attendSS.Pack(tuple.Tuple{studentID, class}), []byte{})

    err = tr.Commit().Get()
  }

  for {
    wrapped()

    if err == nil {
      return
    }

    fe, ok := err.(Error)
    if ok {
      err = tr.OnError(fe).Get()
    }

    if err != nil {
      return
    }
  }
}
```
丑陋！如果要提交多个事务就是一种灾难，so感谢苹果大大的Transactor()。

### 创建简单的示例数据
``` go
var levels = []string{"intro", "for dummies", "remedial", "101", "201", "301", "mastery", "lab", "seminar"}
var types = []string{"chem", "bio", "cs", "geometry", "calc", "alg", "film", "music", "art", "dance"}
var times = []string{"2:00", "3:00", "4:00", "5:00", "6:00", "7:00", "8:00", "9:00", "10:00", "11:00",
                     "12:00", "13:00", "14:00", "15:00", "16:00", "17:00", "18:00", "19:00"}

classes := make([]string, len(levels) * len(types) * len(times))

for i := range levels {
  for j := range types {
    for k := range times {
      classes[i*len(types)*len(times)+j*len(times)+k] = fmt.Sprintf("%s %s %s", levels[i], types[j], times[k])
    }
  }
}
``` 

### 初始化数据库

课程
``` go
_, err = db.Transact(func (tr fdb.Transaction) (interface{}, error) {
  tr.ClearRange(schedulingDir)

  for i := range classes {
    // 这里为每个课程设置了100个位子！！！
    tr.Set(courseSS.Pack(tuple.Tuple{classes[i]}), []byte(strconv.FormatInt(100, 10)))
  }

  return nil, nil
})
``` 

### 获取可报的课程
学生在报名之前都需要从数据库中查询一次，我们可以使用GetRange()来进行范围查找
``` go
func availableClasses(t fdb.Transactor) (ac []string, err error) {
  r, err := t.ReadTransact(func (rtr fdb.ReadTransaction) (interface{}, error) {
    var classes []string
    // 返回courseSS子空间的所有课程(如果不对fdb.RangeOptions{}进行设置的话默认全部)
    ri := rtr.GetRange(courseSS, fdb.RangeOptions{}).Iterator()
    for ri.Advance() {
      kv := ri.MustGet()
      // 解码
      t, err := courseSS.Unpack(kv.Key)
      if err != nil {
        return nil, err
      }
      classes = append(classes, t[0].(string))
    }
    return classes, nil
  })
  if err == nil {
    ac = r.([]string)
  }
  return
}
``` 


### 报名一门课程
``` go
func signup(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})

  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    tr.Set(SCKey, []byte{})
    return
  })
  return
}
```

### 停止学习一门课程
``` go
func drop(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})
  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    // 删除
    tr.Clear(SCKey)
    return
  })
  return
}
```
### OK？问题来了，Seats are limited!座位有限!咋办兄弟!!! 坐在地上吗！！！
所有我们需要对课程的报名人数进行限制！！
获取课程修改
``` go
func availableClasses(t fdb.Transactor) (ac []string, err error) {
  r, err := t.ReadTransact(func (rtr fdb.ReadTransaction) (interface{}, error) {
    var classes []string
    ri := rtr.GetRange(courseSS, fdb.RangeOptions{}).Iterator()
    for ri.Advance() {
      kv := ri.MustGet()
      // 获取一下空闲座位数   
      v, err := strconv.ParseInt(string(kv.Value), 10, 64)
      if err != nil {
        return nil, err
      }
      // 如果有空闲座位就上报上去
      if v > 0 {
        t, err := courseSS.Unpack(kv.Key)
        if err != nil {
          return nil, err
        }
        classes = append(classes, t[0].(string))
      }
    }
    return classes, nil
  })
  if err == nil {
    ac = r.([]string)
  }
  return
}
```

报名课程修改
``` go
func signup(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})
  classKey := courseSS.Pack(tuple.Tuple{class})

  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    // 检查是否报过名
    if tr.Get(SCKey).MustGet() != nil {
      return // already signed up
    }

    // 查看是否有座位
    seats, err := strconv.ParseInt(string(tr.Get(classKey).MustGet()), 10, 64)
    if err != nil {
      return
    }
    if seats == 0 {
      err = errors.New("no remaining seats")
      return
    }
    // 空闲的座位要减一个
    tr.Set(classKey, []byte(strconv.FormatInt(seats - 1, 10)))
    tr.Set(SCKey, []byte{})

    return
  })
  return
}
```
原文中一堆介绍并发性、一致性和幂等等问题，我们都在事务中解决了，有兴趣可以自己看看。

### 弃课修改^_^
``` go
func drop(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})
  classKey := courseSS.Pack(tuple.Tuple{class})
  // 没报名直接返回
  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    if tr.Get(SCKey).MustGet() == nil {
      return // not taking this class
    }
    
    // 获取座位
    seats, err := strconv.ParseInt(string(tr.Get(classKey).MustGet()), 10, 64)
    if err != nil {
      return
    }
    
    // 空闲座位++
    tr.Set(classKey, []byte(strconv.FormatInt(seats + 1, 10)))
    // 将学生赶走。。。。
    tr.Clear(SCKey)

    return
  })
  return
}
``` 

### 限制学生报课数
``` go
func signup(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})
  classKey := courseSS.Pack(tuple.Tuple{class})

  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    if tr.Get(SCKey).MustGet() != nil {
      return // already signed up
    }

    seats, err := strconv.ParseInt(string(tr.Get(classKey).MustGet()), 10, 64)
    if err != nil {
      return
    }
    if seats == 0 {
      err = errors.New("no remaining seats")
      return
    }
    // 获取已报课程数
    classes := tr.GetRange(attendSS.Sub(studentID), fdb.RangeOptions{Mode: fdb.StreamingModeWantAll}).GetSliceOrPanic()
    if len(classes) == 5 {
      err = errors.New("too many classes")
      return
    }

    tr.Set(classKey, []byte(strconv.FormatInt(seats - 1, 10)))
    tr.Set(SCKey, []byte{})

    return
  })
  return
}

```

### 换课。。。(感叹学生事真多。。。)

这里完全是通过事务来做到一致性 nice!!!
``` go
func swap(t fdb.Transactor, studentID, oldClass, newClass string) (err error) {
  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    err = drop(tr, studentID, oldClass)
    if err != nil {
      return
    }
    err = signup(tr, studentID, newClass)
    return
  })
  return
}
```

搞到这里基本结束了，我想大家也基本对foundationDB的事务功能用法有一个大概的了解了吧！总之自己动手丰衣足食!加油吧筒子们！！！
让我们把所有源码全贴上来吧(注释可参考上面)！
``` go
package main

import (
  "github.com/apple/foundationdb/bindings/go/src/fdb"
  "github.com/apple/foundationdb/bindings/go/src/fdb/directory"
  "github.com/apple/foundationdb/bindings/go/src/fdb/subspace"
  "github.com/apple/foundationdb/bindings/go/src/fdb/tuple"

  "fmt"
  "log"
  "strconv"
  "errors"
  "sync"
  "math/rand"
)

var courseSS subspace.Subspace
var attendSS subspace.Subspace

var classes []string

func availableClasses(t fdb.Transactor) (ac []string, err error) {
  r, err := t.ReadTransact(func (rtr fdb.ReadTransaction) (interface{}, error) {
    var classes []string
    ri := rtr.GetRange(courseSS, fdb.RangeOptions{}).Iterator()
    for ri.Advance() {
      kv := ri.MustGet()
      v, err := strconv.ParseInt(string(kv.Value), 10, 64)
      if err != nil {
        return nil, err
      }
      if v > 0 {
        t, err := courseSS.Unpack(kv.Key)
        if err != nil {
          return nil, err
        }
        classes = append(classes, t[0].(string))
      }
    }
    return classes, nil
  })
  if err == nil {
    ac = r.([]string)
  }
  return
}

func signup(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})
  classKey := courseSS.Pack(tuple.Tuple{class})

  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    if tr.Get(SCKey).MustGet() != nil {
      return // already signed up
    }

    seats, err := strconv.ParseInt(string(tr.Get(classKey).MustGet()), 10, 64)
    if err != nil {
      return
    }
    if seats == 0 {
      err = errors.New("no remaining seats")
      return
    }

    classes := tr.GetRange(attendSS.Sub(studentID), fdb.RangeOptions{Mode: fdb.StreamingModeWantAll}).GetSliceOrPanic()
    if len(classes) == 5 {
      err = errors.New("too many classes")
      return
    }

    tr.Set(classKey, []byte(strconv.FormatInt(seats - 1, 10)))
    tr.Set(SCKey, []byte{})

    return
  })
  return
}

func drop(t fdb.Transactor, studentID, class string) (err error) {
  SCKey := attendSS.Pack(tuple.Tuple{studentID, class})
  classKey := courseSS.Pack(tuple.Tuple{class})

  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    if tr.Get(SCKey).MustGet() == nil {
      return // not taking this class
    }

    seats, err := strconv.ParseInt(string(tr.Get(classKey).MustGet()), 10, 64)
    if err != nil {
      return
    }

    tr.Set(classKey, []byte(strconv.FormatInt(seats + 1, 10)))
    tr.Clear(SCKey)

    return
  })
  return
}

func swap(t fdb.Transactor, studentID, oldClass, newClass string) (err error) {
  _, err = t.Transact(func (tr fdb.Transaction) (ret interface{}, err error) {
    err = drop(tr, studentID, oldClass)
    if err != nil {
      return
    }
    err = signup(tr, studentID, newClass)
    return
  })
  return
}

func main() {
  fdb.MustAPIVersion(510)
  db := fdb.MustOpenDefault()

  schedulingDir, err := directory.CreateOrOpen(db, []string{"scheduling"}, nil)
  if err != nil {
    log.Fatal(err)
  }

  courseSS = schedulingDir.Sub("class")
  attendSS = schedulingDir.Sub("attends")

  var levels = []string{"intro", "for dummies", "remedial", "101", "201", "301", "mastery", "lab", "seminar"}
  var types = []string{"chem", "bio", "cs", "geometry", "calc", "alg", "film", "music", "art", "dance"}
  var times = []string{"2:00", "3:00", "4:00", "5:00", "6:00", "7:00", "8:00", "9:00", "10:00", "11:00",
                       "12:00", "13:00", "14:00", "15:00", "16:00", "17:00", "18:00", "19:00"}

  classes := make([]string, len(levels) * len(types) * len(times))

  for i := range levels {
    for j := range types {
      for k := range times {
        classes[i*len(types)*len(times)+j*len(times)+k] = fmt.Sprintf("%s %s %s", levels[i], types[j], times[k])
      }
    }
  }

  _, err = db.Transact(func (tr fdb.Transaction) (interface{}, error) {
    tr.ClearRange(schedulingDir)

    for i := range classes {
      tr.Set(courseSS.Pack(tuple.Tuple{classes[i]}), []byte(strconv.FormatInt(100, 10)))
    }

    return nil, nil
  })

  run(db, 10, 10)
}

func indecisiveStudent(db fdb.Database, id, ops int, wg *sync.WaitGroup) {
  studentID := fmt.Sprintf("s%d", id)

  allClasses := classes

  var myClasses []string

  for i := 0; i < ops; i++ {
    var moods []string
    if len(myClasses) > 0 {
      moods = append(moods, "drop", "switch")
    }
    if len(myClasses) < 5 {
      moods = append(moods, "add")
    }

    func() {
      defer func() {
        if r := recover(); r != nil {
          fmt.Println("Need to recheck classes:", r)
          allClasses = []string{}
        }
      }()

      var err error

      if len(allClasses) == 0 {
        allClasses, err = availableClasses(db)
        if err != nil {
          panic(err)
        }
      }

      switch moods[rand.Intn(len(moods))] {
      case "add":
        class := allClasses[rand.Intn(len(allClasses))]
        err = signup(db, studentID, class)
        if err != nil {
          panic(err)
        }
        myClasses = append(myClasses, class)
      case "drop":
        classI := rand.Intn(len(myClasses))
        err = drop(db, studentID, myClasses[classI])
        if err != nil {
          panic(err)
        }
        myClasses[classI], myClasses = myClasses[len(myClasses)-1], myClasses[:len(myClasses)-1]
      case "switch":
        oldClassI := rand.Intn(len(myClasses))
        newClass := allClasses[rand.Intn(len(allClasses))]
        err = swap(db, studentID, myClasses[oldClassI], newClass)
        if err != nil {
          panic(err)
        }
        myClasses[oldClassI] = newClass
      }
    }()
  }

  wg.Done()
}

func run(db fdb.Database, students, opsPerStudent int) {
  var wg sync.WaitGroup

  wg.Add(students)

  for i := 0; i < students; i++ {
    go indecisiveStudent(db, i, opsPerStudent, &wg)
  }

  wg.Wait()

  fmt.Println("Ran", students * opsPerStudent, "transactions")
}
```

See you！

