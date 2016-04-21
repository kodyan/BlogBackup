title: Toast及时更新显示文字及控制提示消失
date: 2015/3/16 20:28:38 
categories: 
tags: Android
---


- 在测试Toast时频繁让Toast显示文字会有延迟，因为Toast的消息队列中要按指定的duration显示完当前内容后才会显示下一条消息。如何才能让Toast及时更新显示内容呢？
- 有些应用双击Back退出时，会有一个“再次点击返回退出”的提示，当连续两次快速点击Back退出应用时，“再次点击返回退出”提示依然还在，指导显示完指定的duration。你都退出了，而你产生的Toast还在，这样合理吗？

> ## 本文内容 ##
> 
- 及时更新Toast显示的内容
- 控制Toast消失

<!--more-->
我自己写了一个Toast的工具类，实现了及时更新Toast显示的内容，以及控制Toast消失。下面是参考代码：
```java
package com.bupt.booktrade.utils;

import android.content.Context;
import android.widget.Toast;

public class ToastUtils{
	private static Toast mToast;
	public static void showToast(Context context, int msg, int duration) {
		if (mToast == null) {
			mToast = Toast.makeText(context, msg, duration);
		} else {
			mToast.setText(msg);
		}
		mToast.show();
	}
    public static void showToast(Context context, String msg, int duration) {
        if (mToast == null) {
            mToast = Toast.makeText(context, msg, duration);
        } else {
            mToast.setText(msg);
        }
        mToast.show();
    }
	public static void clearToast(){
		mToast.cancel();
	}
}
```
通过`ToastUtils`直接调用这三个静态方法即可。