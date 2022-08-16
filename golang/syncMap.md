# sync.Map

普通的map在并发读写的情况下，会抛出不可被捕获的panic。

一般的做法我们可以考虑给map再加一个互斥锁或者读写锁，或者是将大map拆分成多个小map，降低锁粒度。不过在go官方提供一种并发安全的sync.Map

## 使用

```go
func main() {
	m := sync.Map{}

	// 1. 写入 k-v
	m.Store(1, 1)
	m.Store(2, 2)

	// 2. 读取 k-v, 返回对应的值以及是否存在 map 中
	val, ok := m.Load(1)
	fmt.Println(val, ok) // 1 true
	val, ok = m.Load(3)
	fmt.Println(val, ok) // <nil> false

	// 3. 遍历 map, 当自定义函数返回 false 时，遍历结束
	// 1 1
	// 2 2
	m.Range(func(key, value interface{}) bool {
		fmt.Println(key, value)
		return true
	})

	// 4. 删除 k-v
	m.Delete(2)
	fmt.Println(m.Load(2)) // <nil> false

	// 5. 更新或写入
	// 如果已经存在 k-v, 则使用新 value 更新, 并且返回旧值
	// 如果不存在 k-v, 则写入
	act, loaded := m.LoadOrStore(1, 10)
	fmt.Println(act, loaded) // 1 true
	act, loaded = m.LoadOrStore(2, 20)
	fmt.Println(act, loaded) // 20 false

	// 6. 删除 k-v, 并且 k-v 已经存在的话，会返回旧值
	val, loaded = m.LoadAndDelete(1)
	fmt.Println(val, loaded) // 1 true
	val, loaded = m.LoadAndDelete(3)
	fmt.Println(val, loaded) // <nil> false
}
```

## 源码分析

### 数据结构

sync.Map的数据结构

```go
type Map struct {
	mu Mutex
	// read 包含了 map 内容的一部分
	// read 可以被并发的读而不需要加锁，如果需要对 read 进行写入操作，则需要获取 mu 锁
	read atomic.Value // readOnly
	// dirty 包含了需要 mu 锁的 map 的部分
	dirty map[interface{}]*entry
	// 当 misses 达到一定次数后，会将 dirty 提升为 read，并且下一次的 store 操作会复制一个新的 dirty
	misses int
}

// Map 中 read 实际存储的结构
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true if the dirty map contains some key not in m.
}
```

在sync.Map中包含了`read`和`dirty`两个map，这些概念刚开始可能看的有点懵。可以先忽略具体的含义

```go
// map 中 value 实际存储的结构
type entry struct {
	p unsafe.Pointer // *interface{}
}
```

p有三种状态：

- nil：表示该entry被删除，并且 dirty 为nil
- expunged：表示该entry被删除，但dirty不为nil，且该entry不在dirty中
- 其他：该entry是一个可用的值，并且存在于read中，如果dirty不为nil，也存在于dirty中

### Store操作

```go
func (m *Map) Store(key, value interface{}) {
	// 如果在 read 中，则尝试使用 CAS 直接更新
	// 如果 entry.p 是 expunged，表示已删除，tryStore 则会返回 false
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	// 加锁后再检查一次 read
	if e, ok := read.m[key]; ok {
		// 如果存在 read 中，且 entry.p 为 expunged
		// 1. 说明该 entry 之前被删除了，并且 dirty 不为 nil，entry 也不在 dirty 中
		// 2. 通过 CAS 将 entry.p 更新为 nil
		// 3. 将 entry 写入到 dirty 中
		// 4. 通过 CAS 将 entry.p 更新为新值的指针
		if e.unexpungeLocked() {
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		// 在 read 中不存在该 k-v，但在 dirty 中存在，则直接更新 entry.p
		e.storeLocked(&value)
	} else {
		// read 和 dirty 中都没有 k-v
		if !read.amended {
			m.dirtyLocked()
			// 将 amended 设置为 true，因为接下来需要在 dirty 中添加 k-v，那么 read 和 dirty 将不再相同
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

// 如果 dirty 为 nil，则创建相同大小的 dirty，并且将 read 中未删除的 entry 复制到 dirty 中
func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read, _ := m.read.Load().(readOnly)
	m.dirty = make(map[interface{}]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

// 如果 entry.p 为 nil，则将其更新为 expunged
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}
```

流程梳理：

1. 首先在read中检查，如果存在，且entry.p不是expunged，则直接更新成新值
2. 如果在read中，且entry.p为expunged，则首先将其更新成nil，然后将该entry写入到dirty中，再将entry.p更新成新值
3. 如果不在read中，但在dirty中，则直接将entry.p更新成新值
4. 不在read也不在dirty中时，则检查amended是否为false，两种情况amended为false，1）第一次进行Store操作，2）Load操作miss到达一定次数，dirty提升为read时（这两个情况下dirty应该都为nil？）。如果amended为false，那么我们需要将read中未删除的entry复制到dirty中，同时将已经被删除的entry从nil更新成expunged，然后将amended设置为true
5. 将新的entry写入到dirty中，那么此时read和dirty不再相同，amended应为true

### Load操作

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 在 read 中没有找到 k-v，但是 dirty 中存在 read 没有的 k-v
	if !ok && read.amended {
		m.mu.Lock()
		// double-check，再次从 read 中读一次，防止加锁前，dirty 提升为 read 了，那么之前取的 read 是旧的
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		// 在 read 中没有找到 k-v，但是 dirty 中存在 read 没有的 k-v
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// 无论是否找到，都会使得 sync.Map 的 misses 字段自增。
			// 如果 misses 到达了 dirty 的长度，还需要将 dirty 提升为 read
			m.missLocked()
		}
		m.mu.Unlock()
	}
	// read 和 dirty 中都没有 k-v
	if !ok {
		return nil, false
	}
	return e.load()
}

func (e *entry) load() (value interface{}, ok bool) {
	p := atomic.LoadPointer(&e.p)
    // entry 是被删除的状态
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*interface{})(p), true
}
```

流程梳理：

1. 首先不加锁，尝试从read中获取k-v
2. 如果read中没有，但amended为true，表示dirty中存在read中没有的k-v，因此我们还需要取dirty中查找
3. 由于检查和加锁操作不是原子的，可能在加锁前dirty被提升为read了，所以此时的read可能是旧的，因此要再取一次read，再从read中获取k-v
4. read中未获取到，且amended为true，则尝试从dirty中获取
5. 无论在dirty中是否找到k-v，都要让misses自增，并且如果misses达到dirty的长度，还需要将dirty提升为read
6. 最后如果entry是删除状态，需要返回空

流程总结：

1. 先尝试在read中获取，再加锁（需要double-check）从dirty中获取
2. 无论在dirty中是否找到都认为miss，因此增加misses
3. 如果miss到达一定次数，需要将dirty提升为read，提高之后的load效率，因为这样之后在read中就可以直接获取到k-v

### Delete/LoadAndDelete操作

```go
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	m.LoadAndDelete(key)
}

func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 在 read 中未找到，且 dirty 中有 read 没有的 k-v，再去 dirty 中找
	if !ok && read.amended {
		m.mu.Lock()
		// 与 Load 一样的 double-check
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// 从 dirty 中移除
			delete(m.dirty, key)
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		// 如果存在于 read 或 dirty 中，标记删除
		return e.delete()
	}
	return nil, false
}

func (e *entry) delete() (value interface{}, ok bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		// 已经是删除状态
		if p == nil || p == expunged {
			return nil, false
		}
		// 变换为 nil
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return *(*interface{})(p), true
		}
	}
}
```

流程梳理：

1. 首先尝试从read中查找，如果在read中找到，那么直接标记删除
2. 如果在read中未找到，且dirty中存在read没有的k-v，从dirty中查找
3. 直接从dirty中移除，并且增加misses计数
4. 标记删除时，将entry的状态通过CAS方式设置为nil

流程总结：

1. 与Load操作类似，先从read中找，再加锁（需要double-check）从dirty中查找
2. 如果k-v存在于read中，那么我们标记删除，而不直接移除，这样Load操作会在read中命中，而不需要锁去检查dirty
3. 如果k-v只存在于dirty中，那么我们直接将其从dirty中移除，并且标记为nil，交给GC回收

### entry

entry中有个特殊的状态expunged，为什么需要这个状态值呢？

entry.p的状态图

![p-status](https://qraffa-1304595678.cos.ap-guangzhou.myqcloud.com/img/p-status.svg)

**我们先来看一下什么时候entry.p会成为expunged。**

在`Store`函数中，当在read和dirty中都没有找到我们需要的k-v的时候，我们检查read和amended，如果为false，则调用`dirtyLocked`函数；在该函数内，当dirty为nil时，我们会创建一个和read相同大小的dirty，同时将read中未被删除的entry复制到dirty中，并且将entry.p为nil的，更新成expunged。

**那么什么时候entry.p是nil的呢？**

当我们调用`Delete`函数时，会将entry.p更新成nil，表示删除。如果一个entry只存在于dirty中，会从dirty中直接移除；如果一个entry存在于read中，那么我们不会将其从read中移除，而是设置为nil。

**那么什么情况下一个entry会存在read中呢？**

通过`Store`函数源码，我们知道，一个之前未存在过的新创建的entry都是直接写入到dirty中的，那么写入新的肯定不是影响read的。再看`Load`函数，其中当去dirty中查找entry时，会调用`missLocked`函数。在该函数中，当misses到达一定值后，会将当前的dirty提升到read中，并且将dirty只为nil。那么这样一来entry就会存在于read中了。

结合上面几点，如果我们使用如下操作序列，那么此时会有一个expunged的entry

```go
func statusFunc() {
	var m sync.Map

	// 此时entry.p为pointer，且entry只存在于dirty中
	m.Store(1, 1)

	// 这次Load操作会需要在dirty中查找，因此会触发将dirty提升到read的过程，那么此时entry存在于read中
	m.Load(1)

	// 我们将该entry删除，在read中找到，此时entry.p从pointer变更为nil
	m.Delete(1)

	// 一次新entry的插入，会触发将read中的未删除的entry复制到dirty的操作，而被删除的则从nil变更为expunged
	m.Store(2, 2)
}
```

