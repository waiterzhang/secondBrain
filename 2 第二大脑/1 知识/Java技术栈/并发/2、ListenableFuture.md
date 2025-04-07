# 2、ListenableFuture

## 学习目标

> 学习ListenableFuture的源码实现，如果只能关注五个重点，应该是哪些？
>
> from me:
>
> 1、前面学习的FutureTask通过阻塞的方式进行get这样会浪费性能，那ListenableFuture是如何不使用轮询的情况下，得知任务已经完成了呢？
>
> from GPT：
>
> 1、接口定义： 首先了解ListenableFuture接口本身，特别是它如何扩展Future接口，以及它新增了哪些方法来支持监听器的添加。
>
> 2、回调机制与监听器执行： 理解addListener方法的内部工作原理，包括监听器是如何被存储、管理和在Future完成时被触发的。这是ListenableFuture区别于普通Future的核心特性。  
> 3、AbstractFuture类： AbstractFuture是Guava中实现ListenableFuture的核心抽象类。它提供了大部分基础功能的实现，包括监听器的添加和通知逻辑。深入研究这个类将帮助你理解ListenableFuture的大部分实现细节。  
> 4、线程安全性与并发控制： 由于ListenableFuture可能被多个线程访问，因此它必须是线程安全的。关注源码中是如何处理同步和并发问题的，例如使用锁（synchronized blocks）或原子变量（AtomicReference）来保护状态。  
> 5、异常处理与取消操作： 异常处理和取消操作是异步编程中的重要方面。关注当计算失败时如何传递异常给监听器，以及取消操作如何影响正在进行中或尚未开始的计算和监听器的执行

## 总结

* ListenableFuture提供了增加监听器的机制。AbstrackFuture是其主要实现，所有的监听器以Waiter的形势以等待队列进行存储。其中AbstrackFuture的complete方法是其中的核心方法，它在任务进行终态以后会触发所有的监听器。
* ListenableFuture中出现的异常，会将其进行包裹在成员变量中。
* ListenableFuture中监听器的具体触发时机，实际上实现于Exectors里面。其中会将计算任务和回调逻辑包裹一起串行执行，计算完毕之后调用AbstrackFuture的set方法激活所有的回调逻辑。

## 概念

增加了回调机制的Future。

javadoc：

> A Future that accepts completion listeners. Each listener has an associated executor, and it is invoked using this executor once the future's computation is complete. If the computation has already completed when the listener is added, the listener will execute immediately.  
> See the Guava User Guide article on ListenableFuture.  
> Purpose  
> The main purpose of ListenableFuture is to help you chain together a graph of asynchronous operations. You can chain them together manually with calls to methods like Futures.transform, but you will often find it easier to use a framework. Frameworks automate the process, often adding features like monitoring, debugging, and cancellation. Examples of frameworks include:  
> Dagger Producers  
> The main purpose of addListener is to support this chaining. You will rarely use it directly, in part because it does not provide direct access to the Future result. (If you want such access, you may prefer Futures.addCallback.) Still, direct addListener calls are occasionally useful:

* 这个Future可以接受一个监听器，每个监听器都会有一个关联的执行器。在Future完成之后该执行器会调用监听器。如果添加监听器时任务已经完成，则会立即调用该监听器。

该接口有且仅有一个方法：

​`void addListener(Runnable listener, Executor executor);`​

> Registers a listener to be run on the given executor. The listener will run when the Future's computation is complete or, if the computation is already complete, immediately.  
> There is no guaranteed ordering of execution of listeners, but any listener added through this method is guaranteed to be called once the computation is complete.  
> Exceptions thrown by a listener will be propagated up to the executor. Any exception thrown during Executor.execute (e.g., a RejectedExecutionException or an exception thrown by direct execution) will be caught and logged.  
> Note: For fast, lightweight listeners that would be safe to execute in any thread, consider MoreExecutors.directExecutor. Otherwise, avoid it. Heavyweight directExecutor listeners can cause problems, and these problems can be difficult to reproduce because they depend on timing. For example:  
> The listener may be executed by the caller of addListener. That caller may be a UI thread or other latency-sensitive thread. This can harm UI responsiveness.  
> The listener may be executed by the thread that completes this Future. That thread may be an internal system thread such as an RPC network thread. Blocking that thread may stall progress of the whole system. It may even cause a deadlock.  
> The listener may delay other listeners, even listeners that are not themselves directExecutor listeners.  
> This is the most general listener interface. For common operations performed using listeners, see Futures. For a simplified but general listener interface, see addCallback().  
> Memory consistency effects: Actions in a thread prior to adding a listener happen-before its execution begins, perhaps in another thread.

* 监听器的顺序不保证；
* 重要：监听器的异常将传播到执行器。
* 选择执行监听器的执行器很重要（executor）。轻量级的监听器直接执行可以避免线程切换开销，而重量级的监听器可能导致主线程被阻塞。

AbstractFuture是`ListenableFuture`​的核心实现。下面来看一下这个抽象类的实现细节。

## AbstractFuture

**生命周期管理**

Future中通过一组数字来进行任务生命周期管理，AbstractFuture也需要进行生命周期管理。两者看起来大同小异。来看一下AbstractFuture表征生命周期的代码：

```java
  private static final Object NULL = new Object();

  /** A special value to represent failure, when {@link #setException} is called successfully. */
  private static final class Failure {
   	···
    Failure(Throwable exception) {
      this.exception = checkNotNull(exception);
    }
  }

  /** A special value to represent cancellation and the 'wasInterrupted' bit. */
  private static final class Cancellation {
    ···
    Cancellation(boolean wasInterrupted, @Nullable Throwable cause) {
      this.wasInterrupted = wasInterrupted;
      this.cause = cause;
    }
  }

  /** A special value that encodes the 'setFuture' state. */
  private static final class SetFuture<V> implements Runnable {
    SetFuture(AbstractFuture<V> owner, ListenableFuture<? extends V> future) {
      this.owner = owner;
      this.future = future;
    }
   ···
  }
```

在AbstractFuture中一共有四种生命周期的状态，可以分为中间状态和终态：

中间状态：

* setFuture：标识已经设置了Future，但是还没有计算完成。**注意：这是一个中间状态。**

终态：

* Failure：失败
* Cancellation：取消
* NULL：null

**get**

下面来看一下最简单的无超时时间的Get方法。

```java
public V get() throws InterruptedException, ExecutionException {
    if (Thread.interrupted()) {
      throw new InterruptedException();
    }
    Object localValue = value;
    if (localValue != null & !(localValue instanceof SetFuture)) {
      return getDoneValue(localValue);
    }
    Waiter oldHead = waiters;
    if (oldHead != Waiter.TOMBSTONE) {
      Waiter node = new Waiter();
      do {
        node.setNext(oldHead);
        if (ATOMIC_HELPER.casWaiters(this, oldHead, node)) {
          // we are on the stack, now wait for completion.
          while (true) {
            LockSupport.park(this);
            // Check interruption first, if we woke up due to interruption we need to honor that.
            if (Thread.interrupted()) {
              removeWaiter(node);
              throw new InterruptedException();
            }
            // Otherwise re-read and check doneness. If we loop then it must have been a spurious
            // wakeup
            localValue = value;
            if (localValue != null & !(localValue instanceof SetFuture)) {
              return getDoneValue(localValue);
            }
          }
        }
        oldHead = waiters; // re-read and loop.
      } while (oldHead != Waiter.TOMBSTONE);
    }
    // re-read value, if we get here then we must have observed a TOMBSTONE while trying to add a
    // waiter.
    return getDoneValue(value);
  }
```

* getDoneValue方法的进入条件是一个重点：localValue != null & !(localValue instanceof SetFuture)，如上面所强调，SetFuture是一个中间状态，非终态。所以在该状态时，不应该去获取值。
* 看一下getDoneValue方法:

  源码注释写的非常好，有两点：

  * Unboxes，拆箱。就是把包装成对象的生命周期状态，映射到具体的结果。
  * 非setFuture中间态（默认中间态不会进入这个方法）；  
    关于此处的一点思考：上层具体业务层代码+下层私有工具方法，需要在两处都进行参数校验吗？这个问题曾困扰我一段时间。现在来看，应该是上层业务代码来保证参数的有效性。（此处没有对setFuture方法进行判断）

  ```java
  //Unboxes obj. Assumes that obj is not null or a AbstractFuture.SetFuture.  
  private V getDoneValue(Object obj) throws ExecutionException {
      // While this seems like it might be too branch-y, simple benchmarking proves it to be
      // unmeasurable (comparing done AbstractFutures with immediateFuture)
      if (obj instanceof Cancellation) {
        throw cancellationExceptionWithCause("Task was cancelled.", ((Cancellation) obj).cause);
      } else if (obj instanceof Failure) {
        throw new ExecutionException(((Failure) obj).exception);
      } else if (obj == NULL) {
        return null;
      } else {
        @SuppressWarnings("unchecked") // this is the only other option
        V asV = (V) obj;
        return asV;
      }
    }
  ```

* 如果当前任务还没有执行完，则将当前线程包装成一个Waiter，以头插法插入等待队列。waiters是等待执行队列的头节点，TOMBSTONE是头节点的头节点（主要是个标识作用，简化链表操作）。
* ​`LockSupport.park(this);`​将当前线程阻塞到此处，让出cpu时间片。一般来讲需要别的线程unpark该线程or中断or等待时间已过。带阻塞完成之后，会再次判断；  
  注意：该构造器没有传入阻塞时间，因此默认调用`UNSAFE.park(false, 0L)`​方法，这意味着将无限期阻塞。
* 阻塞结束之后，会去再次getDoneValue获取值。

总结一下get方法：

* get方法仍为阻塞方法，让出cpu时间片。
* 注意获取值当条件为终态，非终态不获取值。
* 通过一个Waiter的阻塞队列，来管理所有的阻塞任务。

看起来和FutureTask的get方法没有本质区别。

**cancel**

下面来看一下取消的代码：

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    Object localValue = value;
    boolean rValue = false;
    if (localValue == null | localValue instanceof SetFuture) {
      // Try to delay allocating the exception. At this point we may still lose the CAS, but it is
      // certainly less likely.
      Throwable cause =
          GENERATE_CANCELLATION_CAUSES
              ? new CancellationException("Future.cancel() was called.")
              : null;
      Object valueToSet = new Cancellation(mayInterruptIfRunning, cause);
      AbstractFuture<?> abstractFuture = this;
      while (true) {
        if (ATOMIC_HELPER.casValue(abstractFuture, localValue, valueToSet)) {
          rValue = true;
          // We call interuptTask before calling complete(), which is consistent with
          // FutureTask
          if (mayInterruptIfRunning) {
            abstractFuture.interruptTask();
          }
          complete(abstractFuture);
          if (localValue instanceof SetFuture) {
            // propagate cancellation to the future set in setfuture, this is racy, and we don't
            // care if we are successful or not.
            ListenableFuture<?> futureToPropagateTo = ((SetFuture) localValue).future;
            if (futureToPropagateTo instanceof TrustedFuture) {
              // If the future is a TrustedFuture then we specifically avoid calling cancel()
              // this has 2 benefits
              // 1. for long chains of futures strung together with setFuture we consume less stack
              // 2. we avoid allocating Cancellation objects at every level of the cancellation
              //    chain
              // We can only do this for TrustedFuture, because TrustedFuture.cancel is final and
              // does nothing but delegate to this method.
              AbstractFuture<?> trusted = (AbstractFuture<?>) futureToPropagateTo;
              localValue = trusted.value;
              if (localValue == null | localValue instanceof SetFuture) {
                abstractFuture = trusted;
                continue;  // loop back up and try to complete the new future
              }
            } else {
              // not a TrustedFuture, call cancel directly.
              futureToPropagateTo.cancel(mayInterruptIfRunning);
            }
          }
          break;
        }
        // obj changed, reread
        localValue = abstractFuture.value;
        if (!(localValue instanceof SetFuture)) {
          // obj cannot be null at this point, because value can only change from null to non-null.
          // So if value changed (and it did since we lost the CAS), then it cannot be null and
          // since it isn't a SetFuture, then the future must be done and we should exit the loop
          break;
        }
      }
    }
    return rValue;
  }
```

从上到下，来解释一下关键步骤：

* 首先关注，主要执行逻辑的进入条件：`localValue == null | localValue instanceof SetFuture`​，这意味着，只有当任务还没有进入终态（结束）的时候才会进行。
* 业务任务还没有完成， 会以`Cancellation`​的形式给一个结果。

​`complete(abstractFuture)`​这段代码**非常重要，** 要点进去看一下：

```java
private static void complete(AbstractFuture<?> future) {
    Listener next = null;
    outer: while (true) {
      future.releaseWaiters();
      future.afterDone();
      // push the current set of listeners onto next
      next = future.clearListeners(next);
      future = null;
      while (next != null) {
        Listener curr = next;
        next = next.next;
        Runnable task = curr.task;
        if (task instanceof SetFuture) {
          SetFuture<?> setFuture = (SetFuture<?>) task;
          future = setFuture.owner;
          if (future.value == setFuture) {
            Object valueToSet = getFutureValue(setFuture.future);
            if (ATOMIC_HELPER.casValue(future, setFuture, valueToSet)) {
              continue outer;
            }
          }
          // other wise the future we were trying to set is already done.
        } else {
          executeListener(task, curr.executor);
        }
      }
      break;
    }
  }
```

大概干了这么几件事情：

* 唤醒所有等待次future的线程。（已经cancel进入终态了，是需要唤醒这些等待者的）
* ​`clearListeners`​，将所有的监听器清除。注意：是从监听器队列清除，但并没有直接删除。而是将这些清除的监听器再次反转链表收集起来。（收集起来准备执行，之所以反转是因为想将执行器的执行顺序与添加顺序保持一致，这只是实现者的好心，接口文档并不保证这一点。）

继续看`complete`​方法，上文说到将所有的监听器再次收集起来。收集起来之后，会遍历所有的监听器，执行其对应的逻辑。future没执行完，给临时结果，执行完了会调用监听器的逻辑。

​`complete`​执行完之后，将取消操作传播到具体的future上。其中如果如果这个future是`TrustedFuture`​的future，guava对其取消机制有了更好的优化。

总结一下cancel方法：

* 取消操作，首先关注任务执行状态。如果是终态，则直接返回结果，此时的取消没有意义。
* 非终态（SetFuture）的取消执行步骤比较多。

  * 即使未完成，也要给一个具体的结果。这个结果非业务上的结果，更多的是作为一个终态的判断标识。
  * 执行清理工作。一是唤醒所有等待此future的waiter线程。二是清空监听器列表。（此处可以理解，因为取消后也算作终态了，自然要唤醒等待着，清空监听器）；三是将所有监听器的值给出去。（上面的清空只是清空队列，而非删除）；
* 将取消操作传播到具体的future上去。（等价于调用真正的cancel方法）

**setFuturn&amp;SetException**

这两个方法都是一个“终结”的方法：

```java
protected boolean set(@Nullable V value) {
    boolean result = sync.set(value);
    if (result) {
      executionList.execute();
    }
    return result;
  }
```

```java
 protected boolean setException(Throwable throwable) {
    boolean result = sync.setException(checkNotNull(throwable));
    if (result) {
      executionList.execute();
    }
    return result;
  }
```

两者有一些共同点：

* 均是在设置完结果之后，运行了所有的执行器，相当于执行了所有的listener。（这就是所谓的回调）
* 对于其中的sync.set(value)、sync.setException(checkNotNull(throwable))方法，两者也几乎一致，都又调用了`complete`​方法，这个`complete`​方法和前面的`complete`​方法不一样，点进去看一下：

  * 此处调用的complete是在已经有结果之后，此时主要就做一件事情：扭转任务的生命周期状态为COMPLETING，顺便把异常给加上去了。

  ```java
      private boolean complete(@Nullable V v, @Nullable Throwable t,
          int finalState) {
        boolean doCompletion = compareAndSetState(RUNNING, COMPLETING);
        if (doCompletion) {
          // If this thread successfully transitioned to COMPLETING, set the value
          // and exception and then release to the final state.
          this.value = v;
          // Don't actually construct a CancellationException until necessary.
          this.exception = ((finalState & (CANCELLED | INTERRUPTED)) != 0)
              ? new CancellationException("Future.cancel() was called.") : t;
          releaseShared(finalState);
        } else if (getState() == COMPLETING) {
          // If some other thread is currently completing the future, block until
          // they are done so we can guarantee completion.
          acquireShared(-1);
        }
        return doCompletion;
      }
  ```

此时的complete和上面的complete有所区别。主要的区别在于调用时间的不同，cancel中的complete时，任务还没有完成，为了保证回调有效就要激活所有的回调。而setXXX的时候已经有了结果，就不用在complete里面激活所有的回调了。

在这里也可以看到对于异常的处理，就是包裹了一下，在exception里面带了出去。

**ListenableFuture的内容基本看完了。了解回调、异常、并发控制等概念。但是还是没有回答问题：如何恰好知道任务已经结束了呢？**

来Futures看一下吧。

## Futures

先看一下最经典的addCallBack的方法：

```java
  public static <V> void addCallback(
      ListenableFuture<V> future, FutureCallback<? super V> callback) {
    addCallback(future, callback, directExecutor());
  }
```

继续看addCallback：

```java
  public static <V> void addCallback(
      final ListenableFuture<V> future,
      final FutureCallback<? super V> callback,
      Executor executor) {
    Preconditions.checkNotNull(callback);
    future.addListener(new CallbackListener<V>(future, callback), executor);
  }
```

此处可以看到将future和callback包装成了一个CallbackListener，这个CallbackListener实现了Runnable，所以重点关注起run方法：

```java
    public void run() {
      final V value;
      try {
        value = getDone(future);
      } catch (ExecutionException e) {
        callback.onFailure(e.getCause());
        return;
      } catch (RuntimeException e) {
        callback.onFailure(e);
        return;
      } catch (Error e) {
        callback.onFailure(e);
        return;
      }
      callback.onSuccess(value);
    }
```

非常的简单。拿到结果就success，出意外了就failure。其中的getDone方法：

```java
  public static <V> V getDone(Future<V> future) throws ExecutionException {
    checkState(future.isDone(), "Future was expected to be done: %s", future);
    return getUninterruptibly(future);
```

可以发现，在此处会检查一下future是不是结束了。如果没有结束，则直接抛出异常。也就是说，当执行回调的时候一定是future已经执行完毕了。

在这个可以看到Futures.addCallBack是如何运转的。整个处理非常的简单。所以还剩最后一个一直没有解决的问题，回调的时机到底是哪里在控制？没有使用轮询的方式，那使用了什么方式呢？我怎么知道future有没有执行完呢？

## MoreExcutors

如何获得一个LisenableFuture的实例呢？可以考虑使用这个方法。这个工具类就是将java的线程池包装成guava的线程池。秘密就在这里面。  
可以使用如下方法提交一个任务，并且获得一个LisenableFuture的实例：

```java
    public static void main(String[] args) throws Exception {
        ListeningExecutorService listeningExecutorService = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(1));
        ListenableFuture<Object> submit = listeningExecutorService.submit(() -> null);
    }
```

看一下listeningDecorator：

```java
public static ListeningExecutorService listeningDecorator(ExecutorService delegate) {
    return (delegate instanceof ListeningExecutorService)
        ? (ListeningExecutorService) delegate
        : (delegate instanceof ScheduledExecutorService)
            ? new ScheduledListeningDecorator((ScheduledExecutorService) delegate)
            : new ListeningDecorator(delegate);
  }
```

直接看最通用一个装饰器类的实例ListeningDecorator：

```java
  private static class ListeningDecorator extends AbstractListeningExecutorService {
    private final ExecutorService delegate;

    ListeningDecorator(ExecutorService delegate) {
      this.delegate = checkNotNull(delegate);
    }

    @Override
    public final boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
      return delegate.awaitTermination(timeout, unit);
    }

    @Override
    public final boolean isShutdown() {
      return delegate.isShutdown();
    }

    @Override
    public final boolean isTerminated() {
      return delegate.isTerminated();
    }

    @Override
    public final void shutdown() {
      delegate.shutdown();
    }

    @Override
    public final List<Runnable> shutdownNow() {
      return delegate.shutdownNow();
    }

    @Override
    public final void execute(Runnable command) {
      delegate.execute(command);
    }
  }
```

这个方法实际啥也没干。看一下其父类`AbstractListeningExecutorService`​方法：

```java
public abstract class AbstractListeningExecutorService extends AbstractExecutorService
    implements ListeningExecutorService {

  /** @since 19.0 (present with return type {@code ListenableFutureTask} since 14.0) */
  @Override
  protected final <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return TrustedListenableFutureTask.create(runnable, value);
  }

  /** @since 19.0 (present with return type {@code ListenableFutureTask} since 14.0) */
  @Override
  protected final <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return TrustedListenableFutureTask.create(callable);
  }

  @Override
  public ListenableFuture<?> submit(Runnable task) {
    return (ListenableFuture<?>) super.submit(task);
  }

  @Override
  public <T> ListenableFuture<T> submit(Runnable task, @Nullable T result) {
    return (ListenableFuture<T>) super.submit(task, result);
  }

  @Override
  public <T> ListenableFuture<T> submit(Callable<T> task) {
    return (ListenableFuture<T>) super.submit(task);
  }
}
```

可以发起`AbstractListeningExecutorService`​重写了`newTaskFor`​方法，`newTaskFor`​方法会在submit的时候调用：

```java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
```

所以，看一下这个newTaskFor的具体实现：

```java
  static <V> TrustedListenableFutureTask<V> create(Runnable runnable, @Nullable V result) {
    return new TrustedListenableFutureTask<V>(Executors.callable(runnable, result));
  }
```

看一下TrustedListenableFutureTask其中run方法实现：

```java
  public void run() {
    TrustedFutureInterruptibleTask localTask = task;
    if (localTask != null) {
      localTask.run();
    }
  }
```

其中调用了TrustedFutureInterruptibleTask的run方法，看看这个run方法，该run方法实际是TrustedFutureInterruptibleTask父类的run方法：

```java
  public final void run() {
    if (!ATOMIC_HELPER.compareAndSetRunner(this, null, Thread.currentThread())) {
      return; // someone else has run or is running.
    }
    try {
      runInterruptibly();
    } finally {
      if (wasInterrupted()) {
        while (!doneInterrupting) {
          Thread.yield();
        }
      }
    }
  }
```

TrustedFutureInterruptibleTask里面重写了runInterruptibly方法，这个是重点：

```java
    void runInterruptibly() {
      // Ensure we haven't been cancelled or already run.
      if (!isDone()) {
        try {
          set(callable.call());
        } catch (Throwable t) {
          setException(t);
        }
      }
    }
```

重点关注set方法。首先是调用call方法。此处是启动了future的业务逻辑。在业务逻辑执行完毕之后，进入set的逻辑：

```java
  protected boolean set(@Nullable V value) {
    Object valueToSet = value == null ? NULL : value;
    if (ATOMIC_HELPER.casValue(this, null, valueToSet)) {
      complete(this);
      return true;
    }
    return false;
  }
```

这个又回到了abstractFuture的熟悉逻辑。将所有监听器执行一遍。此处会触发监听器的回调。

因此，业务逻辑触发的时机就在这里。

‍

‍
