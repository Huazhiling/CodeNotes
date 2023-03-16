- ### Handler，Message，MessageQueue，Looper之间的关系
  [![Handler关系图](img/Handler.jpg "Handler关系图")](https://github.com/Huazhiling/CodeNotes/blob/master/src/Android/img/Handler.jpg)  
  每个线程会有一个Looper，在创建Looper的时候会在创建出一个MessageQueue（MQ），MQ是一个以时间排序由单链表实现的优先队列，管理所有的Message。  
  由于Android是事件驱动，所以又叫事件分发机制，Handler也被称为Handler事件分发机制。  
  Handler可以说是Looper，Message，MQ的驱动者，Handler在创建时会根据当前线程绑定Looper，
  以Looper为链接将Message和MQ也都与自身关联在了一起。
- ### Handler事件分发
  首先，Looper会通过loop()方法开始真正的工作，通过一个"无限"的for循环不停的从MQ中读取，并处理消息。  
  Handler通过sendMessageXXX方法发送一条事件消息，无论调用哪个sendXXX方法，最终都会回到sendMessageAtTime，然后通过MQ的enqueue方法将这个消息加入队列。  
  加入队列的过程是判断这条消息的delay事件，如果时间小于当前最新一条消息的时间，那么会直接插入到队首，第一个消费。否则的话会通过队列循环一个一个判断时间，直到找到合适的位置（可能根据设定的时间插入在中间，也可能在队尾），
  此次，Handler消息入队逻辑完成。
  由于Looper会不听的循坏MQ去去消息，所以会通过MQ的next()方法，拿取队头的消息，在经过nativePollOnce方法的时候，如果当前的MQ里没有消息，那么就会一直阻塞在这里，
  直到超过了设定的时间，或者取到了新消息才会返回。  
  当拿到消息之后，返回该消息，并且把这条消息移除队列，然后通过Handler的dispatchMessage方法，将事件通过回调的方法返回。  
  当Handler把该事件消耗后，此次事件的分发到此结束

- ### Handler回调优先级
  回调优先级顺序如下  
  [![Handler回调顺序.png](img/Handler回调顺序.png "Handler回调顺序")](https://github.com/Huazhiling/CodeNotes/blob/master/src/Android/img/Handler回调顺序.jpg)

# 进阶一下

- ### 一个线程有几个Looper
  一个线程只会有一个Looper，具体通过ThreadLocal类实现，保存的是线程变量，如果当前保存的线程已经存在，则会抛出异常
  ```java
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
  
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
  ```
  在prepare方法里，我们可以看到，如果从TL里面获取到数据不为空，证明当前线程已经有Looper了，就会抛出异常提示。
  TreadLocal的实现会在后续更新

    - ### 为什么Looper的for循环不会导致线程卡死
      一段代码肯定是按照顺序去直接，当执行到最后一行代码的时候，这个程序也就退出了，所以想要一直保持，就必须要让代码无法结束。  
      这里用的就是一段"无限制条件"
      的for循环，因为Android是基于Linux内核的，所以用的是epoll管道机制进行一个事件的通知消费。  
      for循环本身不会导致程序的卡死，宕机。真正造成卡死的原因，是因为在某些方法里做了耗时操作，又或者说做了超过规定时间的事情，比如`Thread.sleep()`。   
      举个例子说，代码上通过`invalidate()`方法想要通知UI更新，或者点击屏幕出发了`dispathTouchEvent()`
      方法进行事件传递，但在做这个这个事情的时候，你又触发了耗时操作，
      就会导致这个方法无法传递，必须要等到耗时操作结束之后才能继续运转，在体验上就会有明显的卡顿，如果耗时过长，那自然就会导致卡死崩溃，系统就会直接提示无响应。
      那既然说了for循环本身不会导致程序卡死，又需要通过for循环不停的去MQ取消息进行分发，那他是怎么做到的呢？接下来我们看代码，通过Java层->
      Native层，看一看这一块逻辑是怎么处理的

      涉及到的类用
  > `Looper.java` `MessageQueue.java`
  > `Looper.cpp` `NativeMessageQueue.cpp` `MessageQueue.cpp`

  Looper.java
    ```java
      public static void prepare() {
          prepare(true);
      }

      private static void prepare(boolean quitAllowed) {
          if (sThreadLocal.get() != null) {
              throw new RuntimeException("Only one Looper may be created per thread");
          }
          sThreadLocal.set(new Looper(quitAllowed));
      }
    
      public static void prepareMainLooper() {
          prepare(false);
          synchronized (Looper.class) {
              if (sMainLooper != null) {
                  throw new IllegalStateException("The main Looper has already been prepared.");
              }
              sMainLooper = myLooper();
          }
      }
    
      /**
      * ================上面为创建并绑定Looper的必要方法================
      */
    
      private Looper(boolean quitAllowed) {
          //绑定MessageQueue
          mQueue = new MessageQueue(quitAllowed);
          mThread = Thread.currentThread();
      }
    
      public static void loop() {
          //获取绑定的Looper
          final Looper me = myLooper();
          if (me == null) {
              throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
          }
          //拿到当前Looper绑定的MQ
          final MessageQueue queue = me.mQueue;
          //这里就是一直被理解为死循环为什么不会导致卡死的地方，具体的可以看后面的next()方法
          for (;;) {
              //开始从MQ中拿取最新的需要处理的消息，可能会阻塞
              Message msg = queue.next();
              if (msg == null) {
                  return;
              }
              /*..忽略一些和本章无关紧要的代码.*/
              try {
                  //拿到消息之后，发给Handler把消息处理掉
                  //此处的target就是Handler
                  msg.target.dispatchMessage(msg);
                  end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
              } finally {
                  if (traceTag != 0) {
                      Trace.traceEnd(traceTag);
                  }
              }
              /*..忽略一些和本章无关紧要的代码.*/
              msg.recycleUnchecked();
          }
      }
      /**
      *   拿到当前线程所持有的Looper
      */
      public static @Nullable Looper myLooper() {
          return sThreadLocal.get();
      }
    ```
  这里获取当前持有Looper，然后回去拿到所绑定的MQ，通过调用next()方法拿到最新的消息事件，如果当前没有需要处理的消息的话，
  那么会处于一个`等待用户输入`的状态，直到有消息回来。后续根据msg中的target，通过dispatchMessage把消息分发下来，target就是所对应的Handler。
    ```java
      public void dispatchMessage(Message msg) {
          if (msg.callback != null) {
              handleCallback(msg);
          } else {
              if (mCallback != null) {
                  if (mCallback.handleMessage(msg)) {
                      return;
                  }
              }
              handleMessage(msg);
          }
      }
    ```
  先会判断msg的callback是否可用，再判断handler的callback是否可以，如果都没有，就调用默认的handleMessage方法。
  总之步骤就是，回调是否可以，都不可用就调用默认方法，否则的话就回调直接处理，不再往下分发

  MessageQueue.java
    ```java
      private native static long nativeInit();
      private native void nativePollOnce(long ptr, int timeoutMillis);
  
      MessageQueue(boolean quitAllowed) {
          mQuitAllowed = quitAllowed;
          mPtr = nativeInit();
      }
  
      Message next() {
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
                  /*..忽略一些和本章无关紧要的代码.*/
              }
              /*..忽略一些和本章无关紧要的代码.*/
          }
      }
    ```
  这个主要关注`nativeInit`和`nativePollOnce`，其它操作就是正常的链表添加/移除。  
  `Looper.cpp，NativeMessageQueue.cpp，android_os_MessageQueue.cpp`  
  nativeInit()方法
    ```
    static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
      NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
      if (!nativeMessageQueue) {
          jniThrowRuntimeException(env, "Unable to allocate native queue");
          return 0;
      }

      nativeMessageQueue->incStrong(env);
      return reinterpret_cast<jlong>(nativeMessageQueue);
    }
    NativeMessageQueue::NativeMessageQueue() :
          mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
      mLooper = Looper::getForThread();
      if (mLooper == NULL) {
          mLooper = new Looper(false);
          Looper::setForThread(mLooper);
      }
    }
    
    Looper::Looper(bool allowNonCallbacks)
    : mAllowNonCallbacks(allowNonCallbacks),
    mSendingMessage(false),
    mPolling(false),
    mEpollRebuildRequired(false),
    mNextRequestSeq(0),
    mResponseIndex(0),
    mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC));
    LOG_ALWAYS_FATAL_IF(mWakeEventFd.get() < 0, "Could not make wake event fd: %s", strerror(errno));
    AutoMutex _l(mLock);
    rebuildEpollLocked();
    }
  
    void Looper::rebuildEpollLocked() {
      // 如果已经有了epoll，要先关闭旧的
      if (mEpollFd >= 0) {
        mEpollFd.reset();
      }
      // 重新创建新的epoll并注册wake管道
      mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC));
  
      struct epoll_event eventItem;
      memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
      //事件可读
      eventItem.events = EPOLLIN;
      //将事件的fd设置到eventItem中
      eventItem.data.fd = mWakeEventFd.get();
      //epoll_ctl用于添加到进行监听
      //这里就把mWakeEventFd添加到了监听里面
      int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &eventItem);
      LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
                          strerror(errno));
      //这里添加一些其它事件，你如键盘，Input等等
      for (size_t i = 0; i < mRequests.size(); i++) {
          const Request& request = mRequests.valueAt(i);
          struct epoll_event eventItem;
          request.initEventItem(&eventItem);
  
          int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, request.fd, &eventItem);
          if (epollResult < 0) {
              ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                    request.fd, strerror(errno));
          }
      }
    }
  ```
  这里可以看到，nativeInit()方法后，会构造一个NativeMessageQueue，在其内部会构造C层的Looper  
  注意：这里的Looper和Java层的完全不是一个东西，不要混为一谈。  
  Java层的Looper会用ThreadLocal保证唯一性，C层的Looper也会通过TLS保证唯一性。  
  这个方法主要还做了创建epoll，并且把需要的事件添加到监听管道中。

  nativePollOnce()方法
  ```
  static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
  }
  
  
  void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
  }
  
  
    int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    /**省略一些*/
        result = pollInner(timeoutMillis);
    }
    
    int Looper::pollInner(int timeoutMillis) {
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }
    // Poll.
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    /**
    *  重点关注这里，epoll_wait会等待管道事件，如果没有事件的话会等待输入
    */
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mPolling = false;

    // Acquire lock.
    mLock.lock();

    // Rebuild epoll set if needed.
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();
        goto Done;
    }

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
    #if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - timeout", this);
    #endif
    result = POLL_TIMEOUT;
    goto Done;
    }

    // Handle all events.
    #if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
    #endif

    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd.get()) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
    Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

    #if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
    ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
    this, handler.get(), message.what);
    #endif
    handler->handleMessage(message);
    } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
    #if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
    ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
    this, response.request.callback.get(), fd, events, data);
    #endif
    // Invoke the callback.  Note that the file descriptor may be closed by
    // the callback (and potentially even reused) before the function returns so
    // we need to be a little careful when removing the file descriptor afterwards.
    int callbackResult = response.request.callback->handleEvent(fd, events, data);
    if (callbackResult == 0) {
      removeFd(fd, response.request.seq);
    }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
  }
  ```
  方法里面关键的还是epoll_wait，下面的是拿到返回之后（也有可能超时返回），看这个是否正常，如果异常，或者超时，都会跳转到Gone的代码块。
  否则的话就是开始处理事件
  