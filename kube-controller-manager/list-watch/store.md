# 5. store

`Store`是一个通用的存储接口，Reflector通过watch server的方式更新数据到store中，store给Reflector提供本地的缓存，让Reflector可以像消息队列一样的工作。

`Store`实现的是一种可以准确的写入对象和获取对象的机制。

```go
// Store is a generic object storage interface. Reflector knows how to watch a server
// and update a store. A generic store is provided, which allows Reflector to be used
// as a local caching system, and an LRU store, which allows Reflector to work like a
// queue of items yet to be processed.
//
// Store makes no assumptions about stored object identity; it is the responsibility
// of a Store implementation to provide a mechanism to correctly key objects and to
// define the contract for obtaining objects by some arbitrary key type.
type Store interface {
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)

	// Replace will delete the contents of the store, using instead the
	// given list. Store takes ownership of the list, you should not reference
	// it after calling this function.
	Replace([]interface{}, string) error
	Resync() error
}
```

其中`Replace`方法会删除原来store中的内容，并将新增的list的内容存入store中，即完全替换数据。

## 5.1.  cache

cache实现了store的接口，而cache的具体实现又是调用`ThreadSafeStore`接口来实现功能的。

cache的功能主要有以下两点：

- 通过keyFunc计算对象的key
- 调用ThreadSafeStorage接口的方法

```go
// cache responsibilities are limited to:
//	1. Computing keys for objects via keyFunc
//  2. Invoking methods of a ThreadSafeStorage interface
type cache struct {
	// cacheStorage bears the burden of thread safety for the cache
	cacheStorage ThreadSafeStore
	// keyFunc is used to make the key for objects stored in and retrieved from items, and
	// should be deterministic.
	keyFunc KeyFunc
}
```

其中ListAndWatch主要用到以下的方法：

**cache.Replace**

```go
// Replace will delete the contents of 'c', using instead the given list.
// 'c' takes ownership of the list, you should not reference the list again
// after calling this function.
func (c *cache) Replace(list []interface{}, resourceVersion string) error {
	items := map[string]interface{}{}
	for _, item := range list {
		key, err := c.keyFunc(item)
		if err != nil {
			return KeyError{item, err}
		}
		items[key] = item
	}
	c.cacheStorage.Replace(items, resourceVersion)
	return nil
}

```

**cache.Add**

```go
// Add inserts an item into the cache.
func (c *cache) Add(obj interface{}) error {
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Add(key, obj)
	return nil
}
```

**cache.Update**

```go
// Update sets an item in the cache to its updated state.
func (c *cache) Update(obj interface{}) error {
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Update(key, obj)
	return nil
}
```

**cache.Delete**

```go
// Delete removes an item from the cache.
func (c *cache) Delete(obj interface{}) error {
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Delete(key)
	return nil
}
```

## 5.2. ThreadSafeStore

cache的具体是调用`ThreadSafeStore`来实现的。

```go
// ThreadSafeStore is an interface that allows concurrent access to a storage backend.
// TL;DR caveats: you must not modify anything returned by Get or List as it will break
// the indexing feature in addition to not being thread safe.
//
// The guarantees of thread safety provided by List/Get are only valid if the caller
// treats returned items as read-only. For example, a pointer inserted in the store
// through `Add` will be returned as is by `Get`. Multiple clients might invoke `Get`
// on the same key and modify the pointer in a non-thread-safe way. Also note that
// modifying objects stored by the indexers (if any) will *not* automatically lead
// to a re-index. So it's not a good idea to directly modify the objects returned by
// Get/List, in general.
type ThreadSafeStore interface {
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
	Replace(map[string]interface{}, string)
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexKey string) ([]string, error)
	ListIndexFuncValues(name string) []string
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
	Resync() error
}
```

**threadSafeMap**

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```