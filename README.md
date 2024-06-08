# build-own-db-in-go

Build your own database from scratch in go （使用 go 从头构建你的数据库）学习笔记

## 概述

### 本书的学习过程

构建一个 b-tree -> 一个简单的 kv 数据库 -> 一个简单的关系型数据库

### 我们要关注的三个问题

- 持久化（Persistence）
    - How not to lose or corrupt your data. Recovering from a crash.
    - 如何避免丢失或损坏数据。从崩溃中恢复。
- 索引（Indexing）
    - Efficiently querying and manipulating your data. (B-tree).
    - 高效地查询和操作您的数据。（B 树）。
- 并发性（Concurrency）
    - How to handle multiple (large number of) clients. And transactions.
    - 如何处理多个（大量）客户。以及事务。

#### 持久化

> Why do we need databases? Why not dump the data directly into files?

直接写入到文件无法保证当程序发生意外时写入的完整性，比方说写入过程中突然崩溃或者意外断电
当发生意外的时候，简单地直接写入文件这个操作可能的结果有

- 仅丢失最后的结果
- 丢失部分数据
- 数据损坏

数据库在持久化层面要解决的就是意外崩溃后的恢复，保证数据的完整性。
如果我们不使用数据的话，也可以解决这个问题，类似于下面的操作：

- 将更新完的全部数据写入到一个新的文件中
- 调用 `fsync` 在新文件上
- 重命名新文件为旧文件，以覆盖旧文件

但是这样的操作只有较为小型的数据库可以，比如说 sqlite

#### 索引

数据库的查询大致分为两类，使用场景也大致分为两种

- OLAP
    - Analytical 分析型
    - 通常需要操作大规模的数据，涉及到聚合（aggregation）、分组（group）、链接（join）等操作
- OLTP
    - Transactional 事务型
    - 通常需要操作小规模数据，主要是根据索引进行点查和索引范围查询
    - 本书仅关注 OLTP

##### 为什么要关注索引？

即使很多应用不是实时系统，但是更快的响应用户需求使用合理的资源都是重要的，通过使用索引，在O(log(n)) 时间内找到数据。
如果忽略掉持久化存储的问题，我们可以假设数据是适合于内存的，那么如何快速找到数据就是数据结构要解决的问题，这个数据结构就是索引。通常数据库的索引会比内存要大，越大的索引越可以记住更多的数据。
常见的索引有 B-Tree 和 LSM-Tree，本书使用 B-Tree

#### 并发性

现代的应用程序不会是顺序地做任何事情，数据库也是一样。数据库常见的并发问题有

- 多个 reader 并发读
- 多个 reader 和 writer 并发地读和写， writer 是否需要独占数据库访问

即使是基于文件的 sqlite 也是支持一定的并发，但是在进程中处理会更为方便，所以大多数的数据库需要使用 server
并发的引入就需要考虑原子化操作，这就增加了一个新的概念：事务（transaction）

## 文件 vs 数据库

就像上面所提到的那样，直接写入文件有两种简单的方式，代码示例如下

```go
// 写入新的文件后对文件重命名以覆盖旧文件
func SaveData(path string, data []byte) error {
  tmp := fmt.Sprintf("%s.tmp.%d", path, randomInt())
  fp, err := os.OpenFile(tmp, os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0664)
  if err != nil {
  return err
  }
  defer fp.Close()
  
  _, err = fp.Write(data)
  if err != nil {
  os.Remove(tmp)
  return err
  }
  
  return os.Rename(tmp, path)
}

// 上述代码存在一个问题，Linux 下文件的写入是缓冲的，所以需要调用 fsync 刷新缓存
func SaveDataFsync(path string, data []byte) error {
  tmp := fmt.Sprintf("%s.tmp.%d", path, randomInt())
  fp, err := os.OpenFile(tmp, os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0664)
  if err != nil {
  return err
  }
  defer fp.Close()
  
  _, err = fp.Write(data)
  if err != nil {
  os.Remove(tmp)
  return err
  }
  
  err = fp.Sync() // fsync
  if err != nil {
  os.Remove(tmp)
  return err
  }
  
  return os.Rename(tmp, path)
}
```

另一种方式就是追加写入的形式，这种方式的好处是可以保证数据的完整性
```go
func LogCreate(path string) (*os.File, error) {
    return os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0664)
}

func LogAppend(fp *os.File, line string) error {
    buf := []byte(line)
    buf = append(buf, '\n')
    _, err := fp.Write(buf)
    if err != nil {
        return err
    }
    return fp.Sync() // fsync
}

```
