## 一、协程的本质

协程在go语言中的表示，就是g结构体。其中，比较重要的是stack：堆栈地址；gobuf：目前协程运行的现场；atomicstatus：协程状态。同时协程需要基于线程，因此go对操作系统线程信息描述抽象为结构体m。

```go
type g struct {
   // Stack parameters.
   // stack describes the actual stack memory: [stack.lo, stack.hi).
   // stackguard0 is the stack pointer compared in the Go stack growth prologue.
   // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
   // stackguard1 is the stack pointer compared in the C stack growth prologue.
   // It is stack.lo+StackGuard on g0 and gsignal stacks.
   // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
   stack       stack   // offset known to runtime/cgo
   stackguard0 uintptr // offset known to liblink
   stackguard1 uintptr // offset known to liblink

   _panic    *_panic // innermost panic - offset known to liblink
   _defer    *_defer // innermost defer
   m         *m      // current m; offset known to arm liblink
   sched     gobuf
   syscallsp uintptr // if status==Gsyscall, syscallsp = sched.sp to use during gc
   syscallpc uintptr // if status==Gsyscall, syscallpc = sched.pc to use during gc
   stktopsp  uintptr // expected sp at top of stack, to check in traceback
   // param is a generic pointer parameter field used to pass
   // values in particular contexts where other storage for the
   // parameter would be difficult to find. It is currently used
   // in three ways:
   // 1. When a channel operation wakes up a blocked goroutine, it sets param to
   //    point to the sudog of the completed blocking operation.
   // 2. By gcAssistAlloc1 to signal back to its caller that the goroutine completed
   //    the GC cycle. It is unsafe to do so in any other way, because the goroutine's
   //    stack may have moved in the meantime.
   // 3. By debugCallWrap to pass parameters to a new goroutine because allocating a
   //    closure in the runtime is forbidden.
   param        unsafe.Pointer
   atomicstatus uint32
   stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
   goid         int64
   schedlink    guintptr
   waitsince    int64      // approx time when the g become blocked
   waitreason   waitReason // if status==Gwaiting

   preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
   preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
   preemptShrink bool // shrink stack at synchronous safe point

   // asyncSafePoint is set if g is stopped at an asynchronous
   // safe point. This means there are frames on the stack
   // without precise pointer information.
   asyncSafePoint bool

   paniconfault bool // panic (instead of crash) on unexpected fault address
   gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
   throwsplit   bool // must not split stack
   // activeStackChans indicates that there are unlocked channels
   // pointing into this goroutine's stack. If true, stack
   // copying needs to acquire channel locks to protect these
   // areas of the stack.
   activeStackChans bool
   // parkingOnChan indicates that the goroutine is about to
   // park on a chansend or chanrecv. Used to signal an unsafe point
   // for stack shrinking. It's a boolean value, but is updated atomically.
   parkingOnChan uint8

   raceignore     int8     // ignore race detection events
   sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
   tracking       bool     // whether we're tracking this G for sched latency statistics
   trackingSeq    uint8    // used to decide whether to track this G
   runnableStamp  int64    // timestamp of when the G last became runnable, only used when tracking
   runnableTime   int64    // the amount of time spent runnable, cleared when running, only used when tracking
   sysexitticks   int64    // cputicks when syscall has returned (for tracing)
   traceseq       uint64   // trace event sequencer
   tracelastp     puintptr // last P emitted an event for this goroutine
   lockedm        muintptr
   sig            uint32
   writebuf       []byte
   sigcode0       uintptr
   sigcode1       uintptr
   sigpc          uintptr
   gopc           uintptr         // pc of go statement that created this goroutine
   ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
   startpc        uintptr         // pc of goroutine function
   racectx        uintptr
   waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
   cgoCtxt        []uintptr      // cgo traceback context
   labels         unsafe.Pointer // profiler labels
   timer          *timer         // cached timer for time.Sleep
   selectDone     uint32         // are we participating in a select and did someone win the race?

   // goroutineProfiled indicates the status of this goroutine's stack for the
   // current in-progress goroutine profile
   goroutineProfiled goroutineProfileStateHolder

   // Per-G GC state

   // gcAssistBytes is this G's GC assist credit in terms of
   // bytes allocated. If this is positive, then the G has credit
   // to allocate gcAssistBytes bytes without assisting. If this
   // is negative, then the G must correct this by performing
   // scan work. We track this in bytes to make it fast to update
   // and check for debt in the malloc hot path. The assist ratio
   // determines how this corresponds to scan work debt.
   gcAssistBytes int64
}
```

![gorun_struct](\blog\picture\gorun_struct.png)

## 二、P结构体

G：协程

M：线程

P：本地队列

![GMP](\blog\picture\GMP.png)

## 三、GMP模型

![GMP模型](\blog\picture\GMP模型.png)

P是M与G之间的中介，P持有一些G，使得每次获取G的时候不用从全局找，大大减少了并发冲突的情况。如果本地也没有G，全局也没有G，那么可以使用窃取式工作分配机制，从其他P中进行窃取。

## 四、Go的锁

sync.Mutex：互斥锁

sync.RWMute：读写锁

sync.WaitGroup：等待组

sync.Once：初始化

### 1. 原子操作

原子操作是一种硬件层面枷锁的机制；保证操作一个变量的时候，其他协程/线程无法访问；只能用于简单变量的简单操作。

## 2. sema锁

也叫信号量锁/信号锁，核心是一个uint32的值，含义是同时可并发的数量，每一个sema锁对应一个SemaRoot结构体。SemaRoot中有一个平衡二叉树用于协程排队。