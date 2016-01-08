---
title:  "Android中Handler Looper源码解析"
date:   2016-01-04 21:30:00 +0800
tags: [Android]
---
## 问题来源

> 所有的模式都是为了解决特定的问题。现在的问题是什么呢？

`Android` `Main Thread` 必须要时刻监听响应用户操作，所以所有需要长时间或者无法保证具体时间（例如网络操作）的操作都不能放在主线程，必须有后台线程（`worker threads`）处理。否则，主线程被阻塞无法响应，最终会提示~~应用程序×××无法响应~~。只要稍一联想这种解决问题的模式和`javaScript`的[Event Looper](http://www.ruanyifeng.com/blog/2013/10/event_loop.html)思路完全一样。

### 解决模式

> `Event Looper`有很多这种解决方式，`线程池`、`生产者消费者队列`等。

`Android`提供的是`单生产者单消费者`模式的解决方式。顾名思义：在`Main Thread`把待解决的任务所需要的数据推送给`Worker Thread`的`Message Queue`，然后再`Worker Thread`中解决问题，最后如果需要的话把处理结果再推送回`Main Thread`的`Message Queue`

### 源码解析

> 全局的看这个`生产者消费者`模式的实现。`android.os.HandlerThread`实现`Worker Thread`；`android.os.MessageQueue`实现`消息队列`;`android.os.Message`实现传送给`消费者`的数据；`android.os.Handler`实现`消费者`处理逻辑；`android.os.Looper`全局调度。

- 首先看启动类`android.os.HandlerThread`的`run`方法（移除我们不关心的代码）：
  
  {% highlight java linenos%}
  public void run() {
          Looper.prepare();
          synchronized (this) {
              mLooper = Looper.myLooper();
              notifyAll();
          }
          onLooperPrepared();
          Looper.loop();
      }
  {% endhighlight %}
  
  首先`Looper.prepare（）`，查看代码（`android.os.Looper#prepare(boolean)`）发现是`sThreadLocal.set(new Looper(quitAllowed))`,
  
  实现线程`Looper`绑定到`Worker Thread`。然后把`Worker Thread`的`Looper`放到于变量中，下面`onLooperPrepared（）`是我们子类必须要继承的方法，在这里要实现`Worker Thread`级别的`Handler`。最终调用核心方法`Looper.loop()`实现整个逻辑的结合。我们看下这个方法（略过部分代码）：
  
  {% highlight java linenos%}
  public static void loop() {
          final Looper me = myLooper();
          final MessageQueue queue = me.mQueue;
           for (;;) {
              Message msg = queue.next(); // might block
            
              msg.target.dispatchMessage(msg);
  
          }
      }
  {% endhighlight %}
  
  很明显的逻辑，循环取出`MessageQueue`的`Message`，然后调用`msg.target`（即`Handler`处理）。而`dispatchMessage`中要么是`callback`要么是`handleMessage`所以我们必须实现其一。例子：
  
  {% highlight java linenos%}
  //msg.target
  Handler handler = new Handler();
  handler.post(new Runable(){
    // Your logic
  });
  {% endhighlight %}
  
  或者
  
  {% highlight java linenos%}
  //handleMessage
  Handler handler = new Handler(){
    public void handleMessage(Message msg) {
      //Your logic  
    }
  };
  {% endhighlight %}
  
  **这2个逻辑的`new`逻辑必须在前面提到的`onLooperPrepared`，因为只有如此，`handler`才是`Worker Thread`级别的对象**
  
  还剩下`Message`没有提到。
  
  `Message`必须要绑定一个`Handler`作为`target`，以便作为回调（`msg.target.dispatchMessage`）。但是我们打开`Handler`源码的`obtainMessage`相关的5个重载方法，这个绑定已被实现。所以你只需传你的`what`(消息code)或者（和）`obj`(消息实体)。
  
  完整的一个下载图片的例子：
  
  {% highlight java linenos%}
  public class DownloadHandlerThread<T> extends HandlerThread {
  
      public static final String TAG = "DownloadHandlerThread";
      private static final int DOWNLOAD_CODE = 1;
  
      private Handler mHandler;
      private Process<T> mProcess;
      private Handler mMainHandler;
      private DownloadListen mDownloadListen;
  
      public interface DownloadListen<T> {
          void downloadListen(T target, Bitmap bitmap);
      }
  
      public DownloadHandlerThread(Process<T> process) {
          super(TAG);
          mProcess = process;
          mMainHandler = new Handler();
      }
  
      @Override
      protected void onLooperPrepared() {
          super.onLooperPrepared();
          mHandler = new Handler() {
              @Override
              public void handleMessage(final Message msg) {
                  super.handleMessage(msg);
                  if (msg.what == DOWNLOAD_CODE) {
                      final Object target = msg.obj;
                      final Bitmap bitmap = mProcess.process((T) target);
                      //这里发送到到主线程
                      mMainHandler.post(new Runnable() {
                          @Override
                          public void run() {
                              if (mDownloadListen != null) {
                                  mDownloadListen.downloadListen(target, bitmap);
                              }
                          }
                      });
                  }
              }
          };
      }
  
      public void sendMessage(T target) {
          Message.obtain(mHandler, DOWNLOAD_CODE, target).sendToTarget();
      }
  	//主线程回调逻辑
      public void setDownloadListen(DownloadListen downloadListen) {
          mDownloadListen = downloadListen;
      }
  }
  {% endhighlight %}
  
  `Activity `的`onCreate`方法调用：
  
  {% highlight java linenos%}
  mPhotoViewHolderDownloadHandlerThread = new DownloadHandlerThread<>(new Process<PhotoViewHolder>() {
              @Override
              public Bitmap process(PhotoViewHolder target) {
                  String url = target.getPhoto().getUrl();
                  if (!Strings.isNullOrEmpty(url)) {
                      final byte[] bytes = PhotoFetcher.getBytes(url);
                      return BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
                  }
                  return null;
              }
          });
          mPhotoViewHolderDownloadHandlerThread.setDownloadListen(new DownloadHandlerThread.DownloadListen<PhotoViewHolder>() {
              @Override
              public void downloadListen(PhotoViewHolder target, Bitmap bitmap) {
                  if (bitmap != null) {
                      target.getImageView().setImageBitmap(bitmap);
                  }
              }
          });
          mPhotoViewHolderDownloadHandlerThread.start();
  {% endhighlight %}
  
  发送消息：
  
  {% highlight java linenos%}
  mPhotoViewHolderDownloadHandlerThread.sendMessage(holder);
  {% endhighlight %} 
  

## 源码应用

  > 这一段逻辑广泛应用，见`android.os.Handler#post`的call hierarchy😀。
  a{{site.baseurl}}a
  ![call hierarchy](/assets/images/post_call_hierarchy.png)













