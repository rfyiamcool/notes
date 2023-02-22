# 源码分析 hashicorp raft 的持久化存储的实现原理

> 本文基于 hashicorp/raft `v1.3.11` 版本进行源码分析

raft 的持久化指的是 raft log 和 raft 一些元数据的持久化, hashicorp/raft 的持久化组件需要实现 StableStore 和 LogStore 接口.

**golang hashicorp raft 原理系列**

- [源码分析 hashicorp raft election 选举的设计实现原理](https://github.com/rfyiamcool/notes/blob/main/hashicorp_raft_election_code.md)
- [源码分析 hashicorp raft replication 日志复制的实现原理](https://github.com/rfyiamcool/notes/blob/main/hashicorp_raft_replication_code.md)
- [源码分析 hashicorp raft 持久化存储的实现原理](https://github.com/rfyiamcool/notes/blob/main/hashicorp_raft_persistence_code.md)
- [源码分析 hashicorp raft snapshot 快照的实现原理](https://github.com/rfyiamcool/notes/blob/main/hashicorp_raft_snapshot_code.md)

## raft 持久化概述

当使用 hashicorp/raft 实例化 raft 对象时, 需要传入实现了 StableStore 和 LogStore 接口的存储对象. 

- LogStore 用来存储 raft log 日志, 需要实现 raft log 日志按照 Index 的增删改查.
- StableStore 用来 `CurrentTerm`, `LastVoteTerm` 和 `LastVoteCand` 的键值. 通常不会使用该对象实现业务的存储.

```go
// LogStore is used to provide an interface for storing
// and retrieving logs in a durable fashion.
type LogStore interface {
	// FirstIndex returns the first index written. 0 for no entries.
	FirstIndex() (uint64, error)

	// LastIndex returns the last index written. 0 for no entries.
	LastIndex() (uint64, error)

	// GetLog gets a log entry at a given index.
	GetLog(index uint64, log *Log) error

	// StoreLog stores a log entry.
	StoreLog(log *Log) error

	// StoreLogs stores multiple log entries.
	StoreLogs(logs []*Log) error

	// DeleteRange deletes a range of log entries. The range is inclusive.
	DeleteRange(min, max uint64) error
}

// StableStore is used to provide stable storage
// of key configurations to ensure safety.
type StableStore interface {
	Set(key []byte, val []byte) error

	// Get returns the value for key, or an empty byte slice if key was not found.
	Get(key []byte) ([]byte, error)

	SetUint64(key []byte, val uint64) error

	// GetUint64 returns the uint64 value for key, or 0 if key was not found.
	GetUint64(key []byte) (uint64, error)
}
```

下面是实例化 hashicorp raft 的例子.

```go
import (
	"os"
	"fmt"

	"github.com/hashicorp/raft"
	"github.com/dgraph-io/badger/v3/options"

	raftbadger "github.com/rfyiamcool/raft-badger"
)

func main() {
	var (
		logStore raft.LogStore
		stableStore raft.StableStore
	)

	cfg = raftbadger.Config{
		DataPath: "/tmp/raft",
		Compression: "zstd", // zstd, snappy
		DisableLogger: true,
	}

	opts := badger.DefaultOptions(cfg.DataPath)
	badgerDB, err := raftbadger.New(cfg, &opts)
	if err != nil {
		fmt.Println("fail to create new badger sotrage, err: %s", err.Error())
		os.Exit(1)
	}

	logStore = badgerDB
	stableStore = badgerDB

	r, err := raft.NewRaft(config, (*fsm)(s), logStore, stableStore, snapshots, transport)
	...
}
```

下面说下 hashicorp raft 存储的几种实现.

## InmemStore 内存型存储

`InmemStore` 是 hashicorp/raft 里用来测试的内存 store, 该 store 没有持久化.

代码地址: `github.com/hashicorp/raft/inmem_store.go`

```go
type InmemStore struct {
	l         sync.RWMutex
	lowIndex  uint64
	highIndex uint64
	logs      map[uint64]*Log
	kv        map[string][]byte
	kvInt     map[string]uint64
}

func NewInmemStore() *InmemStore {
	i := &InmemStore{
		logs:  make(map[uint64]*Log),
		kv:    make(map[string][]byte),
		kvInt: make(map[string]uint64),
	}
	return i
}

// 获取第一个 log index
func (i *InmemStore) FirstIndex() (uint64, error) {
	i.l.RLock()
	defer i.l.RUnlock()
	return i.lowIndex, nil
}

// 获取最后一个 log index
func (i *InmemStore) LastIndex() (uint64, error) {
	i.l.RLock()
	defer i.l.RUnlock()
	return i.highIndex, nil
}

// 获取 index 的日志
func (i *InmemStore) GetLog(index uint64, log *Log) error {
	i.l.RLock()
	defer i.l.RUnlock()
	l, ok := i.logs[index]
	if !ok {
		return ErrLogNotFound
	}
	*log = *l
	return nil
}

// 存储日志
func (i *InmemStore) StoreLog(log *Log) error {
	return i.StoreLogs([]*Log{log})
}

// 存储日志
func (i *InmemStore) StoreLogs(logs []*Log) error {
	i.l.Lock()
	defer i.l.Unlock()
	for _, l := range logs {
		i.logs[l.Index] = l
		if i.lowIndex == 0 {
			i.lowIndex = l.Index
		}
		if l.Index > i.highIndex {
			i.highIndex = l.Index
		}
	}
	return nil
}

// 删除日志
func (i *InmemStore) DeleteRange(min, max uint64) error {
	i.l.Lock()
	defer i.l.Unlock()
	for j := min; j <= max; j++ {
		delete(i.logs, j)
	}
	if min <= i.lowIndex {
		i.lowIndex = max + 1
	}
	if max >= i.highIndex {
		i.highIndex = min - 1
	}
	if i.lowIndex > i.highIndex {
		i.lowIndex = 0
		i.highIndex = 0
	}
	return nil
}

// 配置 kv
func (i *InmemStore) Set(key []byte, val []byte) error {
	i.l.Lock()
	defer i.l.Unlock()
	i.kv[string(key)] = val
	return nil
}

// 获取 kv
func (i *InmemStore) Get(key []byte) ([]byte, error) {
	i.l.RLock()
	defer i.l.RUnlock()
	val := i.kv[string(key)]
	if val == nil {
		return nil, errors.New("not found")
	}
	return val, nil
}

// 配置 index
func (i *InmemStore) SetUint64(key []byte, val uint64) error {
	i.l.Lock()
	defer i.l.Unlock()
	i.kvInt[string(key)] = val
	return nil
}

// 获取 index
func (i *InmemStore) GetUint64(key []byte) (uint64, error) {
	i.l.RLock()
	defer i.l.RUnlock()
	return i.kvInt[string(key)], nil
}
```

## raft-boltdb

hashicorp/raft 官方有个 raft-boltdb 扩展项目, 该项目使用 boltdb 实现 hashicorp/raft 需要的 StableStore 和 LogStore 存储接口.

项目地址: `https://github.com/hashicorp/raft-boltdb`

boltdb 自带 first 和 last 方法, 下面是 FirstIndex 和 LastIndex 的实现.

```go
// FirstIndex returns the first known index from the Raft log.
func (b *BoltStore) FirstIndex() (uint64, error) {
	tx, err := b.conn.Begin(false)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()

	curs := tx.Bucket(dbLogs).Cursor()
	if first, _ := curs.First(); first == nil {
		return 0, nil
	} else {
		return bytesToUint64(first), nil
	}
}

// LastIndex returns the last known index from the Raft log.
func (b *BoltStore) LastIndex() (uint64, error) {
	tx, err := b.conn.Begin(false)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()

	curs := tx.Bucket(dbLogs).Cursor()
	if last, _ := curs.Last(); last == nil {
		return 0, nil
	} else {
		return bytesToUint64(last), nil
	}
}
```

GetLog 获取日志和 StoreLog 存储 raft 日志的实现.

```go
func (b *BoltStore) GetLog(idx uint64, log *raft.Log) error {
	tx, err := b.conn.Begin(false)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	bucket := tx.Bucket(dbLogs)
	val := bucket.Get(uint64ToBytes(idx))

	if val == nil {
		return raft.ErrLogNotFound
	}
	return decodeMsgPack(val, log)
}

func (b *BoltStore) StoreLog(log *raft.Log) error {
	return b.StoreLogs([]*raft.Log{log})
}

func (b *BoltStore) StoreLogs(logs []*raft.Log) error {
	// 开启事务
	tx, err := b.conn.Begin(true)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	batchSize := 0
	for _, log := range logs {
		// 把 index 转换成 []byte
		key := uint64ToBytes(log.Index)

		// log 使用 msgpack 编码为 []byte.
		val, err := encodeMsgPack(log)
		if err != nil {
			return err
		}

		logLen := val.Len()
		bucket := tx.Bucket(dbLogs)

		// 写kv
		if err := bucket.Put(key, val.Bytes()); err != nil {
			return err
		}
		batchSize += logLen
	}

	// 提交事务
	return tx.Commit()
}
```

Set 和 Get 的读写实现, 也没什么可说的.

```go
// Set is used to set a key/value set outside of the raft log
func (b *BoltStore) Set(k, v []byte) error {
	tx, err := b.conn.Begin(true)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	bucket := tx.Bucket(dbConf)
	if err := bucket.Put(k, v); err != nil {
		return err
	}

	return tx.Commit()
}

// Get is used to retrieve a value from the k/v store by key
func (b *BoltStore) Get(k []byte) ([]byte, error) {
	tx, err := b.conn.Begin(false)
	if err != nil {
		return nil, err
	}
	defer tx.Rollback()

	bucket := tx.Bucket(dbConf)
	val := bucket.Get(k)

	if val == nil {
		return nil, ErrKeyNotFound
	}
	return append([]byte(nil), val...), nil
}
```

## raft-badger

boltdb 在写入压力大时表现并不好, 本人使用 badger 实现了 LogStore 和 StableStore 存储. 但 badger 没有 table 和 bucket 的设计, 所以需要自己来实现表.  badger 的 sdk 易用性显然没有 boltdb 好. 另外 `raft-badger` 已经加入到 badger 的首页里了.

[https://github.com/rfyiamcool/raft-badger](https://github.com/rfyiamcool/raft-badger)

![raft-badger](https://github.com/rfyiamcool/raft-badger/raw/main/docs/design1.jpg)

raft log 使用 `logs-` 前缀头, stable 存储则使用 `meta-` 前缀头. 每次读写的时候需要追加 prefix 前缀头. 另外对日志进行迭代时, 需要 seek 的 key 依然需要追加前缀头.

```go
var (
	// bucket for raft logs
	prefixDBLogs = []byte("logs-")

	// bucket for raft config
	prefixDBMeta = []byte("meta-")
)

// buildConfKey prefixDBLogs + key
func buildLogsKey(idx uint64) []byte {
	bs := append([]byte{}, prefixDBLogs...)
	return append(bs, uint64ToBytes(idx)...)
}

// buildMetaKey prefixDBConfig + key
func buildMetaKey(key []byte) []byte {
	var out = make([]byte, 0, len(key)+len(prefixDBMeta))
	out = append(out, prefixDBMeta...)
	return append(out, key...)
}

// StoreLogs is used to store a set of raft logs
func (s *Storage) StoreLogs(logs []*raft.Log) error {
	maxBatchSize := s.db.MaxBatchSize()
	min := uint64(0)
	max := uint64(len(logs))
	ranges := s.generateRanges(min, max, maxBatchSize)
	for _, r := range ranges {
		txn := s.db.NewTransaction(true)
		defer txn.Discard()

		for index := r.from; index < r.to; index++ {
			log := logs[index]
			key := buildLogsKey(log.Index)
			out, err := encodeMsgpack(log)
			if err != nil {
				return err
			}
			if err := txn.Set(key, out.Bytes()); err != nil {
				return err
			}
		}
		if err := txn.Commit(); err != nil {
			return err
		}
	}
	return nil
}

// GetLog is used to get a log from Badger by a given index.
func (s *Storage) GetLog(idx uint64, log *raft.Log) error {
	return s.db.View(func(txn *badger.Txn) error {
		item, err := txn.Get([]byte(buildLogsKey(idx)))
		if item == nil {
			return raft.ErrLogNotFound
		}

		var val []byte
		val, err = item.ValueCopy(val)
		if err != nil {
			return err
		}

		return decodeMsgpack(val, log)
	})
}
```

badger 要实现 FirstIndex 和 LastIndex 方法, 需要配合迭代器扫描才可实现. boltdb 直接就暴露了 First 和 Last 接口.

```go
// FirstIndex get the first index from the Raft log.
func (s *Storage) FirstIndex() (uint64, error) {
	var (
		first = uint64(0)
		err   error
	)

	err = s.db.View(func(txn *badger.Txn) error {
		iter := txn.NewIterator(badger.DefaultIteratorOptions) // order asc
		defer iter.Close()

		var has bool
		iter.Seek(prefixDBLogs)
		if iter.ValidForPrefix(prefixDBLogs) {
			item := iter.Item()
			first = parseIndexByLogsKey(item.Key())
			has = true
		}
		if !has {
			return ErrNotFoundFirstIndex
		}
		return nil
	})
	return first, err
}

var maxSeekKey = append(getPrefixDBLogs(), uint64ToBytes(math.MaxUint64)...)

// LastIndex get the last index from the Raft log.
func (s *Storage) LastIndex() (uint64, error) {
	var (
		last = uint64(0)
		err  error
	)

	err = s.db.View(func(txn *badger.Txn) error {
		opts := badger.IteratorOptions{
			PrefetchValues: true, // prefetch values
			PrefetchSize:   1,    // default 100
			Reverse:        true, // order desc
		}

		iter := txn.NewIterator(opts)
		defer iter.Close()

		var has bool
		iter.Seek(maxSeekKey)
		if iter.ValidForPrefix(prefixDBLogs) {
			item := iter.Item()
			key := item.Key()[len(prefixDBLogs):]
			last = bytesToUint64(key)
			has = true
		}
		if !has {
			return ErrNotFoundLastIndex
		}
		return nil
	})
	return last, err
}
```

## LogCache 缓存组件

hashicorp/raft 内置了 LogStore 的缓存实现, 其实现原理简单粗暴, 就是使用 cache 切片来存放 Log.

代码位置: `github.com/hashicorp/raft/log_cache.go`

```go
type LogCache struct {
	store LogStore

	cache []*Log
	l     sync.RWMutex
}

func (c *LogCache) GetLog(idx uint64, log *Log) error {
	// 通过取余的方式获取缓存 log 
	c.l.RLock()
	cached := c.cache[idx%uint64(len(c.cache))]
	c.l.RUnlock()

	if cached != nil && cached.Index == idx {
		*log = *cached
		return nil
	}

	// 如果缓存没命中, 则使用 store 回源读取.
	return c.store.GetLog(idx, log)
}

func (c *LogCache) StoreLogs(logs []*Log) error {
	// 首先回源写日志
	err := c.store.StoreLogs(logs)
	if err != nil {
		return fmt.Errorf("unable to store logs within log store, err: %q", err)
	}
	c.l.Lock()
	for _, l := range logs {
		// 根据取余的位置覆盖日志
		c.cache[l.Index%uint64(len(c.cache))] = l
	}
	c.l.Unlock()
	return nil
}

// 由于 cache 是个 ringbuffer 结构, 为避免复杂代码, 直接清空.
func (c *LogCache) DeleteRange(min, max uint64) error {
	c.l.Lock()
	c.cache = make([]*Log, len(c.cache))
	c.l.Unlock()

	// 缓存清空, 源 store 也要范围删除.
	return c.store.DeleteRange(min, max)
}
```

FirstIndex 和 LastIndex 没做缓存? 大概是因为该操作只有 snapshot 操作时才会调用, 属于低频操作, 没必要缓存吧.

```go
func (c *LogCache) FirstIndex() (uint64, error) {
	return c.store.FirstIndex()
}

func (c *LogCache) LastIndex() (uint64, error) {
	return c.store.LastIndex()
}
```

## 小结

hashicorp/raft 自带的 InmemStore 存储是全内存的，可用来测试, raft-boltdb 是 hashicorp 官方提供的存储组件, consul 也使用了该持久化组件, raft-badger 是本人开发持久化组件, 就 raft 场景来说, 相比 boltdb 具有更好的性能和压缩效果.