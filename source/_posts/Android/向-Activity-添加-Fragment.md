---
title: 向 Activity 添加 Fragment
date: 2022-07-18 22:32:36
categories:
- Android
tags:
- Android
---

# 向 Activity 添加 Fragment

## 方式一

在 Activity 的布局文件内声明片段

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <fragment android:name="com.example.news.ArticleListFragment"
            android:id="@+id/list"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
    <fragment android:name="com.example.news.ArticleReaderFragment"
            android:id="@+id/viewer"
            android:layout_weight="2"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
</LinearLayout>
```


`<fragment>` 的 android:name 属性指定要在布局中进行实例化的 Fragment 类。

创建 activity 布局时，系统会将每个 fragment 实例化，并调用 OnCreateView() 方法，以检索每个片段的布局。系统会返回插入 fragment 后的 View。

> 注意：每个片段都需要唯一标识符，重启 Activity 时，系统可使用该标识符来恢复片段（您也可以使用该标识符来捕获片段，从而执行某些事务，如将其移除）。可以通过两种方式为片段提供 ID：

为 android:id 属性提供唯一 ID。
为 android:tag 属性提供唯一字符串。


## 方式二

在 activity 运行期间，可以随时将片段添加到 activity 布局中，只需指定 fragment 放入哪个 ViewGroup。

在 activity 中执行片段事务（如添加、移除或替换片段），必须使用 FragmentTransaction API。

可以从 FragmentActivity 获取一个FragmentTransaction实例：

```java
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
```

然后，使用 add() 方法添加片段，指定要添加的片段以及插入哪个视图：

```java
ExampleFragment fragment = new ExampleFragment();
fragmentTransaction.add(R.id.fragment_container, fragment);
fragmentTransaction.commit();
```

一旦通过 FragmentTransaction 做出了更改，就必须调用 commit() 以使更改生效。