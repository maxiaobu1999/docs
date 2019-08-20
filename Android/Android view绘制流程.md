# Android view绘制流程

## ViewRoot

proformTravals():入口

View mView:DecorView，Activity 中的顶级 View

WindowManagerGlobal:进程单例，存储所有 Window 所对应的 ViewRootImpl

Surface mSurface :画布，SurfaceFling使用

## MeasureSpec

MeasureSpec表示的是一个32位的整形值，它的高2位表示测量模式SpecMode，低30位表示某种测量模式下的规格大小SpecSize

> - EXACTLY
> - AT_MOST
> - UNSPECIFIED

```
LayoutParams
getMeasuredWidth()
getWidth()
```

## draw

requestLayout()

Involite()

画图四要素：canvas paint bitmap

dispatchDraw onDraw :viewGroup没有背景只调用dispatchDraw()

##SurfaceFlinger & Surface

SurfaceFlinger:混合数据

surface：提供像素信息

# 内存

oom是什么: 

> Heap.cpp
>
> 分配内存？gc 〉增大内存》gc 带弱引用〉oom
>
> tig：手动调用System.gc() 没用！ 分配内存的时候自然会GC

gc：

>  引用计数法:加一减一tip：循环引用iOS
>
> 标记回收算法：标记GC Roots引用到的对象，其余回收，tip：内存碎片
>
> 复制算法：划分一块空内存块，发现存活直接拷贝进对象，清除原内存块，tip：快，空间换时间
>
> 标记-压缩算法：遍历标记，一起移动存活对象，清除其他：慢时间换空间
>
> 分代：青年代 老年代
>
> 并发GC：都会stop world，一次回收一部分，反复执行

内存泄露

> 静态 内部类 context 
>
> handle thread
>
> 像素信息存放位置recycle()

内存抖动：对象池（Message#obtain()，listview）

内存优化：少用+复用+释放+扩容

少用

>  编码（565 8888）、采样（inSampleSize）
>
> 

复用

> inBitmap 线程池  byte[]数组池
>
> 

释放：内存泄露

扩容：多进程、largeheap配置、匿名共享内存

# mvp

mvc衍生、架构模式。presenter： middle-man 处理表现层逻辑



View model层解藕，即vp相互依赖，通常

一个activity多个presenter

一个class定义view presenter两个接口

# AsyncTask

线程池+FutureTask + Callable:可以cancel（）基于interrupt()，可以get（）阻塞

串行SerialExecutor：Deque 保存task 执行完一个再取一个

# 启动优化

工具：ADT》traceview  AS〉profiler 耗电

activityThread#main() ->attachBaseContext()->ContentPrivude初始化-> onCreate->activity#create、start、resume-》viewRoot#performTraversal()

ContentProvide mulitDex TAG反射27ms 运行时注解、json（编译时注解）

加快布局渲染

> 优化布局include merge ViewStub 约束布局  

> 代码布局：Java替换xml  闪屏页 Java写view直接setContent(View view)

> theme优化：title background不要

1、异步2、线程池3、延时 dispatchDraw中view#post 或IdelHandle 或直接view#post

主线成挂起

> 跨进程调用 启动service

线程池优先队列 延时初始化

应用秒开

>  背景样式与启动页保持一致



# ANR

导出Trace文件  data/anr/traces_XXX.txt

分类：

带行号，其他线程，iowait？内存

InputDispatcher

> findFocusedWindowTargetsLocked:寻找聚焦窗口失败的情况：
>
> - 无窗口，无应用：Dropping event because there is no focused window or focused application.(这并不导致ANR的情况，因为没有机会调用handleTargetsNotReadyLocked)
>
> - 无窗口, 有应用：Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.
>
> checkWindowReadyForMoreInputLocked
>
> ```c++
> // 非按键事件，事件等待队列不为空且头事件分发超时500ms
>         if (!connection->waitQueue.isEmpty()
>                 && currentTime >= connection->waitQueue.head->deliveryTime
>                         + STREAM_AHEAD_EVENT_TIMEOUT) {
>             return String8::format("Waiting to send non-key event because the %s window has not "
>                     "finished processing certain input events that were delivered to it over "
>                     "%0.1fms ago. Wait queue length: %d. Wait queue head age: %0.1fms.",
>                     targetType, STREAM_AHEAD_EVENT_TIMEOUT * 0.000001f,
>                     connection->waitQueue.count(),
>                     (currentTime - connection->waitQueue.head->deliveryTime) * 0.000001f);
>         }
> ```
>
> 

# 数据结构

hashmap 数据+链表/红黑树 扩容因子 复制

linkedHashMap

concurrentHashMap 分段锁



# handler



# view#post

 https://blog.csdn.net/scnuxisan225/article/details/49815269



    private void performTraversals() {
            // cache mView since it is used so much below...
            final View host = mView;
    // 这里面做了一些初始化的操作，第一次执行和后面执行的操作不一样，这里不关
    // 心过多的东西，主要关心attachInfo在此处被初始化完成
    
        // Execute enqueued actions on every traversal in case a detached view enqueued an action
        getRunQueue().executeActions(attachInfo.mHandler);
    
    ...
    performMeasure();
    ...
    performLayout();
    ...
    performDraw();
    }

getRunQueue().executeActions(attachInfo.mHandler);用handle发消息取执行

所以调用在三方法前，执行在三方法后



# 插件化

问题

> 默认的ClassLoader只加载应用安装目录的class文件，不能加载其他apk
>
> 清单文件没注册，PMS中没有组件信息
>
> 没有context，Resource 

应用加载流程：

>  开机PMS扫描应用目录拿appInfo（清单文件、apk安装路径、app存储路径、so路径、resource。。）
>
> AMS通过PMS拿info，执行ActivityThread#main()，
>
> 创建ClassLoader Resource（通过PMS创建） +appInfo合成Context（解决id重名）

ClassLoader：

> 替换PathClassLoader：系统提供，篡改loadClass() 反射运行时属性
>
> AMS通过PMS拿appInfo（apk路径、app存储路径、so路径、resource。。）-》创建context-〉context包含ClassLoader

坑位：映射分配坑位-》启动坑位aty-〉hook ClassLoader-》加载目标aty

context: Aty#attachBaseContext()中替换 resource重名

menifest 资源id so

# 3体积优化

图片 so 一套资源

enum：一个增加1～1.5kb

