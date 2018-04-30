---
layout: post
title: android.intent.category.DEFAULT的用途和使用
categories: [Android]
description: Android category
keywords: category
---



## android.intent.category.DEFAULT

---
### 两个概念

- explicit(明确，显式) intent

    ``` java
    Intent intent= new Intent(MainActivity.this, Secondary.class)； 
    ```

- implicit(隐藏,隐式) intent

    Implicit Intent没有明确的指定要启动哪个Activity ，而是通过设置一些Intent Filter来让系统去筛选合适的Acitivity去启动。


    在官方文档中有这么一段话：

> Android treats all implicit intents passed to startActivity() as if they contained at least one category: "android.intent.category.DEFAULT" (the CATEGORY_DEFAULT constant). Therefore, activities that are willing to receive implicit intents must include "android.intent.category.DEFAULT" in their intent filters

意思是说，每一个通过 startActivity() 方法发出的隐式 Intent 都至少有一个 category，就是 "android.intent.category.DEFAULT"，所以只要是想接收一个隐式 Intent 的 Activity 都应该包括"android.intent.category.DEFAULT" category，不然将导致 Intent 匹配失败。

即：如果要启动的Activity的<intent-filter/>在AndroidManifest.xml文件中没有android.intent.category.DEFAULT这个category的话，就会匹配失败。
    
从上面的论述还可以获得以下信息：
- 一个 Intent 可以有多个 category，但至少会有一个，也是默认的一个 category。
- 只有 Intent 的所有 category 都匹配上，Activity 才会接收这个 Intent。


---



Android 源代码 【正在寻找自动添加的地方...】：

```
寻找ing...
```




引用：https://blog.csdn.net/jason0539/article/details/10049899