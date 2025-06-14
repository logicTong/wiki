
以下从源码角度深入解析 Android 中 `Handler` 的内部实现，涵盖核心组件、工作流程、线程模型、消息机制等关键细节：


### 一、核心组件与类关系
#### 1. 四大核心类
- **`Handler`**：消息的发送者和处理者，负责将消息/任务加入队列，并在目标线程中处理。
- **`MessageQueue`**：消息队列，存储待处理的 `Message` 和 `Runnable`（封装为 `Message` 的 `callback`），内部通过单链表实现。
- **`Looper`**：每个线程的消息循环器，绑定唯一的 `MessageQueue`，负责循环取出消息并交给 `Handler` 处理。
- **`ThreadLocal<Looper>`**：线程局部变量，确保每个线程仅有一个 `Looper` 实例。

#### 2. 类关系图
```
Thread (当前线程)
├─ ThreadLocal<Looper> (存储当前线程的 Looper)
└─ Looper
   ├─ MessageQueue mQueue (消息队列)
   └─ static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<>();
```


### 二、初始化流程：Handler 与 Looper 的绑定
#### 1. Handler 构造函数（关键源码）
```java
public Handler() {
    this(null, false); // 无参构造调用此方法
}

public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
        }
    }
    mLooper = Looper.myLooper(); // 获取当前线程的 Looper（核心！）
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
            + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue; // 获取 Looper 关联的 MessageQueue
    mCallback = callback;
    mAsynchronous = async;
}
```
- **`Looper.myLooper()` 实现**：
  ```java
  public static @Nullable Looper myLooper() {
      return sThreadLocal.get(); // 从 ThreadLocal 中获取当前线程的 Looper
  }
  ```
- **结论**：  
  `Handler` 必须绑定一个 `Looper`，而 `Looper` 通过 `ThreadLocal` 与当前线程一一对应。若当前线程未创建 `Looper`（如子线程未调用 `Looper.prepare()`），构造 `Handler` 会抛出异常。

#### 2. Looper 的创建（线程绑定）
- **主线程（UI 线程）**：  
  在 `ActivityThread.main()` 中，系统自动调用：
  ```java
  Looper.prepareMainLooper(); // 调用 Looper.prepare() 并标记为主 Looper
  Looper.loop(); // 启动消息循环
  ```
- **子线程**：  
  需手动调用：
  ```java
  Looper.prepare(); // 创建 Looper 并存入 ThreadLocal
  new Handler().post(...) // 此时 Handler 绑定当前线程的 Looper
  Looper.loop(); // 启动消息循环（无限循环，需手动退出）
  ```


### 三、消息发送：从 Handler 到 MessageQueue
#### 1. 发送消息的核心方法
- **`post(Runnable r)`**：  
  封装为 `Message` 的 `callback` 发送：
  ```java
  public final boolean post(@NonNull Runnable r) {
      return sendMessageDelayed(getPostMessage(r), 0);
  }

  private static Message getPostMessage(Runnable r) {
      Message m = Message.obtain(); // 从消息池获取 Message
      m.callback = r; // 设置 Runnable 为回调
      return m;
  }
  ```
- **`sendMessage(Message msg)`**：  
  直接发送 `Message`，需手动设置 `what` 或 `obj`：
  ```java
  public final boolean sendMessage(@NonNull Message msg) {
      return sendMessageDelayed(msg, 0);
  }
  ```

#### 2. `enqueueMessage`：消息入队（关键源码）
```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, long uptimeMillis) {
    msg.target = this; // 关键！设置 Message 的 target 为当前 Handler
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis); // 调用 MessageQueue 的入队方法
}
```
- **`msg.target = this`**：  
  确保消息被取出时，能通过 `target` 找到对应的 `Handler` 来处理消息。

#### 3. `MessageQueue.enqueueMessage`：单链表插入
```java
boolean enqueueMessage(Message msg, long when) {
    // 消息不可重复使用，已回收则抛出异常
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) { // 同步块保证线程安全
        if (mQuitting) { // 队列正在退出，消息被丢弃（除非是异步消息且队列未断开）
            boolean res = mAsyncMessages != null && mAsyncMessages.enqueueMessage(msg);
            if (!res && !msg.isAsynchronous()) {
                return false;
            }
        }
        msg.markInUse(); // 标记消息为已使用
        msg.when = when; // 设置消息执行时间（绝对时间，基于 SystemClock.uptimeMillis()）
        Message p = mMessages; // 链表头节点
        boolean needWake;
        if (p == null || when == 0 || when < p.when) { // 插入到链表头部
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; // 当前队列是否阻塞，若阻塞则需要唤醒
        } else { // 插入到链表中间或尾部
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
            msg.next = p;
            prev.next = msg;
        }
        // 若当前队列阻塞（在 nativePollOnce 中等待），则通过 native 方法唤醒
        if (needWake) {
            nativeWake(mPtr); // 唤醒底层 epoll 等待
        }
    }
    return true;
}
```
- **核心逻辑**：  
  消息按 `when` 时间戳排序，形成有序链表；通过 `synchronized` 保证多线程安全；延迟消息（`sendMessageDelayed`）通过 `when` 控制执行时机。


### 四、消息循环：Looper.loop() 的无限循环
#### 1. `Looper.loop()` 源码（核心流程）
```java
public static void loop() {
    final Looper me = myLooper(); // 获取当前线程的 Looper
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue; // 获取 MessageQueue

    // 允许系统监控循环（如 ANR 检测）
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) { // 无限循环，直到队列退出
        Message msg = queue.next(); // 取出消息（可能阻塞）
        if (msg == null) { // 队列退出时返回 null，循环终止
            return;
        }

        try {
            msg.target.dispatchMessage(msg); // 分发消息给 Handler
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        // 回收消息到消息池
        msg.recycleUnchecked();
    }
}
```

#### 2. `MessageQueue.next()`：取消息（阻塞与唤醒）
```java
Message next() {
    final long ptr = mPtr; //  native 指针，关联底层 poll 机制
    int pendingIdleHandlerCount = -1; // 第一次空闲时需要处理的 IdleHandler 数量
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 关键！native 方法阻塞，直到消息可用或被唤醒
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 检查是否有消息到期
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) { // 处理同步屏障（非同步消息）
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) { // 消息未到执行时间，计算超时时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 取出消息，从链表中移除
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    return msg; // 返回消息
                }
            } else {
                nextPollTimeoutMillis = -1; // 无消息时永久阻塞
            }

            // 处理队列退出标记
            if (mQuitting) {
                dispose();
                return null;
            }

            // 处理 IdleHandler（队列空闲时执行）
            if (pendingIdleHandlerCount < 0 && (mMessages == null || now >= mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                mBlocked = true;
                continue;
            }

            // 复制 IdleHandler 列表，避免迭代时修改
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 执行 IdleHandler
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // 置空，允许回收

            boolean keep = false;
            try {
                keep = idler.queueIdle(); // 调用用户注册的 IdleHandler
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) { // 若 IdleHandler 不需要保持，从列表移除
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0; // 执行完 IdleHandler 后重新检查消息
    }
}
```
- **阻塞机制**：  
  通过 `nativePollOnce` 实现底层阻塞（基于 Linux 的 `epoll` 或 `poll`），等待消息到达或被唤醒。延迟消息通过 `nextPollTimeoutMillis` 控制阻塞时间。
- **同步屏障（异步消息）**：  
  当发送异步消息（`Handler` 构造时设置 `async=true` 或通过 `sendMessageAtTime` 发送异步消息），`next()` 会跳过同步消息，优先处理异步消息，用于动画等实时性场景。


### 五、消息处理：Handler.dispatchMessage(msg)
#### 1. 消息分发逻辑
```java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) { // 优先处理 Runnable 回调（post 发送的任务）
        handleCallback(msg);
    } else if (mCallback != null) { // 处理用户设置的 Callback
        if (mCallback.handleMessage(msg)) {
            return;
        }
    }
    handleMessage(msg); // 最后调用用户重写的 handleMessage（最常用方式）
}

private static void handleCallback(Message message) {
    message.callback.run(); // 执行 Runnable
}
```
- **处理顺序**：  
  `msg.callback（Runnable）` → `mCallback（Handler.Callback）` → `handleMessage（用户重写）`。


### 六、消息池：性能优化关键
#### 1. 消息复用机制
`Message` 通过 `obtain()` 方法从消息池中获取，而非直接 `new`，避免频繁创建对象：
```java
public final class Message implements Parcelable {
    // 消息池头部
    private static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;

    // 从消息池获取消息
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // 清除 IN_USE 标记
                sPoolSize--;
                return m;
            }
        }
        return new Message(); // 池空时创建新实例
    }

    // 回收消息到池中（非强引用，避免内存泄漏）
    void recycleUnchecked() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("Message " + this + " cannot be recycled because it is still in use.");
            }
            return;
        }
        flags = FLAG_IN_USE; // 标记为已使用（防止重复回收）
        next = sPool;
        sPool = this;
        sPoolSize++;
    }
}
```
- **注意**：  
  消息池最大容量无显式限制，但过多未回收的消息可能导致内存占用，需避免长时间持有 `Message` 对象。


### 七、线程模型与常见问题
#### 1. 主线程与子线程的区别
| 场景                | 主线程（UI 线程）                          | 子线程                                  |
|---------------------|-------------------------------------------|-----------------------------------------|
| Looper 初始化       | 系统自动调用 `Looper.prepareMainLooper()`   | 需手动调用 `Looper.prepare()` 和 `loop()` |
| Handler 构造        | 直接创建（已有 Looper）                    | 必须先准备 Looper                        |
| 消息循环终止        | 无法手动终止（应用退出时系统终止）         | 需调用 `Looper.myLooper().quit()` 退出    |

#### 2. 子线程使用 Handler 的正确方式
```java
class WorkerThread extends Thread {
    private Handler mHandler;

    @Override
    public void run() {
        Looper.prepare(); // 1. 准备 Looper
        mHandler = new Handler() { // 2. 创建 Handler（绑定当前线程 Looper）
            @Override
            public void handleMessage(@NonNull Message msg) {
                // 处理消息
            }
        };
        Looper.loop(); // 3. 启动消息循环（无限循环，需通过 quit() 退出）
    }

    public Handler getHandler() {
        return mHandler;
    }
}

// 使用
WorkerThread workerThread = new WorkerThread();
workerThread.start();
Handler workerHandler = workerThread.getHandler();
workerHandler.sendEmptyMessage(0); // 向子线程发送消息
```

#### 3. 内存泄漏风险与解决方案
- **原因**：  
  非静态内部类 `Handler` 持有外部类（如 `Activity`）的强引用，若消息延迟发送且 `Activity` 已销毁，消息仍保留对 `Activity` 的引用，导致无法回收。
  
- **解决方案**：  
  ```java
  public class MyActivity extends AppCompatActivity {
      // 静态内部类（不持有 Activity 强引用）
      private static class MyHandler extends Handler {
          private final WeakReference<MyActivity> weakActivity;

          public MyHandler(MyActivity activity) {
              weakActivity = new WeakReference<>(activity);
          }

          @Override
          public void handleMessage(@NonNull Message msg) {
              MyActivity activity = weakActivity.get();
              if (activity != null) {
                  // 处理消息
              }
          }
      }

      private final MyHandler mHandler = new MyHandler(this);

      @Override
      protected void onDestroy() {
          super.onDestroy();
          mHandler.removeCallbacksAndMessages(null); // 清除所有未处理消息
      }
  }
  ```


### 八、总结：Handler 核心特性
1. **线程隔离**：通过 `ThreadLocal` 保证每个线程仅有一个 `Looper`，实现跨线程通信。  
2. **有序调度**：消息按 `when` 排序，支持延迟执行和定时任务（`postDelayed`/`sendMessageDelayed`）。  
3. **性能优化**：消息池复用 `Message` 对象，减少内存分配和垃圾回收。  
4. **灵活处理**：支持三种消息处理方式（`Runnable`、`Handler.Callback`、`handleMessage`）。  

理解 `Handler` 的核心在于掌握 `Looper` 与线程的绑定关系、`MessageQueue` 的入队/出队逻辑，以及消息循环中阻塞与唤醒的底层实现。这些细节不仅是面试重点，也是优化异步任务和避免内存泄漏的关键。