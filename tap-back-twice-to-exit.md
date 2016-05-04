title: 双击Back键退出应用的实现方法
date: 2015/3/16 14:58:47 
categories: 
tags: Android
---
在我们使用app时会注意到，很多应用的退出方式是双击返回键。第一次点击返回键会弹出Toast提示“再按一次返回键退出”，在规定的时间间隔内再次点击返回键就会退出应用。那么这个功能是如何实现的呢？下面给出两种实现方式以及一些有意思的东西。
> ## 本文内容 ##
> 
- 实现方式一
- 实现方式二
- 关于Toast的一些思考

<!--more-->
## 实现方式一 ##
设置一个`flag`，用来标识在设定的时间间隔内是否已经点击过一次Back键，默认为`false`。重写Activity的onBackPressed()方法，捕获Back键的点击操作。如果已经点击过一次Back键，则执行退出操作；否则将`flag`设置为`true`，弹出Toast提示“再按一次返回键退出”，并在规定时间间隔之后将`flag`置为`false`。这样就实现了在规定的时间间隔内，如果用户点击了两次Back键则执行退出应用操作。下面是参考代码：
```java
boolean hasClickBackOnece = false;
......
public void onBackPressed() {
          if (hasClickBackOnece) {
            super.onBackPressed();
            finish();
            return;
        }
        this.hasClickBackOnece = true;
        ToastUtils.showToast(this, “再按一次返回键退出”, Toast.LENGTH_SHORT);
        new Handler().postDelayed(new Runnable() {
            @Override
            public void run() {
                hasClickBackOnece = false;
            }
        }, 2000);
    }
```

## 实现方式二 ##
设置好识别为连续两次点击的时间间隔，并记录点击Back键时的毫秒时间。若`该次点击Back时的毫秒时间 - 已设定的点击毫秒时间 > 时间间隔`，则说明两次点击的时间差没有超过设定的时间间隔，此时执行退出操作；否则弹出Toast提示“再按一次返回键退出”，并将该次点击时的毫秒时间赋给记录值。下面是参考代码：
```java
private static final int TIME_INTERVAL = 2000;
private long mBackPressed;

@Override
public void onBackPressed()
{
    if (System.currentTimeMillis() - mBackPressed <= TIME_INTERVAL) 
    { 
	 //add your action here to exit the app
        super.onBackPressed(); 
        return;
    }
    else { Toast.makeText(getBaseContext(), "Tap back button in order to exit", Toast.LENGTH_SHORT).show(); }

    mBackPressed = System.currentTimeMillis();
}
```

## 关于Toast的一些思考 ##
以上两种方法的代码中设置的时间间隔都是2000毫秒，到底应该设置两次点击的时间间隔是多少才会执行退出操作呢？我是这么考虑的：
> Toast提示“再按一次返回键退出”的duration是多少，则两次点击Back的时间间隔就设定为多少。

识别为两次连续点击的时间间隔与Toast显示“再按一次返回键退出”的持续时间保持一致，这样设置明显是合乎情理的。
上面代码中Toast的持续时间是Toast.LENGTH_SHORT，postDelayed中Runnnable时延设置的是2000毫秒。因为Toast.LENGTH_SHORT代表的时间就是2000毫秒。
点进源码查看Toast.LENGTH_SHORT明明是0啊，怎么是2000毫秒？下一篇博客我会介绍Toast.LENGTH_SHORT和Toast.LENGTH_LONG分别代表多长时间。

## 参考资料 ##
[Android: clicking TWICE the back button to exit activity](http://stackoverflow.com/questions/8430805/android-clicking-twice-the-back-button-to-exit-activity)
