title: Android Touch事件传递总结
date: 2016/5/19 20:45:37
categories: 
tags: Android
---

Android View的Touch事件传递主要涉及三个方法：

- boolean dispatchTouchEvent(MotionEvent ev)

	负责事件的分发，返回值的确定很复杂，涉及是否绑定OnTouchListener、onTouch方法返回值、onTouchEvent返回值，对于ViewGroup与onInterceptTouchEvent返回值有关。
 
- boolean onInterceptTouchEvent(MotionEvent ev)

	是否拦截事件。只有ViewGroup有该方法，因为ViewGroup一般会包含子View，有时需要考虑对Touch事件的拦截。
 
- boolean onTouchEvent(MotionEvent event)

	消费Touch事件。返回值决定Touch事件是否向上传递给父ViewGroup。
	
<!--more-->

看上面的描述肯定不懂，自己写个例子试验一下呗。

![](http://7xoze0.com1.z0.glb.clouddn.com/android-view-touch.png?imageView2/2/w/400)

ViewGroupA和ViewGroupB继承自ViewGroup，作为两个自定义的View容器。
红色部分的View直接继承自View类。
在这三个自定义类中重写前面提到的相应方法，默认只打Log，同样给Activity的dispatchTouchEvent和onTouchEvent打Log，在不同情形下来看看输出。

## 一、默认情形
- 点击一下红色区域即ViewContent
- 默认都不拦截
- 默认都不消费

```
D/MainActivity: MainActivity dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onInterceptTouchEvent ACTION_DOWN
D/ViewContent: ViewContent dispatchTouchEvent ACTION_DOWN
D/ViewContent: ViewContent onTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity dispatchTouchEvent ACTION_UP
D/MainActivity: MainActivity onTouchEvent ACTION_UP
```
调用顺序出来了，一个完整的Touch事件流由DOWN事件开始，经历0个或若干个MOVE事件（该例中未移动），最后UP事件结束。

事件的分发由上至下，事件到达某一层时先调用dispatchTouchEvent，由其决定将事件分发给哪个方法处理。

对于ViewGroup会传递给该层的onInterceptTouchEvent决定是否拦截，对于View则直接传递给onTouchEvent决定是否消费，该例中各层的onTouchEvent都返回默认false即都不消费事件，则事件会由子View的onTouchEvent向上传递至上层的onTouchEvent。

该例中的DOWN事件最终由MainActivity的onTouchEvent消费了，当后续的UP事件到来时，MainActivity的dispatchTouchEvent直接把UP事件交由其onTouchEvent处理。

示意图如下：
![](http://www.trinea.cn/wp-content/uploads/2016/01/touch1.jpg?dc9529&b1d01c)

## 二、某一层消费事件（onTouchEvent返回true）
- 让ViewGroupB即蓝色区域接收到DOWN事件后，onTouchEvent返回true，即ViewGroupB消费了DOWN事件
- onInterceptTouchEvent都不拦截事件
- 点击红色区域

```
D/MainActivity: MainActivity dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onInterceptTouchEvent ACTION_DOWN
D/ViewContent: ViewContent dispatchTouchEvent ACTION_DOWN
D/ViewContent: ViewContent onTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity dispatchTouchEvent ACTION_UP
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_UP
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_UP
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_UP
D/ViewGroupB: ViewGroupB onTouchEvent ACTION_UP
D/MainActivity: MainActivity onTouchEvent ACTION_UP
```
可以看出，DOWN事件从上至下分发，最下层的ViewContent没有消费，向上传递时被ViewGroupB消费了，后续的UP事件会分发到ViewGroupB时直接交给它的onTouchEvent处理。

**因为DOWN事件被ViewGroupB消费了，后续事件也不用考虑拦截了（后续分发事件到ViewGroupB层时，ViewGroupB的onInterceptTouchEvent没有调用），dispatchTouchEvent后直接给ViewGroupB的onTouchEvent处理。**

示意图：

![](http://www.trinea.cn/wp-content/uploads/2016/01/touch2.jpg?dc9529&b1d01c)

## 三、某一层拦截事件（onInterceptTouchEvent返回true）

### 1. 接收到DOWN事件就拦截
- ViewGroupB的onInterceptTouchEvent在收到DOWN事件时返回true
- 都不消费事件，即onTouchEvent默认返回。
- 点击红色区域

```
D/MainActivity: MainActivity dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onInterceptTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity dispatchTouchEvent ACTION_UP
D/MainActivity: MainActivity onTouchEvent ACTION_UP
```
可以看到，ViewGroupB的onInterceptTouchEvent在收到DOWN事件时返回true，即拦截了DOWN事件，那么该事件直接交给它的onTouchEvent处理了，事件没有下发到下层，若不消费事件则继续向上找消费者。

### 2. DOWN事件不拦截，后续事件拦截
我们让ViewGroupB接受DOWN事件时不拦截，即下层会收到DOWN事件，让最底层View消费DOWN，在MOVE事件到来时ViewGroupB拦截，会发生什么呢？

- ViewGroupB接受DOWN事件时不拦截
- 红色区域的ViewContent收到DOWN的onTouchEvent返回true，即消费事件
- ViewGroupB接受MOVE事件时onInterceptTouchEvent返回true，即拦截MOVE事件
- 点击红色区域

```
D/MainActivity: MainActivity dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onInterceptTouchEvent ACTION_DOWN
D/ViewContent: ViewContent dispatchTouchEvent ACTION_DOWN
D/ViewContent: ViewContent onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity dispatchTouchEvent ACTION_MOVE
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_MOVE
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_MOVE
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_MOVE
D/ViewGroupB: ViewGroupB onInterceptTouchEvent ACTION_MOVE
D/ViewContent: ViewContent dispatchTouchEvent ACTION_CANCEL
D/ViewContent: ViewContent onTouchEvent ACTION_CANCEL
D/MainActivity: MainActivity onTouchEvent ACTION_MOVE
D/MainActivity: MainActivity dispatchTouchEvent ACTION_MOVE
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_MOVE
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_MOVE
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_MOVE
D/ViewGroupB: ViewGroupB onTouchEvent ACTION_MOVE
D/MainActivity: MainActivity onTouchEvent ACTION_MOVE
D/MainActivity: MainActivity dispatchTouchEvent ACTION_UP
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_UP
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_UP
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_UP
D/ViewGroupB: ViewGroupB onTouchEvent ACTION_UP
D/MainActivity: MainActivity onTouchEvent ACTION_UP
```
看到了吗，DOWN事件没什么问题，从各层分发直至找到它的消费者——ViewContent。但是，当后续的MOVE产生时，在分发过程中被ViewGroupB拦截，这时本来的消费者即下一层的ViewContent会收到CANCEL事件。

当MOVE再次产生时，直接分发到ViewGroupB层就停止了，会交给ViewGroupB的onTouchEvent处理MOVE，在不消费MOVE的情况下，MOVE事件都不会向上传递给ViewGroupA的onTouchEvent。

![](http://7xoze0.com1.z0.glb.clouddn.com/anroid-view-touch-cancel.png)

### 3. 子View请求父ViewGroup不拦截事件
在2的基础上，DOWN被最底层消费，而MOVE和后续事件被上层拦截了。
如果在这种情形下，下层还想收到后续事件怎么办呢？可以调用：

```java
getParent().requestDisallowInterceptTouchEvent(true);
```

这句代码的作用是下层View请求父ViewGroup不拦截事件，在2的例子中最底层View消费DOWN后，调用这句，就可以继续收到后续事件了。

```
D/MainActivity: MainActivity dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_DOWN
D/ViewGroupA: ViewGroupA onInterceptTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_DOWN
D/ViewGroupB: ViewGroupB onInterceptTouchEvent ACTION_DOWN
D/ViewContent: ViewContent dispatchTouchEvent ACTION_DOWN
D/ViewContent: ViewContent onTouchEvent ACTION_DOWN
D/MainActivity: MainActivity dispatchTouchEvent ACTION_MOVE
D/ViewGroupA: ViewGroupA dispatchTouchEvent ACTION_MOVE
D/ViewGroupB: ViewGroupB dispatchTouchEvent ACTION_MOVE
D/ViewContent: ViewContent dispatchTouchEvent ACTION_MOVE
D/ViewContent: ViewContent onTouchEvent ACTION_MOVE
D/MainActivity: MainActivity onTouchEvent ACTION_MOVE
```

>注意1：在调用requestDisallowInterceptTouchEvent方法后，后续事件被分发时，各层都不走onInterceptTouchEvent方法。

>注意2:  当某一层的View在调用了setOnClickListener或setOnTouchListener后，情况和注意1一样，后续事件被分发时，各层都不走onInterceptTouchEvent方法。

>注意3: 某一层在onTouchEvent消费DOWN事件，显示返回true后，后续事件会尝试继续找它，对于无显示返回true的后续事件，最终还会传递到MainActivity的onTouchEvent。

## 参考资料
关于Android Touch事件传递的文章和讲解网上很多，其实有下面两个＋写Demo实验就够了。

- 一个Dave Smith的讲解视频：[Mastering the Android Touch System](http://v.youku.com/v_show/id_XODQ1MjI2MDQ0.html)
- 上面视频的课件：[Mastering the Android Touch System.pdf](http://trinea.github.io/download/pdf/android/PRE_andevcon_mastering-the-android-touch-system.pdf)