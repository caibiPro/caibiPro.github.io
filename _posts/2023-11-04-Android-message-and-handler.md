---
title: 理解Android消息系统：Handler, Looper, Message， Message-Queue
author: mingqing
date: 2023-11-04 17:19:00 +0800
categories: [Android]
tags: [android, handler, thread]
math: true
mermaid: true
---

## Android中的线程

当应用在 Android 中启动时，它会创建第一个执行线程，称**为“主”线程**。主线程负责将事件分派到相应的用户界面小组件，以及与 Android UI 工具包中的组件进行通信。

为了保持应用程序的响应速度，必须**避免使用主线程来执行任何可能最终导致其阻塞的操作**。

**网络操作**和**数据库调用**，以及**某些组件的加载**，都是在主线程中应避免的常见操作示例。当它们在主线程中被调用时，它们会被同步调用，这意味着在操作完成之前，UI 将保持完全无响应。因此**，它们通常在单独的线程中执行**，从而避免在执行时阻塞 UI（即，它们已从 UI 异步执行）

> Categorize all threading components into two basic categories:

> 将所有线程组件分为两个基本类别：

- **Threads that *are* attached to an activity/fragment:** These threads are tied to the lifecycle of the activity/fragment and are terminated as soon as the activity/fragment is destroyed. （附加到 Activity/Fragment 的线程：这些线程与 Activity/Fragment 的生命周期相关联，并在 Activity/Fragment 被销毁后立即终止。）

- **Threads that *are not* attached to any activity/fragment:** These threads can continue to run beyond the lifetime of the activity/fragment (if any) from which they were spawned.（未附加到任何 Activity/Fragment 的线程：这些线程可以在生成它们的 Activity/Fragment（如果有）的生存期之后继续运行。）

## Communicate with the UI thread from another thread

由于只有一个线程更新 UI，即主线程，我们使用不同的其他线程在后台执行多个任务，但最后，要更新 UI，我们需要将结果发布到主线程或 UI 线程。

如果要管理一大组线程，则管理与所有这些线程的通信将很复杂。因此，Android 提供了处理程序，使**进程间通信**更容易。

所以Handler 机制是 Android 中最重要的**异步通信**机制，即将需要执行的任务分配给其它线程：

1. **子线程更新 UI** : 在子线程中更新 UI , 就是在子线程中将刷新 UI 的任务分配给了主线程 ; ( 子线程刷新 UI 会崩溃 )
2. **主线程网络操作** : 在主线程中 , 将网络通信等耗时的操作分配给子线程 ( 该子线程需要转成 Looper 线程 ) , 避免 UI 卡顿 ; ( 主线程访问网络会崩溃 )

### Handler 机制中涉及到的组件 :

1. Handler ( 消息处理者 ) : 定义具体的代码操作逻辑 , 处理收到消息 ( Message ) 后的具体操作 ;
2. Message ( 消息 ) : 定义具体消息 , 其中可以封装不同的变量 , 为 Handler 指定操作的类型 , 或执行操作所需的数据 ;
3. Looper ( 消息遍历者 ) : 消息的遍历者 , 遍历 MessageQueue 中的消息 , 分发给 Handler 处理 ;
4. MessageQueue ( 消息队列 ) : 封装在 Looper 中 , 每个 Looper 中封装了一个 MessageQueue , 是 Looper 消息遍历的重要组件 , 用户不直接调用该组件 ;

先看下这几个类的关系，MessageQueue是一个包含了Message的队列。一个Looper中包含有一个MessageQueue，Message中有对Handler的引用。

### Handler 机制中的封闭性与线程交互

1. **线程内部相对封闭的运行系统** : 整个 Looper 线程内部是一个封闭运行的系统 , Looper 一直不停的再遍历 MessageQueue , 将 消息 或 操作 取出 , 交给 Handler 执行 ;
2. **线程交互** : Handler 还有另外一个职责就是负责与外部线程的交互 , 在外部线程中调用 Handler 将消息回传给本 Looper 线程 , 放入 MessageQueue 队列中 ;

## Looper

A Looper is responsible for managing a thread’s message queue. It processes messages in the order they’re received and dispatches them to the appropriate Handler. A Looper is usually associated with a single thread, and each thread can have at most one Looper.

要点：

1. 一个android的主线程中有且仅有一个Looper，当程序启动时这个looper就开始不断的从MessageQueue里取出消息来进行处理。应该是一个while的循环。当没有消息时，messageQueue.next()就处于阻塞状态，直到有新的消息取出。
2. 按照消息的接收顺序处理消息，并将其分派给相应的Handler
3. 新开一个线程是默认是没有Looper的。但是也可以给它加一个Looper，有了Looper这个线程就可以进行消息处理了。
4. 在同一个app中，其它线程可以取的主线程的Looper，这样就可以实现子线程和主线程之间的通信。

### 创建Looper

Looper的字面意思是“循环者”，它被设计用来使一个普通线程变成**Looper线程**。所谓Looper线程就是循环工作的线程。在程序开发中（尤其是GUI开发中），我们经常会需要一个线程不断循环，一旦有新任务则执行，执行完继续等待下一个任务，这就是Looper线程。使用Looper类创建Looper线程很简单：

```java
publicclass LooperThread extends Thread {
    @Override
    publicvoid run() {
        // 将当前线程初始化为Looper线程
        Looper.prepare();
        // ...其他处理，如实例化handler
        // 开始循环处理消息队列
        Looper.loop();
    }
}
```

通过上面两行核心代码，你的线程就升级为Looper线程了！！！是不是很神奇？让我们放慢镜头，看看这两行代码各自做了什么。

普通子线程 转为Looper子线程 代码示例 :

```kotlin
package kim.hsl.handler;

import android.os.Handler;
import android.os.Looper;
import android.os.Message;

import androidx.annotation.NonNull;

/**
 * 将普通线程转为 Looper 线程
 *
 * 1. 定义 Handler 成员 ( 可以定义若干个 )
 * 2. Looper.prepare()
 * 3. 实例化 Handler 成员
 * 4. Looper.loop()
 *
 * @author hsl
 */
public class LooperThread extends Thread {
    /**
     * 1. 定义时不要实例化
     * Handler 实例化需要关联 Looper 对象
     * Looper 对象在 Looper
     */
    private Handler handler;

    @Override
    public void run() {
        super.run();
        
        //2. 将线程转为 Looper 线程
        //主要是创建 Looper 放入 ThreadLocal 对象中
        Looper.prepare();

        //3. 创建 Handler 必须在 Looper.prepare() 之后, 否则会崩溃
        handler = new Handler(){
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                //TODO 处理 Handler 相关业务逻辑
            }
        };

        //4. Looper 开始轮询 MessageQueue, 将消息调度给 Handler 处理
        Looper.loop();
    }

    public Handler getHandler() {
        return handler;
    }

    public void setHandler(Handler handler) {
        this.handler = handler;
    }
}
```

### Looper.prepare()

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/56D977E5-3073-4CDE-94E0-531A82E602E9_2/nL5zGI7C6ZEi8OyobxrqdmVQ0UM4XRhT0KyZt2Co7P0z/Image.png)

通过上图可以看到，现在你的线程中有一个Looper对象，它的内部维护了一个消息队列MQ。注意，**一个Thread只能有一个Looper对象**，为什么呢？咱们来看源码。

通过源码，prepare()背后的工作方式一目了然，其核心就是将**looper对象定义为ThreadLocal**。如果你还不清楚什么是ThreadLocal，请参考[《理解ThreadLocal》](http://blog.csdn.net/qjyong/article/details/2158097)。

```java
publicclass Looper {
    // 每个线程中的Looper对象其实是一个ThreadLocal，即线程本地存储(TLS)对象
	privatestaticfinal ThreadLocal sThreadLocal =new ThreadLocal();
    // Looper内的消息队列
	final MessageQueue mQueue;
    // 当前线程
    Thread mThread;
    // 其他属性...
    // 每个Looper对象中有它的消息队列，和它所属的线程
	private Looper() {
        mQueue =new MessageQueue();
        mRun =true;
        mThread = Thread.currentThread();
    }
    // 我们调用该方法会在调用线程的TLS中创建Looper对象

	public static final void prepare() {
        if (sThreadLocal.get() !=null) {
            // 试图在有Looper的线程中再次创建Looper将抛出异常
			throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper());
    }
    // 其他方法
}
```

### Looper.loop()

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/A48DBC76-B1E8-484D-8F61-DD77419EFA4A_2/rYBCqlAfBKw4XXcxOUNCgNb5MGxChTaxeyFeeQXMCsoz/Image.png)

调用loop方法后，Looper线程就开始真正工作了，它不断从自己的MQ中取出队头的消息(也叫任务)执行。其源码分析如下：

```java
public static final void loop(){
    Looper me=myLooper();  //得到当前线程Looper
	MessageQueue queue=me.mQueue;  //得到当前looper的MQ
	// 这两行没看懂= = 不过不影响理解
	Binder.clearCallingIdentity();
	finallong ident=Binder.clearCallingIdentity();
	// 开始循环
	while(true){
		Message msg=queue.next(); // 取出message
		if(msg!=null){
			if(msg.target==null){
        		// message没有target为结束信号，退出循环
        		return;
            }
        	// 日志
			if(me.mLogging!=null) {
				me.mLogging.println(
        		">>>>> Dispatching to "+msg.target+""
        		msg.callback+": "+msg.what);
            }
        	// 非常重要！将真正的处理工作交给message的target，即后面要讲的handler
        	msg.target.dispatchMessage(msg);
        	// 还是日志
       		if(me.mLogging!=null) {
				me.mLogging.println(
        		"<<<<< Finished to    "+msg.target+""
        		+msg.callback);
            }
        	// 下面没看懂，同样不影响理解
        	finallong newIdent=Binder.clearCallingIdentity();
       		if(ident!=newIdent){
        		Log.wtf("Looper","Thread identity changed from 0x"
        		+Long.toHexString(ident)+" to 0x"
				+Long.toHexString(newIdent)+" while dispatching to "
				+msg.target.getClass().getName()+""
				msg.callback+" what="+msg.what);
            }
        	// 回收message资源
        	msg.recycle();
        }
    }
}
```

除了prepare()和loop()方法，Looper类还提供了一些有用的方法，比如

### Looper.myLooper()

获取了与当前线程关联的 Looper

```java
public static final Looper myLooper() {
	// 在任意线程调用Looper.myLooper()返回的都是那个线程的looper
	return (Looper)sThreadLocal.get();
}
```

### getThread()

得到looper对象所属线程：

```java
public Thread getThread() {
	return mThread;
}
```

### quit()

结束looper循环：

```java
public void quit() {
	// 创建一个空的message，它的target为NULL，表示结束循环消息
	Message msg = Message.obtain();
	// 发出消息
	mQueue.enqueueMessage(msg, 0);
}
```

## Handler

A Handler is a class that is responsible for processing messages and running them on the appropriate thread. A Handler can be associated with a specific thread and can post messages to that thread’s message queue. Handlers are often used in Android to update the UI from a background thread.

什么是handler？handler扮演了往MQ上添加消息和处理消息的角色（只处理由自己发出的消息），即**通知MQ它要执行一个任务(sendMessage)，并在loop到自己的时候执行该任务(handleMessage)，整个过程是异步的**。handler创建时会关联一个looper，默认的构造方法将关联当前线程的looper，不过这也是可以set的。默认的构造方法：

```java
public class handler {
	final MessageQueue mQueue;  // 关联的MQ
	final Looper mLooper;  // 关联的looper
	final Callback mCallback; 
 	// 其他属性
	public Handler() {
        // 没看懂，直接略过，，，

		if (FIND_POTENTIAL_LEAKS) {
			final Class<?extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) ==0) {
				Log.w(TAG, "The following Handler class should be static or leaks might occur: "+klass.getCanonicalName());
            }
        }
        // 默认将关联当前线程的looper
        mLooper = Looper.myLooper();
        // looper不能为空，即该默认的构造方法只能在looper线程中使用
		if (mLooper ==null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }

        // 重要！！！直接把关联looper的MQ作为自己的MQ，因此它的消息将发送到关联looper的MQ上
        mQueue = mLooper.mQueue;
        mCallback =null;
    }
    // 其他方法
}
```

要点：

1. 与特定线程相关联（主要是Looper所在线程），并post/send消息到该特定线程上的消息队列上。（也就是Looper关联的Message-Queue中）
2. 主要应用场景：从后台线程来更新界面

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/046A48EA-C35D-4831-A60F-BCE33EBE606C_2/cTItEcn1U3e05XH48uyZUYN6gVzdnvfFsIvtUKM4kxgz/Image.png)

下面我们就可以为之前的LooperThread类加入Handler：

```java
public class LooperThread extends Thread {
    private Handler handler1;
    private Handler handler2;
    @Override
    publicvoid run() {
		// 将当前线程初始化为Looper线程
        Looper.prepare();
        // 实例化两个handler
        handler1 = new Handler();
        handler2 = new Handler();
        // 开始循环处理消息队列
        Looper.loop();
    }
}
```

加入handler后的效果如下图：可以看到，**一个线程可以有多个Handler，但是只能有一个Looper！**

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/A73040A6-DBE2-4602-A8E2-361C568713A3_2/ySxFrlJ5VzJw06vu6bYHCrWLBvYYewakBx4TXEkFgIgz/Image.png)

### 创建Handler

创建了一个与当前线程关联的新 Handler 对象

```java
Handler handler = new Handler();
```

### Handler发送消息

有了handler之后，我们就可以使用下列方法向MQ上发送消息了：

Handler 既可以发送**静态的**消息 ( Message ) , 又可以发送**动态的**操作 ( Runnable ) ;

1. 当 Handler 所在的 Looper 线程接收到消息 ( Message ) 时 , Looper 轮询到该 消息 ( Message ) 后 , 执行该消息对应的业务逻辑 , 这些逻辑一般都是在 Handler 中提前定义好的 ;
2. 当 Handler 所在的 Looper 线程接收到操作 ( Runnable ) 时 , Looper 轮询到该 操作 ( Runnable ) 后 , 直接运行该 Runnable 中的 run() 方法 ;
- `post(Runnable)`
- `postAtTime(Runnable, long)`
- `postDelayed(Runnable, long)`
- `sendEmptyMessage(int)`
- `sendMessage(Message)`
- `sendMessageAtTime(Message, long)`
- `sendMessageDelayed(Message, long)`

光看这些API你可能会觉得handler能发两种消息，一种是Runnable对象，一种是message对象，这是直观的理解，但其实post发出的Runnable对象最后都被封装成message对象了，见源码：

```java
// 此方法用于向关联的MQ上发送Runnable对象，它的run方法将在handler关联的looper线程中执行
public final boolean post(Runnable r) {
	// 注意getPostMessage(r)将runnable封装成message
	return  sendMessageDelayed(getPostMessage(r), 0);
}

private final Message getPostMessage(Runnable r) {
	Message m = Message.obtain();  //得到空的message
	m.callback = r;  //将runnable设为message的callback，
	return m;
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	boolean sent =false;
   	MessageQueue queue = mQueue;
	if (queue != null) {
		msg.target =this;  // message的target必须设为该handler！
		sent = queue.enqueueMessage(msg, uptimeMillis);
    } else {
		RuntimeException e =new RuntimeException(
			this+" sendMessageAtTime() called with no mQueue"
		Log.w("Looper", e.getMessage(), e
	}
	return sent;
}
```

其他方法就不罗列了，总之通过handler发出的message有如下特点：

1. message.target为该handler对象，这确保了looper执行到该message时能找到处理它的handler，即loop()方法中的关键代码

```java
msg.target.dispatchMessage(msg);
```

2. post发出的message，其callback为Runnable对象

### Handler处理消息

说完了消息的发送，再来看下handler如何处理消息。消息的处理是通过核心方法`dispatchMessage(Message msg)` 与钩子方法 `handleMessage(Message msg)` 完成的：

```kotlin
// 处理消息，该方法由looper调用
public void dispatchMessage(Message msg){
    if (msg.callback != null) {
        // 如果message设置了callback，即runnable消息，处理callback！
        handleCallback(msg);
    } else {
        // 如果handler本身设置了callback，则执行callback
        if (mCallback != null) {
        /* 这种方法允许让activity等来实现Handler.Callback接口，避免了自己编写handler重写handleMessage方法。见http://alex-yang-xiansoftware-com.iteye.com/blog/850865 */
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 如果message没有callback，则调用handler的钩子方法handleMessage
        handleMessage(msg);
    }
}

// 处理runnable消息
private final void handleCallback(Message message){
    //直接调用run方法！
    message.callback.run();
}
// 由子类实现的钩子方法
public void handleMessage(Message msg){}
```

可以看到，除了[`handleMessage`](http://developer.android.com/reference/android/os/Handler.html#handleMessage%28android.os.Message%29)`(`[`Message`](http://developer.android.com/reference/android/os/Message.html)` msg)`和Runnable对象的run方法由开发者实现外（实现具体逻辑），handler的内部工作机制对开发者是透明的。这正是handler API设计的精妙之处！

### Handler的用处

Handler拥有下面两个重要的特点：

1. handler可以在**任意线程发送消息**，这些消息会被添加到关联的MQ上

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/660ABCED-AAC6-4A32-A4C2-EA0AE164093B_2/BSfq2ciItMn3x4xKMioK0RgEBd4xxiayvLFRuoQZW38z/Image.png)

2. handler是在它**关联的looper线程中处理消息**的

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/A0BC4754-B04B-44DD-A296-1940C217D88C_2/asWy15cmAfQoLxZ7R7IZ8XcZdqkrVMph8WBbWzcFaKEz/Image.png)

这就解决了android最经典的不能在其他非主线程中更新UI的问题。**android的主线程也是一个looper线程**（looper在android中运用很广），我们在其中创建的handler默认将关联主线程MQ。因此，利用handler的一个solution就是在activity中创建handler并将其引用传递给worker thread，worker thread执行完任务后使用handler发送消息通知activity更新UI。(过程如图)

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/B2732B49-8A5E-4FD0-BB9E-076617BE7A86/309D1CC7-3871-4A26-BD55-06C39EDFAD72_2/8Ri1XExJRmWE4r8bM83rz194x31FsvtVm7yjqWp7cm4z/Image.png)

下面给出sample代码，仅供参考：

Handler:

```kotlin
package com.mingqing.broadcastbestpractice;

import android.os.Handler;
import android.os.Looper;

class MyHandler extends Handler {

    public MyHandler(Looper looper, Callback callback) {
        super(looper, callback);
    }
}
```

Activity:

```kotlin
package com.mingqing.broadcastbestpractice;

import com.mingqing.broadcastbestpractice.databinding.ActivityTestDriverBinding;

import android.app.Activity;
import android.os.Bundle;
import android.os.Looper;
import android.widget.TextView;

public class TestDriverActivity extends Activity {
    private TextView textview;
    private ActivityTestDriverBinding binding = null;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // 视图绑定
        super.onCreate(savedInstanceState);
        binding = ActivityTestDriverBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
        textview = binding.textView;

        // 创建并启动工作线程
        Looper mainLooper = getMainLooper();

        // Handler与主线成的Looper关联，并定义好消息处理的回调函数
        MyHandler myHandler = new MyHandler(mainLooper, msg -> {
            String result = msg.getData().getString("message");
            // 更新UI
            appendText(result);
            return true;
        });

        // 开启工作线程并传入Handler的引用，实现Long Operation的异步处理获取数据，并在主线程更新UI
        Thread workerThread = new Thread(new SampleTask(myHandler));
        workerThread.start();
    }

    public void appendText(String msg) {
        textview.setText(textview.getText() + "\n" + msg);
    }
}
```

Work Thread:

```kotlin
package com.mingqing.broadcastbestpractice;

import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.util.Log;

public class SampleTask implements Runnable {
    private static final String TAG = SampleTask.class.getSimpleName();
    private final Handler handler;

    public SampleTask(Handler handler) {
        super();
        this.handler = handler;
    }

    @Override
    public void run() {
        try {
            // 模拟执行某项任务，下载等
            Thread.sleep(5000);
            // 任务完成后通知activity更新UI
            Message msg = prepareMessage("task completed!");
            // message将被添加到主线程的MQ中
            handler.sendMessage(msg);
        } catch (InterruptedException e) {
            Log.d(TAG, "interrupted!");
        }

    }

    private Message prepareMessage(String str) {
        Message result = handler.obtainMessage();
        Bundle data = new Bundle();
        data.putString("message", str);
        result.setData(data);
        return result;
    }
}
```

当然，handler能做的远远不仅如此，由于它能post Runnable对象，它还能与Looper配合实现经典的Pipeline Thread(流水线线程)模式。请参考此文[《Android Guts: Intro to Loopers and Handlers》](http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/)

## Message

在整个消息处理机制中，message又叫task，封装了任务携带的信息和处理该任务的handler。message的用法比较简单，这里不做总结了。但是有这么几点需要注意（待补充）：

1. 尽管Message有public的默认构造方法，但是你应该通过`Message.obtain()`来从消息池中获得空消息对象，以节省资源。
2. 如果你的message只需要携带简单的int信息，请优先使用`Message.arg1`和`Message.arg2`来传递信息，这比用Bundle更省内存
3. 擅用`message.what`来标识信息，以便用不同方式处理message。

## MessageQueue

A MessageQueue is responsible for holding messages that are waiting to be processed. Messages are added to the MessageQueue using the Handler’s post() method. The Looper processes messages from the MessageQueue in the order they were added.

MessageQueue MessageQueue 负责保存等待处理的消息。使用 Handler 的 post() 方法将消息添加到 MessageQueue 中。Looper 按照添加消息的顺序处理来自 MessageQueue 的消息。

### 创建MessageQueue

获取了与当前线程的 Looper 关联的 MessageQueue。

```java
MessageQueue queue = Looper.myQueue();
```

## 简单示例

创建一个每秒更新 UI 的新线程。我们可以使用 Handler、Looper 和 MessageQueue 来实现这一点

```java
class MyThread extends Thread {
    private Handler mHandler;   

 public void run() {
   Looper.prepare(); 
      
   mHandler = new Handler() {
            public void handleMessage(Message msg) {
                // Update UI here
            }
        };
   Looper.loop();
 }

 public void postUpdate() {
     mHandler.postDelayed(new Runnable() {
         public void run() {
                // Create message and add to MessageQueue
              Message msg = mHandler.obtainMessage();
               mHandler.sendMessage(msg);
                // Post another update after 1 second
                postUpdate();
            }
        }, 1000);
    }
}

MyThread thread = new MyThread();
thread.start();
thread.postUpdate();
```

在上面的示例中，我们创建了一个名为 MyThread 的新线程。在 MyThread 的 run() 方法中，我们将创建一个新的 Handler 和一个新的 Looper。我们还将重写 Handler 的 handleMessage()方法来更新 UI。

在 MyThread 的 postUpdate()方法中，我们使用 postDelayed() 方法将新消息发布到处理程序的 MessageQueue。我们还会在一秒钟后再次调用 postUpdate()以继续更新 UI。
