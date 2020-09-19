## ***为什么需要读写锁?***

与传统锁不同的是读写锁的规则是可以共享读，但只能一个写，总结起来为：*`读读不互斥，读写互斥，写写互斥`*，而一般的独占锁是：*`读读互斥，读写互斥，写写互斥`*，而场景中往往**读远远大于写**，读写锁就是为了这种优化而创建出来的一种机制。

注意是*`读远远大于写`*，一般情况下独占锁的效率低来源于高并发下对临界区的激烈竞争导致线程上下文切换。因此当并发不是很高的情况下，读写锁由于需要额外维护读锁的状态，可能还不如独占锁的效率高。因此需要根据实际情况选择使用。

## *ReadWriteLock的实现原理*

在Java中`ReadWriteLock`的主要实现为`ReentrantReadWriteLock`，其提供了以下特性：

- **公平性选择**：支持公平与非公平（默认）的锁获取方式，吞吐量非公平优先于公平。

- **可重入**：读线程获取读锁之后可以再次获取读锁，写线程获取写锁之后可以再次获取写锁

- **可降级**：写线程获取写锁之后，其还可以再次获取读锁，然后释放掉写锁，那么此时该线程是读锁状态，也就是降级操作

## ReentrantReadWriteLock的结构

`ReentrantReadWriteLock`的核心是由一个基于AQS的同步器`Sync`构成，然后由其扩展出`ReadLock`（共享锁），`WriteLock`（排它锁）所组成。

![640](../../图片/java/reentrantlockreadwritelock_struct.png)

并且从`ReentrantReadWriteLock`的构造函数中可以发现`ReadLock`与`WriteLock`使用的是同一个Sync，具体怎么实现同一个队列既可以为共享锁，又可以表示排他锁下文会具体分析。

**ReentrantReadWriteLock构造函数**

```java
public ReentrantReadWriteLock(boolean fair) {
       sync = fair ? new FairSync() : new NonfairSync();
       readerLock = new ReadLock(this);
       writerLock = new WriteLock(this);
   }
```



## *Sync的实现*

`sync`是读写锁实现的核心，`sync`是基于AQS实现的，在AQS中核心是state字段和双端队列，那么一个一个问题来分析。

## ***Sync如何同时表示读锁与写锁？***

**读写锁状态获取**

```java
static final int SHARED_SHIFT = 16;
static final int SHARED_UNIT = (1 << SHARED_SHIFT);
static final int MAX_COUNT = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** Returns the number of shared holds represented in count */
static int sharedCount(int c) { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

从代码中获取读写状态可以看出其是把`state（int32位）`字段分成高16位与低16位，其中高16位表示读锁个数，低16位表示写锁个数，如下图所示（图来自Java并发编程艺术）。

![640](../../图片/java/reentrantlockreadwritelock_state.png)

该图表示当前一个线程获取到了写锁，并且重入了两次，因此低16位是3，并且该线程又获取了读锁，并且重入了一次，所以高16位是2，当写锁被获取时如果读锁不为0那么读锁一定是获取写锁的这个线程。

## *读锁的获取*

读锁的获取主要实现是AQS中的`acquireShared`方法，其调用过程如下代码。

**读锁获取入口**

```java
// ReadLock
public void lock() {
    sync.acquireShared(1);
}
// AQS
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

其中`doAcquireShared(arg)`方法是获取失败之后AQS中入队操作，等待被唤醒后重新获取，那么关键点就是`tryAcquireShared(arg)`方法，方法有点长，因此先总结出获取读锁所经历的步骤，获取的第一部分步骤如下：

- 操作1：读写需要互斥，因此当存在写锁并且持有写锁的线程不是该线程时获取失败。
- 操作2：是否存在等待写锁的线程，存在的话则获取读锁需要等待，避免写锁饥饿。(写锁优先级是比较高的)
- 操作3：CAS获取读锁，实际上是state字段的高16位自增。
- 操作4：获取成功后再ThreadLocal中记录当前线程获取读锁的次数。

**读锁获取的第一部分**

```java
protected final int tryAcquireShared(int unused) {
          Thread current = Thread.currentThread();
          int c = getState();
          // 操作1：存在写锁，并且写锁不是当前线程则直接去排队
          if (exclusiveCount(c) != 0 &&
              getExclusiveOwnerThread() != current)
              return -1;

          int r = sharedCount(c);
          // 操作2：读锁是否该阻塞，对于非公平模式下写锁获取优先级会高，如果存在要获取写锁的线程则读锁需要让步，公平模式下则先来先到
          if (!readerShouldBlock() && 
              // 读锁使用高16位，因此存在获取上限为2^16-1
              r < MAX_COUNT &&
              // 操作3：CAS修改读锁状态，实际上是读锁状态+1
              compareAndSetState(c, c + SHARED_UNIT)) {
              // 操作4：执行到这里说明读锁已经获取成功，因此需要记录线程状态。
              if (r == 0) {
                  firstReader = current; // firstReader是把读锁状态从0变成1的那个线程
                  firstReaderHoldCount = 1;
              } else if (firstReader == current) { 
                  firstReaderHoldCount++;
              } else {
                  // 这些代码实际上是从ThreadLocal中获取当前线程重入读锁的次数，然后自增下。
                  HoldCounter rh = cachedHoldCounter; // cachedHoldCounter是上一个获取锁成功的线程
                  if (rh == null || rh.tid != getThreadId(current))
                      cachedHoldCounter = rh = readHolds.get();
                  else if (rh.count == 0)
                      readHolds.set(rh);
                  rh.count++;
              }
              return 1;
          }
          // 当操作2，操作3失败时执行该逻辑
          return fullTryAcquireShared(current);
      }
```

当操作2，操作3失败时会执行`fullTryAcquireShared(current)`，为什么会这样写呢？个人认为是一种补偿操作，**操作2与操作3失败并不代表当前线程没有读锁的资格**，并且这里的读锁是共享锁，有资格就应该被获取成功，因此给予补偿获取读锁的操作。在`fullTryAcquireShared(current)`中是一个循环获取读锁的过程，大致步骤如下：

- 操作5：等同于操作2，存在写锁，且写锁线程并非当前线程则直接返回失败
- 操作6：当前线程是重入读锁，这里只会偏向第一个获取读锁的线程以及最后一个获取读锁的线程，其他都需要去AQS中排队。
- 操作7：CAS改变读锁状态
- 操作8：同操作4，获取成功后再ThreadLocal中记录当前线程获取读锁的次数。

**读锁获取的第二部分**

```java
final int fullTryAcquireShared(Thread current) {
           HoldCounter rh = null;
           // 最外层嵌套循环
           for (;;) {
               int c = getState();
               // 操作5：存在写锁，且写锁并非当前线程则直接返回失败
               if (exclusiveCount(c) != 0) {
                   if (getExclusiveOwnerThread() != current)
                       return -1;
                   // else we hold the exclusive lock; blocking here
                   // would cause deadlock.
               // 操作6：如果当前线程是重入读锁则放行
               } else if (readerShouldBlock()) {
                   // Make sure we're not acquiring read lock reentrantly
                   // 当前是firstReader，则直接放行,说明是已获取的线程重入读锁
                   if (firstReader == current) {
                       // assert firstReaderHoldCount > 0;
                   } else {
                       // 执行到这里说明是其他线程，如果是cachedHoldCounter（其count不为0）也就是上一个获取锁的线程则可以重入，否则进入AQS中排队
                       // **这里也是对写锁的让步**，如果队列中头结点为写锁，那么当前获取读锁的线程要进入队列中排队
                       if (rh == null) {
                           rh = cachedHoldCounter;
                           if (rh == null || rh.tid != getThreadId(current)) {
                               rh = readHolds.get();
                               if (rh.count == 0)
                                   readHolds.remove();
                           }
                       }
                       // 说明是上述刚初始化的rh，所以直接去AQS中排队
                       if (rh.count == 0)
                           return -1;
                   }
               }
               if (sharedCount(c) == MAX_COUNT)
                   throw new Error("Maximum lock count exceeded");
               // 操作7：修改读锁状态，实际上读锁自增操作
               if (compareAndSetState(c, c + SHARED_UNIT)) {
                   // 操作8：对ThreadLocal中维护的获取锁次数进行更新。
                   if (sharedCount(c) == 0) {
                       firstReader = current;
                       firstReaderHoldCount = 1;
                   } else if (firstReader == current) {
                       firstReaderHoldCount++;
                   } else {
                       if (rh == null)
                           rh = cachedHoldCounter;
                       if (rh == null || rh.tid != getThreadId(current))
                           rh = readHolds.get();
                       else if (rh.count == 0)
                           readHolds.set(rh);
                       rh.count++;
                       cachedHoldCounter = rh; // cache for release
                   }
                   return 1;
               }
           }
       }
```



## ***读锁的释放***

**读锁释放入口**

```java
// ReadLock
public void unlock() {
    sync.releaseShared(1);
}
// Sync
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared(); // 这里实际上是释放读锁后唤醒写锁的线程操作
        return true;
    }
    return false;
}
```

读锁的释放主要是`tryReleaseShared(arg)`函数，因此拆解其步骤如下：

- 操作1：清理ThreadLocal中保存的获取锁数量信息
- 操作2：CAS修改读锁个数，实际上是自减一

**读锁的释放流程**

```java
protected final boolean tryReleaseShared(int unused) {
         Thread current = Thread.currentThread();
         // 操作1：清理ThreadLocal对应的信息
         if (firstReader == current) {;
             if (firstReaderHoldCount == 1)
                 firstReader = null;
             else
                 firstReaderHoldCount--;
         } else {
             HoldCounter rh = cachedHoldCounter;
             if (rh == null || rh.tid != getThreadId(current))
                 rh = readHolds.get();
             int count = rh.count;
             // 已释放完的读锁的线程清空操作
             if (count <= 1) {
                 readHolds.remove();
                 // 如果没有获取锁却释放则会报该错误
                 if (count <= 0)
                     throw unmatchedUnlockException();
             }
             --rh.count;
         }
         // 操作2：循环中利用CAS修改读锁状态
         for (;;) {
             int c = getState();
             int nextc = c - SHARED_UNIT;
             if (compareAndSetState(c, nextc))
                 // Releasing the read lock has no effect on readers,
                 // but it may allow waiting writers to proceed if
                 // both read and write locks are now free.
                 return nextc == 0;
         }
     }
```

## ***写锁的获取***

**写锁的获取入口**

```java
// WriteLock
  public void lock() {
        sync.acquire(1);
    }
// AQS
  public final void acquire(int arg) {
        // 尝试获取，获取失败后入队，入队失败则interrupt当前线程
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

写锁的获取也主要是`tryAcquire(arg)`方法，这里也拆解步骤：

- 操作1：如果读锁数量不为0或者写锁数量不为0，并且不是重入操作，则获取失败。
- 操作2：如果当前锁的数量为0，也就是不存在操作1的情况，那么该线程是有资格获取到写锁，因此修改状态，设置独占线程为当前线程

**清单9：写锁的获取**

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    // 操作1：c != 0，说明存在读锁或者写锁
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)  
        // 写锁为0，读锁不为0 或者获取写锁的线程并不是当前线程，直接失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // 执行到这里说明是写锁线程的重入操作，直接修改状态，也不需要CAS因为没有竞争
        setState(c + acquires);
        return true;
    }
    // 操作2：获取写锁，writerShouldBlock对于非公平模式直接返回fasle，对于公平模式则线程需要排队，因此需要阻塞。
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

## ***写锁的释放***

#### **写锁的释放入口**

```java
// WriteLock
public void unlock() {
        sync.release(1);
    }
// AQS
public final boolean release(int arg) {
    // 释放锁成功后唤醒队列中第一个线程
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

写锁的释放主要是`tryRelease(arg)`方法，其逻辑就比较简单了，注释很详细。

**写锁的释放**

```java
protected final boolean tryRelease(int releases) {
     // 如果当前线程没有获取写锁却释放，则直接抛异常
     if (!isHeldExclusively())
         throw new IllegalMonitorStateException();
     // 状态变更至nextc
     int nextc = getState() - releases;
     // 因为写锁是可以重入，所以在都释放完毕后要把独占标识清空
     boolean free = exclusiveCount(nextc) == 0;
     if (free)
         setExclusiveOwnerThread(null);
     // 修改状态
     setState(nextc);
     return free;
 }
```

## ***锁降级操作哪里体现？***

锁降级操作指的是一个线程获取写锁之后再获取读锁，然后读锁释放掉写锁的过程。在`tryAcquireShared(arg)`获取读锁的代码中有如下代码。

**写锁降级策略**

```
Thread current = Thread.currentThread();
            // 当前状态
            int c = getState();
            // 存在写锁，并且写锁不等于当前线程时返回，换句话说等写锁为当前线程时则可以继续往下获取读锁。
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
。。。。。读锁获取。。。。。
```

那么锁降级有什么用？答案是为了可见性的保证。在`ReentrantReadWriteLock`的javadoc中有如下代码，其是锁降级的一个应用示例**（个人觉得是为了活跃性，避免死锁）**。

```java
class CachedData {
  Object data;
  volatile boolean cacheValid;
  final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

  void processCachedData() {
    // 获取读锁
    rwl.readLock().lock();
    if (!cacheValid) {
      // Must release read lock before acquiring write lock，不释放的话下面写锁会获取不成功，造成死锁
      rwl.readLock().unlock();
     // 获取写锁
      rwl.writeLock().lock();
      try {
        // Recheck state because another thread might have
        // acquired write lock and changed state before we did.
        if (!cacheValid) {
          data = ...
          cacheValid = true;
        }
        // Downgrade by acquiring read lock before releasing write lock
        // 这里再次获取读锁，如果不获取那么当写锁释放后可能其他写线程再次获得写锁，导致下方`use(data)`时出现不一致的现象
        // 这个操作就是降级
        rwl.readLock().lock();
      } finally {
        rwl.writeLock().unlock(); // Unlock write, still hold read
      }
    }

    try {
    // 使用完后释放读锁
      use(data);
    } finally {
      rwl.readLock().unlock();
    }
  }
 }}
```

## ***公平与非公平的区别***

**公平下的Sync**

```java
static final class FairSync extends Sync {
     private static final long serialVersionUID = -2274990926593161451L;
     final boolean writerShouldBlock() {
         return hasQueuedPredecessors(); // 队列中是否有元素，有责当前操作需要block
     }
     final boolean readerShouldBlock() {
         return hasQueuedPredecessors();// 队列中是否有元素，有责当前操作需要block
     }
 }
```

公平下的Sync实现策略是所有获取的读锁或者写锁的线程都需要入队排队，按照顺序依次去尝试获取锁。

**清单14：非公平下的Sync**

```java
static final class NonfairSync extends Sync {
       private static final long serialVersionUID = -8159625535654395037L;
       final boolean writerShouldBlock() {
           // 非公平下不考虑排队，因此写锁可以竞争获取
           return false; // writers can always barge
       }
       final boolean readerShouldBlock() {
           /* As a heuristic to avoid indefinite writer starvation,
            * block if the thread that momentarily appears to be head
            * of queue, if one exists, is a waiting writer.  This is
            * only a probabilistic effect since a new reader will not
            * block if there is a waiting writer behind other enabled
            * readers that have not yet drained from the queue.
            */
           // 这里实际上是一个优先级，如果队列中头部元素时写锁，那么读锁需要等待，避免写锁饥饿。
           return apparentlyFirstQueuedIsExclusive();
       }
   }
```

非公平下由于抢占式获取锁，写锁是可能产生饥饿，因此解决办法就是提高写锁的优先级，换句话说获取写锁之前先占坑