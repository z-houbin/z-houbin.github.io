# Handler消息机制解析和简单实现



## 前言

Handler 是安卓的上层消息机制接口，通常用于不同线程之间的消息通信，比如子线程通知主线程更新UI，主线程通知子线程处理耗时操作，同时支持定时消息和延迟消息。Handler的消息机制在安卓开发中有着举足轻重的作用，我们有必要了解其基本流程和实现原理。



## Handler 解析

这里以一段Handler的基础使用代码来逐渐展开，步骤包括：

- 创建Handler实例
- 重写handleMessage接收消息
- 调用send或者post发送消息

```java
        Handler handler = new Handler(Looper.myLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                System.out.println(msg.toString());
            }
        };
        handler.sendEmptyMessage(0);
```

首先从构造函数入手，既然可以空参数那么证明其他参数会自行处理，直接进入默认构造

 ```java
 public Handler()
 public Handler(@Nullable Callback callback)
 public Handler(@NonNull Looper looper)
 public Handler(@NonNull Looper looper, @Nullable Callback callback)
 public Handler(boolean async)
 public Handler(@Nullable Callback callback, boolean async) 
 public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async)
 ```

默认构造会走到如下构造函数，参数为 [null，false]

```java
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

忽略前面不必要的代码，在构造里面判断Looper.myLooper()是否为null，同时获取了mLooper.mQueue对象。这里暂停Handler源码分析，看看Looper是什么。



## Looper 解析

Looper主要用于处理线程的消息循环，线程默认是没有关联Looper的，调用Looper.prepare() 准备和关联当前线程，采用ThreadLocal保证了线程的封闭，调用Looper.loop()开始处理消息。

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
	    sThreadLocal.set(new Looper(quitAllowed));
}
```
loop 函数获取了当前线程对应Looper，如果不存在会抛出RuntimeException，所以在Looper使用中需要配对使用prepare和loop函数。

继续向下发现有获取Looper对应mQueue（消息队列），mQueue的初始化在构造函数中。

接下来是消息内容的循环处理，如果获取到消息为null会直接跳出循环，Looper流程则结束。正常获取到消息则会调用消息的 msg.target.dispatchMessage(msg)，msg.target 对象为Handler实例，在这里我们只要重写Handler的dispatchMessage(msg)就可以接收到消息了。接下来看看mQueue消息队列是如何处理消息的。

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
	final Looper me = myLooper();
	if (me == null) {
		throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
	}
	if (me.mInLoop) {
		Slog.w(TAG, "Loop again would have the queued messages be executed"
				+ " before this one completed.");
	}

	me.mInLoop = true;
	final MessageQueue queue = me.mQueue;

	// Make sure the identity of this thread is that of the local process,
	// and keep track of what that identity token actually is.
	Binder.clearCallingIdentity();
	final long ident = Binder.clearCallingIdentity();

	// Allow overriding a threshold with a system prop. e.g.
	// adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
	final int thresholdOverride =
			SystemProperties.getInt("log.looper."
					+ Process.myUid() + "."
					+ Thread.currentThread().getName()
					+ ".slow", 0);

	boolean slowDeliveryDetected = false;

	for (;;) {
		Message msg = queue.next(); // might block
		if (msg == null) {
			// No message indicates that the message queue is quitting.
			return;
		}

		// This must be in a local variable, in case a UI event sets the logger
		final Printer logging = me.mLogging;
		if (logging != null) {
			logging.println(">>>>> Dispatching to " + msg.target + " " +
					msg.callback + ": " + msg.what);
		}
		// Make sure the observer won't change while processing a transaction.
		final Observer observer = sObserver;

		final long traceTag = me.mTraceTag;
		long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
		long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
		if (thresholdOverride > 0) {
			slowDispatchThresholdMs = thresholdOverride;
			slowDeliveryThresholdMs = thresholdOverride;
		}
		final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
		final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

		final boolean needStartTime = logSlowDelivery || logSlowDispatch;
		final boolean needEndTime = logSlowDispatch;

		if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
			Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
		}

		final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
		final long dispatchEnd;
		Object token = null;
		if (observer != null) {
			token = observer.messageDispatchStarting();
		}
		long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
		try {
			msg.target.dispatchMessage(msg);
			if (observer != null) {
				observer.messageDispatched(token, msg);
			}
			dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
		} catch (Exception exception) {
			if (observer != null) {
				observer.dispatchingThrewException(token, msg, exception);
			}
			throw exception;
		} finally {
			ThreadLocalWorkSource.restore(origWorkSource);
			if (traceTag != 0) {
				Trace.traceEnd(traceTag);
			}
		}
		if (logSlowDelivery) {
			if (slowDeliveryDetected) {
				if ((dispatchStart - msg.when) <= 10) {
					Slog.w(TAG, "Drained");
					slowDeliveryDetected = false;
				}
			} else {
				if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
						msg)) {
					// Once we write a slow delivery log, suppress until the queue drains.
					slowDeliveryDetected = true;
				}
			}
		}
		if (logSlowDispatch) {
			showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
		}

		if (logging != null) {
			logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
		}

		// Make sure that during the course of dispatching the
		// identity of the thread wasn't corrupted.
		final long newIdent = Binder.clearCallingIdentity();
		if (ident != newIdent) {
			Log.wtf(TAG, "Thread identity changed from 0x"
					+ Long.toHexString(ident) + " to 0x"
					+ Long.toHexString(newIdent) + " while dispatching to "
					+ msg.target.getClass().getName() + " "
					+ msg.callback + " what=" + msg.what);
		}

		msg.recycleUnchecked();
	}
}

```



## MessageQueue 解析

MessageQueue 用于Looper的消息分发，包含了要发送的消息列表，可以通过Looper.myQueue()获取到当前线程的消息队列但是无法查看到具体的消息内容。MessageQueue里面两个重要的函数分别是enqueueMessage和next。

enqueueMessage用于添加消息，主要是单链表的插入操作
```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }

    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

next函数用于读取下一条消息，如果消息队列中没有消息，那么next方法会一直阻塞知道下一条消息到来。当有新消息到来时，next会返回新的消息并从单链表中删除消息。

```java
@UnsupportedAppUsage
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

消息的分发以及处理搞清楚了，那么回过头来看看消息是如何添加的



##  Handler 解析2

回到前面的发送消息的地方，看看消息的发送流程是怎样的

```
handler.sendEmptyMessage(0);
```

经过发送函数的层层重载，最终调用到的发送消息如下

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
                               long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

第一个参数为消息队列，第二个参数为消息对象，第三个参数为消息接收的时间，默认为SystemClock.uptimeMillis()。在这里会设置消息的target为当前Handler用于消息回调，最后调用MessageQueue的enqueueMessage添加消息到队列中。



## 小结

消息机制的整个流程大致清晰了

1. Looper.prepare() 绑定和准备消息队列
2. 创建Handler对象准备发送接收消息
3. Looper.loop() 开始循环处理消息
4. Handler发送消息



基于以上分析流程，简单实现了一遍Handler，源码见 [⚡SampleHandler](https://github.com/z-houbin/SampleHandler)

