+++
title = '阅读：Go 并发编程实战 1-4'
date = 2024-05-22T15:28:22+08:00
tags = ['Go', '并发编程', '笔记']
+++

## Go 并发编程实战

https://time.geekbang.org/column/intro/100061801 挺好的实战教程

## 资源并发访问问题

使用 10 个线程对变量 counter 进行增加，每个增加 10000，结果最后是 10 \* 10000 吗？

### 互斥锁

临界区就是一个被共享的资源。为了避免竞争导致并发结果不对，可以使用互斥锁，限定临界区只能同时由一个线程持有，比如 Mutex 锁。

### Race Detector

Go race detector 基于 Google 的 C/C++ sanitizers 技术实现，在编译期检测对共享变量的非同步访问。

## Mutex 实现原理

最初的 Mutex 使用信号量控制 goroutine 的阻塞休眠和唤醒 + flag 字段标记是否持有锁（CAS 进行判断，0 未持有，1 锁被持有且无等待，n 锁被持有且有 n-1 个等待）

Go 可以使用 defer 来释放锁

初期的 Mutex 存在几个问题：排队请求锁需要上下文切换，性能不好

### State 字段

2011 年调整 Mutex 的 state 字段为 int32，第一位表示锁是否被持有，第二位表示唤醒的 goroutine，第三部分是阻塞等待的数量。

#### 获取锁

1. CAS 检测 state 是否为 0，是则 Swap 成 mutexLocked 状态 `atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked)` ---> Fast Path 获取锁
2. 检测旧状态并且加锁 `old | mutexLocked`，如果加锁失败则进入等待 `new = old + 1 << mutexWaiterShift`
3. 如果被唤醒则消除 mutexWoken 状态 `new &^= mutexWoken`
4. CAS 旧状态进入新状态，如果未被加锁则继续循环，否则请求信号量唤醒。

用位运算来判断状态。

#### 释放锁

1. `atomic.AddInt32(&m.state, -mutexLocked)` 去掉锁标志
2. 检测旧状态，如果没有等待者，则退出
3. 如果有等待者，设置唤醒标志

注：Unlock 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，Unlock 也不会对此进行检查。Mutex 的这个设计一直保持至今。

### 自旋

在 2011 年的基础上，不论是新来的 goroutine 还是被唤醒的，都会尝试请求锁且可以自旋 `runtime.canSpin(iter)` 缓解了饥饿问题。

目前的 Mutex 源码非常复杂 [mutex.go](https://go.dev/src/sync/mutex.go) 分成了 LockFast 和 LockSlow，也加入了超时的一些判断，都在解决饥饿问题。

### 饥饿模式、正常模式

被唤醒的 goroutine 和新来的 goroutine 需要竞争，会导致饥饿产生。如果 waiter 获取不到锁的时间超过了阈值（1ms），Mutex 进入饥饿模式。

- 饥饿模式：Mutex 拥有者会直接把锁交给队列最前面的等待者，新来的 goroutine 不会尝试获取锁，不自旋，加入等待队列尾部。
- 正常模式：如果 waiter 已经是队列最后一个，没有其他等待锁的 goroutine，或者 waiter 等待时间小于阈值（1ms）。

正常模式具有更好的性能。

```go
const (
	mutexLocked           = 1 << iota // iota 0, 1 << 0 = 1
	mutexWoken                        // iota 1, 1 << 1 = 2
	mutexStarving                     // iota 2, 1 << 2 = 4
	mutexWaiterShift      = iota      // iota 3
	starvationThresholdNs = 1e6
)
```

这种用法是 Constant declarations

## Mutex 易错场景

1. Lock/Unlock 不是成对出现，会产生死锁，或 Unlock 一个未加锁的 Mutex 而导致 panic。（太多 if-else/重构/误写/...）
2. 重入，Java 中有 ReentrantLock 可重入锁，而 Mutex 是不可重入锁。 Lock() 之后不可以再 Lock()。
3. 死锁，避免死锁的四个条件：互斥、拥有和等待、不可剥夺、环路等待。Go 运行时有死锁检测功能
4. Copy 已使用的 Mutex，sync 库的同步原语在使用后是不能复制的（Mutex 是有状态的对象，state 记录锁状态），复制使用后的 Mutex 会直接加锁。避免值拷贝传入函数参数，go vet 可以检测这种情况，使用 copylock 分析器静态分析。

### Go 实现可重入锁

锁需要记住当前是哪个 goroutine 持有该锁

#### goroutine id

通过 `runtime.Stack` 获取栈帧信息和 goroutine id，比如 `goroutine 1 [running]` 通过字符串处理得到 goroutine id。或者直接调用 `petermattis/goid` 等第三方库。

```go
type RecursiveMutex struct {
	sync.Mutex
	owner     int64 // 当前持有锁的 goroutine id
    recursion int32 // 这个 goroutine 重入的次数
}
func (m *RecursiveMutex) Lock() {
	gid := goid.Get()    // 如果当前持有锁的goroutine就是这次调用的goroutine,说明是重入
	if atomic.LoadInt64(&m.owner) == gid {
		m.recursion++
		return
	}
	m.Mutex.Lock()    // 获得锁的goroutine第一次调用，记录下它的goroutine id,调用次数加1
	atomic.StoreInt64(&m.owner, gid)
	m.recursion = 1
}

func (m *RecursiveMutex) Unlock() {
	gid := goid.Get()    // 非持有锁的goroutine尝试释放锁，错误的使用
	if atomic.LoadInt64(&m.owner) != gid {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", m.owner, gid))
    }    // 调用次数减1
	m.recursion--
	if m.recursion != 0 { // 如果这个goroutine还没有完全释放，则直接返回
		return
	}    // 此goroutine最后一次调用，需要释放锁
	atomic.StoreInt64(&m.owner, -1)
	m.Mutex.Unlock()
}
```

#### Token

Go 开发者没有暴露 goroutine id，开发者可以自己提供 token

```go
type RecursiveMutex struct {
	sync.Mutex
	token int64
	recursion int32
}

func(m *RecursiveMutex) Lock(token int64) {
	if atomic.LoadInt64(&m.token) == token {
		m.recursion++
		return
	}
	m.Mutex.Lock()
	atomic.StoreInt64(&m.token, token)
	m.recursion = 1
}

func(m *RecursiveMutex) Unlock(token int64) {
	if atomic.LoadInt64(&m.token) != token {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", m.token, token))
    }    // 调用次数减1
	m.recursion--
	if m.recursion != 0 { // 如果这个goroutine还没有完全释放，则直接返回
		return
	}    // 此goroutine最后一次调用，需要释放锁
	atomic.StoreInt64(&m.token, 0)
	m.Mutex.Unlock()
}
```

### 流行项目 issues

#### Docker (Moby)

[Moby #36114](https://github.com/moby/moby/pull/36114/files)

`hotAddVHDsAtStart` 方法对 serviceVM 进行 VHD 的添加，并且会对 svm 加锁，如果添加 VHS 失败会调用 `hotRemoveVHDsAtStart` 方法去掉 VHD，而这个方法也进行了 svm.Lock()

因为 Mutex 不可重入，会导致死锁，所以改成了 `hotRemoveVHDsNoLock` 不用锁的方法来移除 VHD。

[Moby #34881](https://github.com/moby/moby/pull/34881/files) 这个 PR 在 if 判断后 return 却没有释放锁

其他还有 36840、37583、35517、35482、33305、32826、30696、29554、29191、28912、26507 等 issues。

#### Kubernetes

[Kubernetes #45192](https://github.com/kubernetes/kubernetes/pull/45192/files?diff=split&w=0) 忘记 Unlock 导致死锁

[Kubernetes #72361](https://github.com/kubernetes/kubernetes/pull/72361/files?diff=split&w=0) 使用 mutex 解决竞争问题

[Kubernetes #71617](https://github.com/kubernetes/kubernetes/pull/71617/files?diff=split&w=0) 使用 mutex 解决竞争问题

[Kubernetes #70605](https://github.com/kubernetes/kubernetes/pull/70605/files?diff=split&w=0) 使用 mutex 解决竞争问题

#### gRPC-go

[gRPC-go #795](https://github.com/grpc/grpc-go/pull/795/files) Unlock() 写错了？所以还是 defer unlock 合理。

[gRPC-go #1318](https://github.com/grpc/grpc-go/pull/1318/files) 添加 Lock 防止 data races

[gRPC-go #2074](https://github.com/grpc/grpc-go/pull/2074/files) 添加 Lock 防止 data races

#### etcd

Distributed reliable key-value store for the most critical data of a distributed system

[etcd #10419](https://github.com/etcd-io/etcd/pull/10419/files?diff=split) etcd Compact 方法调用的时候，使用了 mu.Lock() 而 Store 方法也会调用
