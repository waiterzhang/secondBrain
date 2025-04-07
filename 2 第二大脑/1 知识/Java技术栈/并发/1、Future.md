# 1、Future

## 学习目标

> from GPT：
>
> 学习 `FutureTask`​ 类时，你应该关注以下核心概念和实现：
>
> 1. ​`Future`​ 接口；
> 2. ​`Callable`​ 接口；
> 3. 异步任务执行；
> 4. 状态管理；
> 5. 回调和通知；
> 6. 并发操作和线程安全性；

## 总结

* 通过一组生命周期状态来管理任务的执行状态；
* 重点关注get方法的阻塞，阻塞主要有由awaitDone方法实现。该方法的核心点在于：循环检查任务执行状态，如果任务处于进行中，则让出cpu时间片等待下一次检查，直到任务执行成功。可见，这个轮询cpu的方式必然造成其性能的浪费。这也是future这个方法的主要缺点。
* 异常会通过setException方法来包装。
* 次要关注一些对于并发的处理。

  * 其中对于当前状态、节点的切换，大量使用了cas操作。
  * 维持了一个waiterNode的待执行队列，来管理多个任务。

## 概念

> A Future represents the result of an asynchronous computation. Methods are provided to check if the computation is complete, to wait for its completion, and to retrieve the result of the computation. The result can only be retrieved using method get when the computation has completed, blocking if necessary until it is ready. Cancellation is performed by the cancel method. Additional methods are provided to determine if the task completed normally or was cancelled. Once a computation has completed, the computation cannot be cancelled. If you would like to use a Future for the sake of cancellability but not provide a usable result, you can declare types of the form Future<?> and return null as a result of the underlying task.
>
> The [`FutureTask`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/FutureTask.html "class in java.util.concurrent")​ class is an implementation of `Future`​ that implements `Runnable`​, and so may be executed by an `Executor`​.

* 这个Future表示一个异步计算的结果；
* 当计算完成，使用get方法获取结果，如果还没有完成，则会一直阻塞到其成功；
* cancel可以用来取消这个异步计算，而一旦计算完成，则不可取消。如果已经取消了再去get，会抛出异常。

Future只是一个接口，而FutureTask是其中的核心实现类。下面将通过Futuretask来描述具体的实现。

Future的概念：

> A cancellable asynchronous computation. This class provides a base implementation of [`Future`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html "interface in java.util.concurrent")​, with methods to start and cancel a computation, query to see if the computation is complete, and retrieve the result of the computation. The result can only be retrieved when the computation has completed; the `get`​ methods will block if the computation has not yet completed. Once the computation has completed, the computation cannot be restarted or cancelled (unless the computation is invoked using [`runAndReset()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/FutureTask.html#runAndReset--)​).A `FutureTask`​ can be used to wrap a [`Callable`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Callable.html "interface in java.util.concurrent")​ or [`Runnable`](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html "interface in java.lang")​ object. Because `FutureTask`​ implements `Runnable`​, a `FutureTask`​ can be submitted to an [`Executor`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html "interface in java.util.concurrent")​ for execution.

* 该类符合Future对于Get和Cancel的定义：get获取结果会导致阻塞、一旦计算完成无法取消或者重新计算。
* 这个类可以被包装为Callable或者Runnable的接口，因为他实现了这两个接口。所以它可以被提交给一个Executor来执行。

## 源码

任务执行有其生命周期，跟踪计算任务的生命周期是FutureTask的重要一环。

```java
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

在类中定义了一组描述生命周期的变量。可能的状态转移仅有四种：

```java
  Possible state transitions:
     * NEW -> COMPLETING -> NORMAL // 正常完成
     * NEW -> COMPLETING -> EXCEPTIONAL // 异常
     * NEW -> CANCELLED // 取消
     * NEW -> INTERRUPTING -> INTERRUPTED // 中断
```

下面来看一下生命周期感知的两个方法：

```java
    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    public boolean isDone() {
        return state != NEW;
    }
```

* 如果在CANCELLED、INTERRUPTING、INTERRUPTED状态下均为取消。注意EXCEPTIONAL异常状态下，不视为“已取消”。
* isDown不是代表已完成吗？可是非new状态下还有很多其他状态呀，看起来并非已完成。

看一下get方法：

```java
    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```

在COMPLETING及以下都属于未完成状态，重点看awaitDone方法：

```java
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }
```

* 如果线程中断，则将这个节点移除。removeWaiter(q)，通过遍历列表的形式，将整个等待队列上引用为null的thread从等待队列中移除。

  > 线程中断和cpu分片被剥夺是两个不同维度的概念，此处的线程中断指的是手动调用`interrupt()`​方法来进行中断。在具体业务上来讲，该中断方法会在cancel中调用。因此可以理解，当cancel的时候，该线程中断，并将其在等待执行的队列移除合情合理。
  >

* 检查状态

  * 如果状态超过COMPLETING，则可以视为已完成。
  * 如果正在COMPLETING，则让出Cpu时间片进行等待。

    > ​`Thread.yield()`​ 是一个静态方法，它是 `Thread`​ 类提供的一个简单的线程调度器提示方法。调用 `Thread.yield()`​ 的作用是给予其他具有相同优先级的线程执行的机会，即让出当前线程的CPU执行时间片。
    >
    > 这是一个标志，不代表立刻就会让出去时间片。
    >
  * 将当前节点加入等待队列。

总结一下awaitDone方法：

* 如果已经取消，则不再等待；
* 已经完成，则结束等待。
* 还未完成则让出cpu来等待。
* 有一个等待队列来控制各个节点。

从这个awaitDone方法出去，代表已经完成。下面再来看看`report`​方法，这个方法非常的简单：

```java
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```

如果处于正常状态则直接返回value，如果超过CANCELLED则直接抛出异常。

所以在此可以总结一下Get方法：

* 等待任务完成。
* 返回值，有异常抛出异常。

下面来看一下cancel方法：

```java
 public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```

* 当任务未开始 且 成功中断/取消的时候，则意味着取消成功。
* mayInterruptIfRunning这个布尔值，代表在任务执行过程中是否可以进行中断。如果可以进行中断，在通过原子操作将其设置为中断状态。

下面来看一下`finishCompletion`​这个函数：

```java
 /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```

> UNSAFE.compareAndSwapObject四个参数的解释说明：
>
> 1、被执行对象；2、偏移量（要检查的位置）3、预期的值 4、新值
>
> 如果该对象的指定位置的值与预期相符，则将其更新为新值。

* 将所有已经完成的线程，从等待队列中移除。未完成的节点将通过unpark进行唤醒。为什么首先将头节点置null，又挨个将已完成的节点置null，我猜可能是因为尽快让垃圾回收器回收？

执行完还有一个done方法，他什么也不做。如果需要补充done，可以进行重写。

最后是将callable = null;显式的让垃圾回收器回收。

‍

下面再看一下run方法：

```java
 public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

* 先看状态判定：只有生命周期为new的情况下 && 当前线程拿到执行权限，才执行；
* callable是任务执行逻辑，这个在构造器里面会初始化callable
* 出现异常，异常会被setExp
* ​`handlePossibleCancellationInterrupt`​，如果执行说明发生了中断。来看一下这个中断方法：

  ```undefined
   private void handlePossibleCancellationInterrupt(int s) {
          // It is possible for our interrupter to stall before getting a
          // chance to interrupt us.  Let's spin-wait patiently.
          if (s == INTERRUPTING)
              while (state == INTERRUPTING)
                  Thread.yield(); // wait out pending interrupt
      }
  ```
  注释写道：在任务执行期间，确保任何有取消导致的中断，仅被传递到一个任务中。

  看代码逻辑，发现当发现中断时，会不断通过自旋来保证中断结束。这里应该是为了避免并发问题。
