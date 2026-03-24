---
title: "Go 调度器(三)：GMP高效调度的核心机制"
date: 2025-03-15
categories: [Go, 运行时]
tags: [go, scheduler, gmp]
---

基本思路：

*   work stealing（任务窃取）机制：详解该机制的实现，如何提高cpu利用率的
*   hand off（切换移交）机制：详解该机制的实现，如何避免线程阻塞导致资源限制的
*   协作式抢占：详解gmp模型的调度时机

## 1. 程序的启动

**启动阶段由 `runtime·rt0_go`(核心启动函数) 汇编函数调用**，调用顺序为：

1.  **`runtime·osinit`**：获取系统信息（如CPU核心数）
2.  **`runtime·schedinit`**：初始化调度器。
3.  **`runtime·newproc`**：创建主Goroutine（执行 `runtime.main`）。
4.  **`runtime·mstart`**：启动调度循环。

具体作用在 **Go 调度器(二)** 章中讲过，不再赘述；

> m0 和 g0 是什么：
>
> m0：一个程序会启动多个m，第一个启动的叫m0
>
> g0：每个m创建后都会创建一个g0，用于执行 runtime 下的调度工作

m0的启动是在`runtime·rt0_go`(核心启动函数) 汇编函数函数中执行的(同时也会初始化一个g0和m0绑定)，然后调用mstart->mstart1->schedule启动调度循环；

非m0的启动首先从 startm 方法开始启动，然后从空闲的p链表中获取一个p与m绑定，最后 schedule 启动调度循环；

## 2. work stealing

当p的本地队列为空时，m不会空转，而是从其他p的本地队列中取出带运行的g提供给m执行，最大化提高cpu利用率。

### 2.1 核心源码分析

work stealing是发生在调度循环中的，所以需要分析下调度器是如何进行调度循环的；GMP源码中主要由schedule函数去处理调度器的调度循环，进入到这个方法中里永远不再返回，请看源码分析：

```go
func schedule() {
    mp := getg().m  // 获取当前线程(m)的运行时结构体指针

    if mp.locks != 0 { // 检查持有锁的状态(防止持有锁时触发调度导致死锁)
		throw("schedule: holding locks")
	}

	if mp.lockedg != 0 { // 处理锁定的 Goroutine
		stoplockedm()
		execute(mp.lockedg.ptr(), false)
	}

	if mp.incgo { // 禁止在 CGO 调用上下文中调度
		throw("schedule: in cgo")
	}

    // 上面是在做进入循环调度前的检查
    // 下面开始进入循环调度的入口, 进入后不再返回
top:
    pp := mp.p.ptr() // 获取当前m绑定的p(逻辑处理器)
	pp.preempt = false // 重置 p 的抢占标志位（表示开始新一轮调度）

    // 自旋状态（mp.spinning=true）表示 M 正在跨 P 窃取任务
    // 若本地队列（pp.runnext 或 pp.runq）非空却仍自旋，属逻辑错误
	if mp.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
		throw("schedule: spinning with local work")
	}

    //------------------------------------------------------------核心---------------------------------------------------------
    // 循环调度获取可运行的g
    // 返回值:
    // gp：目标 Goroutine
    // inheritTime：是否继承剩余时间片（避免过度抢占）
    // tryWakeP：是否需要唤醒空闲 p（当窃取到阻塞任务时）
	gp, inheritTime, tryWakeP := findRunnable()
    //-------------------------------------------------------------------------------------------------------------------------
    
    ...

    // 清除 m 的自旋标记并更新全局计数器
	if mp.spinning {
		resetspinning()
	}
    
    ...

    // 当 findRunnable 发现阻塞任务时, 唤醒新线程处理
	if tryWakeP {
		wakep()
	}
	if gp.lockedm != 0 {
		startlockedm(gp)
		goto top
	}

    // 切换到目标 Goroutine 执行用户代码
	execute(gp, inheritTime)
}
```

`findRunnable` 源码解析，`Work-Stealing` 的核心机制在这里实现：

```go
/*
作用: 为当前m找到可运行的 goroutine
主要分为四步:
1. 
*/
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
	mp := getg().m

top:
	pp := mp.p.ptr()
    
    ...

    // 当前p先判断每处理61个任务就去全局队列中获取g, 确保调度的公平
    // 在"sched.runqsize/gomaxprocs + 1"、"max"、"len(_p_.runq))/2"三个数字中取最小的数字作为获取的G数量
	if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
        gp := globrunqget(pp, 1) // 此处是获取1个g
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

	// 从p的本地队列中获取
	if gp, inheritTime := runqget(pp); gp != nil {
		return gp, inheritTime, false
	}

    // 从全局队列获取(此处在"sched.runqsize/gomaxprocs + 1"、"len(_p_.runq))/2"两个数字中取最小的数字作为获取的G数量)
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

    // 从epoll里取
	if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 {
		if list, delta := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			netpollAdjustWaiters(delta)
			trace := traceAcquire()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.ok() {
				trace.GoUnpark(gp, 0)
				traceRelease(trace)
			}
			return gp, false, false
		}
	}

    // work stealing机制的核心代码
	if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() {
		if !mp.spinning {
			mp.becomeSpinning()
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		if gp != nil {
			return gp, inheritTime, false
		}
		if newWork {
			goto top
		}

		now = tnow
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			pollUntil = w
		}
	}
    
    ...
    
    // 没找到就阻塞, 等待唤醒再次执行循环调度
	stopm()
	goto top
}
```

`work stealing` 核心源码解析：

```go
// 参数：now int64：当前时间戳（纳秒），用于计时器检查
// 返回值：
//	gp *g：窃取到的 Goroutine
//  inheritTime bool：是否继承剩余时间片（本函数固定返回 false）
//  rnow/pollUntil int64：更新时间戳和下一个计时器到期时间
//  newWork bool：是否发现新任务（如 GC 任务或触发的计时器）
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
	pp := getg().m.p.ptr()

	ranTimer := false

	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		stealTimersOrRunNextG := i == stealTries-1  // 仅最后一次循环 (i=3) 处理计时器和 runnext 队列，避免高频操作

		for enum := stealOrder.start(cheaprand()); !enum.done(); enum.next() { // heaprand() 生成随机起始点，避免全局竞争热点
			if sched.gcwaiting.Load() {
				return nil, false, now, pollUntil, true
			}
			p2 := allp[enum.position()]
			if pp == p2 { // 跳过自身
				continue
			}
            
            // 最后一次循环还没找到可运行的g, 则从随机到的p中取出一个可运行的g(保底)
			if stealTimersOrRunNextG && timerpMask.read(enum.position()) {
				tnow, w, ran := p2.timers.check(now) // 检查目标p的计时器
				now = tnow
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
                
                 // 计时器已触发, 从p的本地队列中获取可执行的g
				if ran {
					if gp, inheritTime := runqget(pp); gp != nil {
						return gp, inheritTime, now, pollUntil, ranTimer
					}
					ranTimer = true
				}
			}

			if !idlepMask.read(enum.position()) {  // 目标p非空闲
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil { // 尝试窃取目标p本地队列中一半的g
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}
	}

	return nil, false, now, pollUntil, ranTimer
}

// 参数：pp：当前 P（执行窃取的 P）、p2：目标 P（被窃取的 P）、stealRunNextG：是否窃取 p2.runnext（最高优先级任务）
// 返回值：返回窃取的 其中一个 Goroutine 或 nil（失败时）
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {
	t := pp.runqtail // 当前 P 本地队列的 尾部指针（环形缓冲区）（用于后续追加窃取的 Goroutine）
    
    // 窃取p2一半g的核心逻辑：
    // 从p2本地队列获取一半(len(p2.runq)/2), 然后从 pp.runq[t] 开始追加, 返回实际窃取g的数量
	n := runqgrab(p2, &pp.runq, t, stealRunNextG)
    
    // 窃取失败
	if n == 0 {
		return nil
	}
    
    // 返回窃取的最后一个g
	n--
	gp := pp.runq[(t+n)%uint32(len(pp.runq))].ptr()
	if n == 0 {
		return gp
	}
    
    ...
    
	return gp
}
```

### 2.2 work stealing 的调度时机

work stealing 的时机就是调findRunnable的时机，调findRunnable就是调schedule的时机，所以找到所有会调度schedule的时机即可：(应该是以下四个)

1.  `execute` 正常执行完一个g，让出cpu，开启下一轮schedule调度

    execute -> gogo -> 用户代码 -> mcall -> goexist0 -> schedule

2.  `gopark`主动让出cpu (chan阻塞、锁阻塞、sleep等)

    gopark -> mcall -> park\_m -> schedule

    park\_m会将当前g的状态转为waiting，然后接绑和当前m的关系，然后重新开启调度循环为这个m找可运行的g

3.  通过 `sysmon` 抢占让出 cpu

    其实就是 hand off 机制

4.  有系统调用 `Syscall` 则让出 cpu

    进入系统调用前先保存执行现场，然后切换到\_Gsyscall 状态，最后标记抢占，等待被抢占走；

    系统调用退出时，切到 G0 下把G状态切回来，如果有可执行的P则直接执行，如果没有则放到全局队列里，等待调度(schedule )；

### 2.3 为何高效

也是对 work stealing 机制的总结吧，其高效性主要体现在以下两个方面：

*   设计原理层面的高效性
    1.  **低开销的负载均衡**
        *   按需触发：仅当 p 的本地队列为空时，才从其他 p 的队列尾部“窃取”约一半的 g
        *   批量窃取：单次窃取多个 g（最多 50%），减少跨 p 同步频率，避免频繁锁竞争
        *   随机目标：通过随机算法选择被窃取的 p，避免多线程同时窃取同一目标引发的冲突
    2.  **局部性原理**
        *   g 优先放入当前 p 的本地队列，减少全局锁争用
        *   窃取时从其他 p 的队列尾部取 g（最新加入的任务），保留其队列头部的“热任务”，减少缓存失效
    3.  **去中心化**
        *   依赖 P 之间的自主协调，而非全局调度器决策，减少集中式瓶颈
*   提升 CPU 利用率的优化
    1.  **减少空闲等待**
        *   空闲 p 通过窃取快速获取任务，保持 CPU 繁忙
    2.  **抢占与窃取的协同**
        *   运行超 10ms 的 g 被抢占标记，释放 p 给其他任务，空闲 p 窃取时可能获得被抢占的 g，可均衡长/短任务资源
    3.  **适应异构任务负载**
        *   短任务密集：本地队列快速消化，窃取触发少
        *   长任务主导：窃取机制将新任务分流至空闲 p，避免长任务阻塞新任务
        *   突发任务：全局队列作为缓冲，窃取机制加速任务分发

work stealing 使 Go 在维持高并发吞吐的同时，将 CPU 闲置率控制在极低水平，尤其适合 I/O 密集与任务动态生成的场景。

## 3. hand off

hand off机制的主要作用是抢占当前运行的 g，以便将执行权切换给其他等待的 g，这样可以确保系统的资源能够更有效地分配给各个任务，从而提高整体性能；抢占主要分为两种情况：

1.  某个 g 运行时间过长(超过10ms)而发生抢占，为了防止出现饿死的协程。
2.  某个 g 长时间处于系统调用中而发生抢占，为了提高系统的并发性能。

### 3.1 核心源码分析

hand off 的核心逻辑在 retake 函数中实现，来看看 retak 是如何抢占 g 的：

> 源码位置：src/runtime/proc.go 6201

```go
func retake(now int64) uint32 {
	n := 0
	lock(&allpLock)
    // 遍历所有的p
	for i := 0; i < len(allp); i++ {
		pp := allp[i]
		if pp == nil {
			continue
		}
        
		pd := &pp.sysmontick // 用于 sysmon 线程记录被监控 p 的系统调用时间和运行时间
		s := pp.status // p 的状态
		sysretake := false
        
         // 当 p 处于运行或系统调用状态
		if s == _Prunning || s == _Psyscall {
			t := int64(pp.schedtick) // 获取 p 的调度时钟计数，调度一次则 +1
			if int64(pd.schedtick) != t { // 如果系统监控信息中的调度时钟与当前 P 的不一致，则更新系统监控信息
				pd.schedtick = uint32(t)
				pd.schedwhen = now
            } else if pd.schedwhen+forcePreemptNS <= now {  // 距离上次调度的时间已经超过一定阈值(10ms)，则设置抢占标志
                 preemptone(pp) // 只是设置抢占标志这里并不做抢占(具体在g栈扩容时做抢占)
				sysretake = true
			}
		}
        
        // 当 p 处于系统调用状态
		if s == _Psyscall {
			t := int64(pp.syscalltick)
            // 未进行系统抢占 && 系统监控信息中的系统调用时钟与当前 P 的不一致
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}

            // p 的本地运行队列为空
            // 有空闲状态的m 或者 有空闲的 p
            // 距离监控线程记录的系统调用的时间大于10ms
            // 核心含义: 系统不繁忙, 阻塞着也没关系, 不需要抢占去提高性能
			if runqempty(pp) && sched.nmspinning.Load()+sched.npidle.Load() > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}

			unlock(&allpLock)
            
			incidlelocked(-1)
             ...
             // 尝试将 p 状态从 _Psyscall(系统调用) 改为 _Pidle(空闲)
			if atomic.Cas(&pp.status, s, _Pidle) {
                 ...
				n++
				pp.syscalltick++
				handoffp(pp) // 然后找一个新的m接管当前p
			}
             ...
			incidlelocked(1)
			lock(&allpLock)
		}
	}
	unlock(&allpLock)
	return uint32(n) // 返回触发抢占的 P 数量
}
```

以上函数的主要逻辑是：

1.  遍历所有的p，检查有哪些p满足抢占条件，满足的设置抢占；
2.  当p处于运行状态时，检查若同一 goroutine 的运行时间超过了10ms，则对需要抢占，抢占的方式是通过preemptone函数设置抢占标志，待这个g栈扩容时进行抢占；
3.  当p处于阻塞状态时，说明当前p中有g处于系统调用，满足以下三个条件就调用handoff函数寻找一个新的m接管p；
    *   是p 的本地运行队列不空，有g在等待调度
    *   没有自旋的m且没有空闲的p
    *   当前p的系统调用时间过长(超过10ms)
    *   以上这些条件都说明系统繁忙，需要抢占当前p提高系统的并发性能

**handoff(pp)的抢占过程**
handoff函数的作用是为p重新寻找一个m重新开始调度循环；

> 源码位置：runtime/proc.go 3008

```go
func handoffp(pp *p) {
    // 1. p的本地队列非空 或 全局队列大小不为0, 则需要启动一个m来执行任务
	if !runqempty(pp) || sched.runqsize != 0 {
		startm(pp, false, false)
		return
	}

    // 2. 如果追踪已启用或正在关闭，并且追踪读取器可用
	if (traceEnabled() || traceShuttingDown()) && traceReaderAvailable() != nil {
		startm(pp, false, false)
		return
	}

    // 3. 如果垃圾回收的 blacken 模式已启用，并且存在需要标记的工作
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(pp) {
		startm(pp, false, false)
		return
	}

    // 4. 如果没有空闲的m且没有空闲的p, 则尝试将一个m设置成自旋状态
	if sched.nmspinning.Load()+sched.npidle.Load() == 0 && sched.nmspinning.CompareAndSwap(0, 1) { // TODO: fast atomic
		sched.needspinning.Store(0) // 设置无需新增自旋线程, 避免无效线程创建
		startm(pp, true, false)
		return
	}
    
	lock(&sched.lock)
	if sched.gcwaiting.Load() {
		pp.status = _Pgcstop
		pp.gcStopTime = nanotime()
		sched.stopwait--
		if sched.stopwait == 0 {
			notewakeup(&sched.stopnote)
		}
		unlock(&sched.lock)
		return
	}
	if pp.runSafePointFn != 0 && atomic.Cas(&pp.runSafePointFn, 1, 0) {
		sched.safePointFn(pp)
		sched.safePointWait--
		if sched.safePointWait == 0 {
			notewakeup(&sched.safePointNote)
		}
	}
    
    // 5. 如果全局可执行队列不为空, 启动一个 M 来执行任务
	if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}

    // 6. 如果当前空闲的 P 数量为 gomaxprocs-1，并且上次轮询的时间不为零
	if sched.npidle.Load() == gomaxprocs-1 && sched.lastpoll.Load() != 0 {
		unlock(&sched.lock)
		startm(pp, false, false)
		return
	}

	when := pp.timers.wakeTime()
    // 若都不满足绑定条件, 则将p放入空闲队列等待下次自旋的m绑定
	pidleput(pp, 0)
	unlock(&sched.lock)

	if when != 0 {
		wakeNetPoller(when)
	}
}
```

handoff 会对某些条件进行检查，满足某些条件就会调用 startm 函数启动一个新的 m 关联当前 p 来调度执行任务；
以下是六种启动新 m 绑定 p 的场景及其设计原理的详细分析：

1.  **本地或全局队列有等待任务**

```go
if !runqempty(pp) || sched.runqsize != 0 {
	startm(pp, false, false)
	return
}
```

原因：p 的本地运行队列（`runq`）或全局队列（`sched.runqsize`）中有待运行的 g；<br>
意义：避免任务积压，确保就绪的 g 能及时被执行。启动新 m 绑定 p 可立即消费队列中的任务，防止调度延迟；

2.  **程序跟踪或关闭期间需处理跟踪任务**

```go
if (traceEnabled() || traceShuttingDown()) && traceReaderAvailable() != nil {
	startm(pp, false, false)
	return
}
```

原因：程序启用了执行跟踪（`trace.enabled`）或正在关闭（`trace.shutdown`），且存在待处理的跟踪数据（`traceReaderAvailable()`）；<br>
意义：保证跟踪数据的完整收集，尤其在程序关闭时需及时处理残留的跟踪事件，避免数据丢失；

3.  **垃圾回收需要执行标记任务**

```go
if gcBlackenEnabled != 0 && gcMarkWorkAvailable(pp) {
	startm(pp, false, false)
	return
}
```

原因：垃圾收集器处于标记阶段（`gcBlackenEnabled != 0`），且当前 P 有可执行的 GC 标记任务（`gcMarkWorkAvailable(pp)`）；<br>
意义：推进垃圾回收进度。gc 标记任务需并发执行以缩短 STW 时间，启动新 m 确保标记工作不被延迟；

4.  **系统无自旋线程且无空闲p**

```go
if sched.nmspinning.Load()+sched.npidle.Load() == 0 && sched.nmspinning.CompareAndSwap(0, 1) {
	sched.needspinning.Store(0)
	startm(pp, true, false)
	return
}
```

原因：系统中既无自旋 m（`nmspinning=0`），也无空闲 p（`npidle=0`），表明所有 m 均繁忙或无 m 可窃取任务；<br>
意义：**维持系统并发度**。通过 CAS 操作设置 `nmspinning=1` 并启动一个自旋 m（`spinning=true`），该 m 将主动窃取其他 p 的任务，防止任务因无 m 执行而饥饿；

5.  **全局队列有任务但当前 p 无任务**

```go
if sched.runqsize != 0 {
	unlock(&sched.lock)
        startm(pp, false, false)
	return
}
```

原因：全局运行队列（`sched.runqsize`）不为空，但当前 p 的本地队列为空；<br>
意义：解决**任务分配不均**问题。全局队列中的 g 可能来自阻塞恢复或新建的跨 p 任务，启动新 m 可避免全局队列任务因无 p 处理而饥饿；

6.  **几乎所有的p都空闲(gomaxprocs-1)，且曾有网络事件**

```go
if sched.npidle.Load() == gomaxprocs-1 && sched.lastpoll.Load() != 0 {
	unlock(&sched.lock)
	startm(pp, false, false)
	return
}
```

原因：系统中仅剩 1 个 p 忙碌（`npidle = GOMAXPROCS-1`），且曾有过网络轮询（`sched.lastpoll != 0`，记录上次轮询时间）；<br>
意义：针对**网络密集型应用**的优化。当几乎所有 p 空闲时，可能存在未处理的网络事件（如 socket 就绪），启动新 m 可及时处理这些事件，避免网络延迟；

### 3.2 总结

*   避免线程闲置(资源复用)：m因g阻塞时会将m和当前p剥离开，新建一个m或取闲置的m去管理阻塞的g...
*   动态负载均衡(任务无缝衔接)：阻塞 m 释放 p 后，调度器立刻为 p 分配一个新的 m(空闲队列或者新建)，确保 p 本地队列的任务不中断；
*   降低系统调用开销(阻塞优化)：系统调用仅阻塞单个 m, 不影响其他 m 的调度执行；
*   提升吞吐(并行扩展)：和work stealing机制联动，m 阻塞时, p 被转移(hand off)；m 空闲时主动窃取其他 p 的任务(work stealing)；
*   避免资源浪费饥饿问题(公平性)：若 m 长时间阻塞（如系统调用超时），监控线程（`sysmon`）强制剥离 p，防止任务饥饿。

<br>

## 4. 调度流程图

go func 的大致调度流程图：

![image-20260324171358801](/assets/img/posts/image-20260324171358801.png)