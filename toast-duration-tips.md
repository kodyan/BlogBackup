title: Toast.LENGTH_SHORT和Toast.LENGTH_LONG
date: 2015/3/16 19:07:26 
categories: 
tags: Android
---
上一篇博客中介绍了实现双击Back键退出应用的两种实现方式，设定的时间间隔是2000毫秒，我说因为Toast.LENGTH_SHORT就是代表2000毫秒，所以识别为两次连续点击Back的时间间隔就应该设定为2000毫秒。下面介绍我是如何通过查看源码知道Toast.LENGTH_SHORT和Toast.LENGTH_LONG分别代表多少时间的。

<!--more-->
当在代码中点击`Toast.LENGTH_SHORT`的`LENGTH_SHORT`进入`Toast.java`的源码时可以看到：
```java
    /**
     * Show the view or text notification for a short period of time.  This time
     * could be user-definable.  This is the default.
     * @see #setDuration
     */
    public static final int LENGTH_SHORT = 0;

     /**
     * Show the view or text notification for a long period of time.  This time
     * could be user-definable.
     * @see #setDuration
     */
    public static final int LENGTH_LONG = 1;
```
`Toast.LENGTH_SHORT`和`Toast.LENGTH_LONG`分别是0和1，这显然不是实际的显示时间。
再看`Toast`的`makeText()`方法：
```java
    /**
     * Make a standard toast that just contains a text view.
     *
     * @param context  The context to use.  Usually your {@link android.app.Application}
     *                 or {@link android.app.Activity} object.
     * @param text     The text to show.  Can be formatted text.
     * @param duration How long to display the message.  Either {@link #LENGTH_SHORT} or
     *                 {@link #LENGTH_LONG}
     *
     */
    public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
        Toast result = new Toast(context);

        LayoutInflater inflate = (LayoutInflater)
                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
        tv.setText(text);
        
        result.mNextView = v;
        result.mDuration = duration;

        return result;
    }
```
在`makeText()`方法中将传给`makeText()`的duration赋值给了变量mDuration。
再去`show()`方法中查看：
```java
    /**
     * Show the view for the specified duration.
     */
    public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
```
`mDuration`传给了`INotificationManager`实例`service`的`enqueueToast`方法。再追踪`enqueueToast`就追踪不到了。显然这个`INotificationManager`是关键，去Android源码中搜搜看吧。
[androidxref.com](http://androidxref.com/)是个查看Android源码的好地方，选择KitKat - 4.4.4_r1，搜索`INotificationManager`，结果中有`INotificationManager.aidl`，是Android接口定义语言，是为了提供一些机制在不同进程之间进行数据通信。点进去看看，果然有一句：
```java
void enqueueToast(String pkg, ITransientNotification callback, int duration);
```
继续返回搜索`enqueueToast`，结果除了`INotificationManager`、`Toast`还有一个`NotificationManagerService`类，点进去查看源码，搜索到实现`enqueueToast()`方法的位置，继续定位`showNextToastLocked()`方法->`scheduleTimeoutLocked(ToastRecord r)`方法，好了，看到了一句：
```java
long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
```
可以看出，duration等于Toast.LENGTH_LONG时delay取LONG_DELAY，否则取SHORT_DELAY。再看看这两个静态变量的值是多少：
```java
private static final int LONG_DELAY = 3500; // 3.5 seconds
private static final int SHORT_DELAY = 2000; // 2 seconds
```
好了，终于找到了。结论就是：
> Toast.LENGTH_LONG代表的是3500毫秒，其他设定的毫秒值包括Toast.LENGTH_SHORT都是2000毫秒。