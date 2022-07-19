---
title: Android 笔记
date: 2022-07-18 22:56:44
categories:
- Android 笔记
tags:
- Android 笔记
---

# 清单文件

# Activities

## Fragments
### 创建一个 fragment
一个 fragment 表示某个 activity 中用户接口的一个模块化部分。一个 fragment 有它自己的生命周期，接受它自己的输入事件，并且你可以在activity 运行时添加或移除 fragment。

#### 设置环境
fragments 需要 AndroidX fragment 库的依赖。为此，你需要添加 Google Maven 仓库到你的项目 build.grade 文件。
```css
buildscript {
    ...
    repositories {
        google()
        ...
    }
}
allprojects {
    repositories {
        google()
        ...
    }
}
```

为了将 AndroidX Fragment 库包含到你的项目，需要在你的 App 的 build.gradle 文件添加如下依赖：

```css
dependencies {
    def fragment_version = "1.3.5"

    // Java language implementation
    implementation "androidx.fragment:fragment:$fragment_version"
    // Kotlin
    implementation "androidx.fragment:fragment-ktx:$fragment_version"
}
```

#### 创建一个 fragment 类
为了创建一个 fragment，需要继承 AndroidX 的 `Fragment` 类，并覆盖它的方法，类似你创建一个 `Activity` 类。为了创建一个定义了自己的布局的最小 fragment，需要为基本构造器提供 fragment 的布局资源。如下所示：
```java
class ExampleFragment extends Fragment {
    public ExampleFragment() {
        super(R.layout.example_fragment);
    }
}
```

Fragment 库还提供了更多专业的 fragment 基类：

- DialogFragment
显示悬浮对话框，使用此类创建一个对话框是一个对于使用 Activity  dialog helper 方法的好的替代方案，因为 fragment 会自动处理对话框的创建和清理。详细信息参考 DialogFragment。

- PreferenceFragmentCompat
显示作为列表的 Preference 对象的层次结构。你可以使用 
PreferenceFragmentCompat 来为你的 App 创建一个设置屏幕。

#### 添加一个 fragment 到 activity
通常，你的 fragment 必须嵌入在 AndroidX `FragmentActivity` 中，用以贡献一部分 UI 到 activity 布局。`FragmentActivity` 是 `AppCompatActivity` 基类，因此如果你已经在你的 App 中为 AppCompatActivity 提供了向后兼容性，那么你不必改变你的 activity 基类。

你可以通过两种方式添加 fragment：
- 在 activity 的布局文件中定义片段
- 在 activity 布局文件定义 fragment 容器，后面通过程序添加到 activity。

在任何一种情况下，你都需要添加一个 `FragmentContainerView`，定义了 fragment 应该放在 activity 试图层次结构中的位置。

强烈建议：使用 Fragment 作为 fragment 的容器，因为 `FragmentContainerView` 包含了其他 View Group（如 FrameLayout）没有提供的修复程序。

##### 通过 XML 添加 fragment
为了声明将 fragment 添加到你的 activity 布局 XML，你需要添加一个 `FragmentContainerView` 元素：
```xml
<!-- res/layout/example_activity.xml -->
<androidx.fragment.app.FragmentContainerView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:name="com.example.ExampleFragment" />
```
Android:name 指定要实例化的 fragment 类名。当 activity 布局填充时，指定的 fragment 会实例化，当新实例化 fragment 时，onInflate() 会被调用，并且会创建一个 `FragmentTransaction` 去将 fragment 添加到 `FragmentManager`。

##### 程序化添加 fragment
为了程序化添加 fragment 到 activity，布局应该引入 `FragmentContainerView` 作为 fragment 容器，如下所示：

```xml
<!-- res/layout/example_activity.xml -->
<androidx.fragment.app.FragmentContainerView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragment_container_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```
与 XML 方式不同，android:name 属性并未使用，因此不会自动实例化特定 fragment。相反，使用 FragmentTransaction 来实例化 fragment 并将其添加到 activity 的布局中。

当你的 activity 在运行时，你可以制造 fragment transaction 例如添加、删除或者替换 fragment。在 `FragmentActivity` 中，你可以获得 `FragmentManager` 的实例，你可以通过它创建 `FragmentTransaction`。 在 activity 的 onCreate() 方法中，你可以使用 FragmentTransaction.add() 实例化你的 fragment，传递参数 ViewGroup ID 和 fragment Class，然后提交事务，如下图所示：

```java
public class ExampleActivity extends AppCompatActivity {
    public ExampleActivity() {
        super(R.layout.example_activity);
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (savedInstanceState == null) {
            getSupportFragmentManager().beginTransaction()
                .setReorderingAllowed(true)
                .add(R.id.fragment_container_view, ExampleFragment.class, null)
                .commit();
        }
    }
}
```

> 注意：当执行一个 FragmentTransaction 时，你应该总是使用 setReorderingAllowed(true)。更多信息参考官网。


在前面的例子中，请注意，只有在 savedInstanceState 为 null 时，才会创建 fragment 事务。这是为了确保仅仅当 activity 第一次创建的时候，fragment 才会添加一次。发生配置更改或者 activity recreate，savedInstanceState 不再为 null，并且不需要再添加一次 fragment，因为 fragment 可以自动从 savedInstanceState 恢复。

如果你的 fragment 需要一些初始化数据，你可以通过在调用 FragmentTransaction.add() 时提供一个 Bundle，如下图所示：

```java
public class ExampleActivity extends AppCompatActivity {
    public ExampleActivity() {
        super(R.layout.example_activity);
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (savedInstanceState == null) {
            Bundle bundle = new Bundle();
            bundle.putInt("some_int", 0);

            getSupportFragmentManager().beginTransaction()
                .setReorderingAllowed(true)
                .add(R.id.fragment_container_view, ExampleFragment.class, bundle)
                .commit();
        }
    }
}
```
在 fragment 中，你可以通过调用 `requireArguments()` 来获取参数 Bundle，并且可以使用适合的 getter 方法来获取每个参数。
```java
class ExampleFragment extends Fragment {
    public ExampleFragment() {
        super(R.layout.example_fragment);
    }

    @Override
    public void onViewCreated(@NonNull View view, Bundle savedInstanceState) {
        int someInt = requireArguments().getInt("some_int");
        ...
    }
}
```


# OkHttp
```css
implementation 'com.squareup.okhttp3:okhttp:4.9.0'
```

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```