---
layout: post
title: Android中，在子线程使用Toast会报错？
categories: [Android]
description: Android Toast
keywords: Toast
---



## Android中，在子线程使用Toast会报错？

---
在知乎看到一个问题：
https://www.zhihu.com/question/51099935/answer/387828477
>在子线程中使用Toast抛出异常，提示错误显示:Can't create handler inside thread that has not called Looper.prepare()

@森羴 的回答：https://www.zhihu.com/question/51099935/answer/125487934

Toast,Handler,分析为什么抛异常。ActivityThread和ViewRootImpl分析到底什么叫子线程不能更新UI。

Toast本质上是一个window，跟activity是平级的，checkThread只是Activity维护的View树的行为。Toast使用的无所谓是不是主线程Handler，吐司操作的是window，不属于checkThread抛主线程不能更新UI异常的管理范畴。它用Handler只是为了用队列和时间控制排队显示吐司。即使是子线程，先Looper.prepare,再show吐司，再Looper.loop一样可以吐出来，只不过loop操作会阻塞这个线程，没人这么玩罢了，都是让Toast用主线程的Handler，这个是在ActivityThread里初始化的，本来就是阻塞处理所有的UI交互逻辑。贴个代码吧还是～～  
``` Java
new Thread(){
        public void run(){
          Looper.prepare();//给当前线程初始化Looper
          Toast.makeText(getApplicationContext(),"你猜我能不能弹出来～～",0).show();
          //Toast初始化的时候会new Handler();无参构造默认获取当前线程的Looper，如果没有prepare过，则抛出题主描述的异常。上一句代码初始化过了，就不会出错。
          Looper.loop();
          //这句执行，Toast排队show所依赖的Handler发出的消息就有人处理了，Toast就可以吐出来了。但是，这个Thread也阻塞这里了，因为loop()是个for (;;) ...
        }
  }.start();
```

我的补充,和 @森羴回答结合起来看效果更好😄：

``` Java
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
    return makeText(context, null, text, duration);//①注意，这里looper为空
}
```

``` Java
public static Toast makeText(@NonNull Context context, @Nullable Looper looper,
            @NonNull CharSequence text, @Duration int duration) {
        Toast result = new Toast(context, looper);//② looper为null

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

``` Java
public Toast(@NonNull Context context, @Nullable Looper looper) {
        mContext = context;
        mTN = new TN(context.getPackageName(), looper);//③创建TN
        mTN.mY = context.getResources().getDimensionPixelSize(
                com.android.internal.R.dimen.toast_y_offset);
        mTN.mGravity = context.getResources().getInteger(
                com.android.internal.R.integer.config_toastDefaultGravity);
    }
```

其中：

``` Java
 mTN = new TN(context.getPackageName(), looper);
```
``` Java
TN(String packageName, @Nullable Looper looper) {
    ......
    if (looper == null) {
        // Use Looper.myLooper() if looper is not specified.
        //④从当前线程获取looper，一般为主线程，故在主线程中时，让Toast用主线程的Looper创建一个Handler
        looper = Looper.myLooper(); 
        if (looper == null) {
            throw new RuntimeException(
                    "Can't toast on a thread that has not called Looper.prepare()");
        }
    }
    mHandler = new Handler(looper, null) {
    }
    ......
}
```


看源码可以很清楚地分析出问题原因，多看源码对上层APP开发有很大益处 ; )