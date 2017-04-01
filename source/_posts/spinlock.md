---
title: 用golang实现一个自旋锁
date: 2017-04-01 15:05:13
tags: [golang,spinlock]
categories: [技术]
---
In software engineering, a spinlock is a lock which causes a thread trying to acquire it to simply wait in a loop ("spin") while repeatedly checking if the lock is available(摘自wikipedia)。
<!--more-->
# 自旋锁
利用cpu提供的原子原语来实现。
## 直接上码
```go
type spinLock uint32

func (sl *spinLock) Lock() {
	for !atomic.CompareAndSwapUint32((*uint32)(sl), 0, 1) {
		runtime.Gosched()
	}
}
func (sl *spinLock) Unlock() {
	atomic.StoreUint32((*uint32)(sl), 0)
}

func NewSpinLock() sync.Locker {
	var lock spinLock
	return &lock
}
```
