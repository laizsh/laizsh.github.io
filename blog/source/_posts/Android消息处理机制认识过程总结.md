# Android消息处理机制认识过程总结

作为 Android 中最重要的组成部分之一，总结一下我对消息通信机制（ Handler、Looper ）的认识过程。

## Handler 通信基本原理

我刚接触 Android 的 Handler 时是处于一种**知其然不知其所以然**的状态，使用 Handler 也完全是因为子线程上更新 UI 会报错，接着上网查到可以使用 Handler 来传递到主线程更新 UI，然后就把代码 copy 到工程中使用，发现 work 后也就不深究了，当时就把「更新 UI 」和「 Handler 」绑定在一块了。

直到有一次面试，面试官问我 Handler 消息通信机制的原理，我傻傻地只懂把代码流程描述出来却无法解释原理时，才发现原来我知道的只是皮毛。

接着，我通过郭神的博客第一次比较全面的了解到Handler异步消息通信的原理([http://blog.csdn.net/guolin_blog/article/details/9991569](http://blog.csdn.net/guolin_blog/article/details/9991569))，才恍然大悟，才较清晰地明白 Thread、 Looper、 MessageQueue、 Handler、 Message 之间的关系， 包括：
1. 每个线程对应一个 Looper 和 一个 MessageQueue，而 Thread 和 Looper 的一一对应关系是通过 ThreadLocal 来实现的。
2. Handler 是对应创建该 Handler 的那个线程和该线程对应的 MessageQueue。Handler有两个功能：计划一些 Message 或者 Runnable 在未来的某一刻执行；将某个行为加入到不同线程中去执行。
3. 一个 Thread 可对应有 1 个 Looper，1 个 MessageQueue，多个 Handler， Looper 的职责是循环地检查一个线程对应的 MessageQueue 中的 Nessage， 逐个取出来在对应的 Handler 的 HandleMessage 中作对应响应，当然，我们自己创建的 Thread 默认情况下都是还没有 Looper 在运行的，需要调用 Looper.prepare() 以及 Looper.loop() 来。

这里盗用一下郭神的图：

![](http://img.blog.csdn.net/20130817090611984?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

理解了这个关系后，我才发现，Google 设计 Android 时是多么的贴心，让我们可以如此方便地使用 Handler 这种形象清晰的模式来进行线程间的异步通信以及计划任务。试想，如果没有 Handler 机制，我们还能使用什么方式来进行这种线程间的通信？愚蠢的我一下子也只能想到接收消息的线程要执行一个死循环来不停地检测一些共享变量(当然还要考虑变量的线程同步问题)，可想而知是多么的麻烦。

后来，我又了解到继承于 Thread 的 HandlerThread， 这是又一个贴心的设计，HandlerThread 在创建的时候已经自动创建并执行了 Looper，使用者也只需要创建对应的 Handler 来处理相关的数据就好了。这样，我们就可以创建一个带有 Handler 的子线程，这个子线程可以负责一些特定的任务，让其他线程来通知执行。这种方式让我在许多项目中应用来解耦功能，让一些相关的功能模块化和串行化，极大地提高了功能的内聚化和系统的可拓展性。

## 容易误解的地方

当我感觉一切都很顺利的时候，问题来了。

看过 Looper 源码的人都知道， Looper.loop() 方法中有个死循环 for(;;)

```java

public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);

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

如果真的是个简单的死循环，我那一丁点操作系统的基础告诉我，那该线程不就一直地会占用 CPU 吗？如果有好几个线程都是在死循环当中并且大部分时间都没有消息处理， 不就会导致 CPU 的消耗很高且不合理吗？

终于，虽然不影响使用，但这个问题开始困扰我了。

带着这个问题上网搜索，基本找不到答案... 大部分还是在详细地讲解 Handler 的基本通信模式和原理。

不过，幸运的是，，终于找到老罗的一篇文章有提到，[http://blog.csdn.net/luoshengyang/article/details/6817933](http://blog.csdn.net/luoshengyang/article/details/6817933)。文章里面有提到，在死循环的 Message msg = queue.next() 中，在 MessageQueue 的 next() 方法里在没有消息时会通过 epoll 机制的 epoll_wait 将线程挂起来进入休眠，并且通过管道写的方式来唤醒线程来处理新的消息，所以这段看似死循环的代码是会在无新消息处理时进入休眠状态的，因此并不会一直占用 CPU 。

另外可阅读下面相关文章来了解和区分一些概念：

1. 关于 epoll 机制的一篇文章 [http://www.nowamagic.net/academy/detail/13321005](http://www.nowamagic.net/academy/detail/13321005)
2. 关于 阻塞/非阻塞、同步/异步 知乎的一篇回答 [https://www.zhihu.com/question/19732473](https://www.zhihu.com/question/19732473)
3. 知乎中的一个相关问题 [Android中为什么主线程不会因为Looper.loop()里的死循环卡死？](https://www.zhihu.com/question/34652589)


## 总结

整个 Android 消息机制 的学习过程中，还是比较依赖别人的分析来学习，所以今后自己要多主动阅读源码来寻查原因，这样得着会更深刻。


