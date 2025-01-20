---
title: 了解 Fragment
date: 2023-03-13 11:32:06
category: 笔记
---
开发者文档中 Fragment 的内容还挺多的，加上现在其实也不急着学这个，所以这里就记录最基本的部分好了。

<!-- more -->

## 使用 Fragment

Fragment 表示应用界面中可重复使用的一部分，是一种可以嵌入在 Activity 当中的 UI 片段，具有自己的生命周期，并且可以处理自己的输入事件。**Fragment 不能独立存在**，而是必须由 Activity 或另一个 Fragment 托管。Fragment 的视图层次结构会成为宿主的视图层次结构的一部分，或附加到宿主的视图层次结构中。

一般情况下，Fragment 需要被内嵌到`FragmentActivity`类或者是它的子类当中。例如，可以在`MainActivity`的布局中嵌入 Fragment ，是因为`MainActivity`继承自`AppCompatActivity`，而`AppCompatActivity`又继承自`FragmentActivity`类。

添加 Fragment 一般有两种方式：

- 在布局文件中使用`<fragment>`标签，然后在标签内填入需要嵌入的 Fragment 的完整类名（包括包名）。
- 使用 Fragment 容器，例如`<FragmentContainerView>`：先在布局文件中使用`<FragmentContainerView>`标签，然后在 Activity 中通过代码的方式将 Fragment 添加进去。

`<FragmentContainerView>`被强烈推荐使用，因为其包含了对 Fragment 的优化，这些优化是其它布局例如`FrameLayout`所没有的。甚至还可以用`<FragmentContainerView>`代替`<fragment>`。

### 直接嵌入

首先来看个简单的例子：如何将两个 Fragment 嵌入一个 Activity 中，并且这两个 Fragment 各占据一半屏幕。

左半部分的 Fragment 的 XML 部分，使用线性布局，背景设置为了红色，包含了一个文本：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#FF4500"
    tools:context=".LeftFragment">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:text="Left Fragment" />

</LinearLayout>
```

右半部分的 Fragment 的 XML 部分，使用线性布局，背景设置为了蓝色，包含了一个按钮：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#4876FF"
    tools:context=".RightFragment">

    <Button
        android:id="@+id/rightFragBt"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="BUTTON" />

</LinearLayout>
```

Activity 的 XML 部分，需要注意的内容写在了注释中：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    tools:context=".MainActivity">

    <!--
    * 使用 fragment 标签引入 Fragment ，对于要引入的 Fragment ，需要用 name 属性准确指定类名，包括包名
    * layout_width="0dp" 需要和 layout_weight="1" 搭配使用
    * 使用 layout_weight="1" 属性的控件需要被包含在 LinearLayout 中
    * 必须要给 Fragment 添加 id ，否则运行报错
    * 可以使用 class 属性代替 android:name
    -->

    <fragment
        android:id="@+id/leftFrag"
        android:name="com.risingsun.androidlab.LeftFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1" />

    <fragment
        android:id="@+id/rightFrag"
        android:name="com.risingsun.androidlab.RightFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1" />

</LinearLayout>
```

接着需要重写两个 Fragment 类中的`onCreateView()`，通过`inflate()`将布局返回：

```kotlin
override fun onCreateView(
    inflater: LayoutInflater, container: ViewGroup?,
    savedInstanceState: Bundle?
): View? {
    // Inflate the layout for this fragment
    return inflater.inflate(R.layout.fragment_right, container, false)
}
```

剩下的什么都不用修改，运行程序就能看到效果，还不错。

如果想要以最简单的方式创建一个 Fragment ，只需要将该 Fragment 的布局文件作为参数传入继承的`Fragment`类的构造函数中：

```kotlin
class BlankFragment : Fragment(R.layout.fragment_blank)
```

这样甚至都不用去重写`onCreateView()`。

### 动态添加

在上面例子的基础上，在 Activity 的 XML 部分，将引入的左半部分的 Fragment 的代码删除，然后添加一个 FrameLayout 作为容器：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    tools:context=".MainActivity">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/leftLayout"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1" />

    <fragment
        android:id="@+id/rightFrag"
        android:name="com.risingsun.androidlab.RightFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1" />

</LinearLayout>
```

接着修改 Activity 中的代码：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        var temp = true  // 标记位
        findViewById<Button>(R.id.rightFragBt).setOnClickListener {
            if (temp) loadFragment(LeftFragment())
            else loadFragment(RightFragment())
            temp = !temp
        }
    }

    private fun loadFragment(fragment: Fragment) {
        val fragmentManager = supportFragmentManager
        fragmentManager.beginTransaction().apply {
            replace(R.id.leftLayout, fragment)
            commit()
        }
    }
}
```

上面的代码中，首先为右半部分的 Fragment 中的按钮添加了点击事件，具体操作是根据标记位来加载不一样的 Fragment 。然后定义了一个`loadFragment()`函数，用来执行加载 Fragment 的操作，具体逻辑为：首先通过`getSupportFragmentManager()`获取一个`FragmentManager`类对象，然后通过这个类的`beginTransaction()`获取一个`FragmentTransaction`类对象，然后再调用这个类的`replace()`来执行替换操作，在上述例子中，就是将 id 为`leftLayout`的容器中的内容替换成了方法接收到的 Fragment 对象，最后执行`commit()`提交更改。

程序执行，具体效果就是第一次点击按钮，左半部分加载了之前定义的红色的 Fragment ，再按一次按钮就变成和右边相同的蓝色的 Fragment ，再按一次按钮又会变回红色的 Fragment ，如此反复。

#### 添加到返回栈

`replace()`每次调用，容器的内容就会被新的 Fragment 替换，当执行返回操作以后，app 就会直接退出。如果想让 Fragment 都被添加到一个栈中去管理的话，就可以在`commit()`前执行`addToBackStack(null)`，该方法接收一个字符串参数，用于表明这个返回栈的状态，此处可以传入`null`。

添加了`addToBackStack(null)`之后，程序再次运行，点击右半边 Fragment 中的按钮 3 次，也就是加载了 3 次 Fragment 之后，就需要执行 3 次返回操作才能将这些 Fragment 出栈，执行第 4 次返回操作的时候，就可以退出 app 了。

实际上，官方更加推荐使用 Jetpack 中的 Navigation 库来管理应用的导航，这样不仅更加规范和统一，而且也不再需要和`FragmentManager`直接交互。

### 根据屏幕加载布局

借助限定符（qualifier）可以让程序在运行时根据设备的分辨率或屏幕大小来决定加载哪个布局。

首先在 Activity 的布局文件中添加一个 Fragment ，然后在资源文件夹中添加一个名为`layout-large`的文件夹，在该文件夹内同样添加一个名为`activity_main.xml`的布局文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    tools:context=".MainActivity">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/leftFrag"
        android:name="com.risingsun.androidlab.BlankFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1" />

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/rightFrag"
        android:name="com.risingsun.androidlab.BlankFragment"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="3" />

</LinearLayout>
```

此时如果时以手机的形式运行应用，会得到只有一个 Fragment 的界面，而如果以平板运行，则会得到左右共两个 Fragment 的界面。

另外，有时候为了可以更加灵活地为不同设备加载布局（不管它们是不是被系统认定为 large），这时就可以使用最小宽度限定符（smallest-width qualifier）。最小宽度限定符允许对屏幕的宽度指定一个最小值（以 dp 为单位），然后以这个最小值为临界点，屏幕宽度大于这个值的设备就加载这个布局，屏幕宽度小于这个值的设备就加载另一个布局。

例如在资源目录下新建`layout-sw600dp`文件夹，然后在这个文件夹下新建`activity_main.xml`布局，这就意味着，当程序运行在屏幕宽度大于等于 600dp 的设备上时，会加载 layout-sw600dp/activity_main 布局，当程序运行在屏幕宽度小于 600dp 的设备上时，则仍然加载默认的 layout/activity_main 布局。

### DialogFragment

用于展示一个浮动的对话框，fragments 将自动处理`Dialog`类的创建和清理工作。

- [DialogFragment | Android Developers](https://developer.android.com/reference/androidx/fragment/app/DialogFragment)
- [Displaying dialogs with DialogFragment | Android Developers](https://developer.android.com/guide/fragments/dialogs)

### PreferenceFragmentCompat

将`Preference`类对象的层次结构以 List 的形式展示出来，还可以利用`PreferenceFragmentCompat`构建一个 app 的设置界面。

- [Preference | Android Developers](https://developer.android.com/reference/androidx/preference/Preference)
- [Settings | Android Developers](https://developer.android.com/develop/ui/views/components/settings)

## 与 Activity 交互

Fragment 在界面上虽然是嵌入在 Activity 中的，但是在逻辑上，Fragment 和 Activity 属于两个不同的类。`FragmentManager`提供了一个叫`findFragmentById`的函数，用来在 Activity 中去引用那些在布局上位于 Activity 之内的 Fragment 的实例。例如现在在 Activity 的布局中通过 fragment 标签引入了`LeftFragment`，并且这个标签的 id 叫`leftFrag`，并且假设`LeftFragment`类中存在一个函数叫`foo()`，那么就可以：

```kotlin
val fragment = supportFragmentManager.findFragmentById(R.id.leftFrag) as LeftFragment
fragment.foo()
```

在 Fragment 中调用 Activity 也比较简单，直接调用`getActivity()`就可以拿到和 Fragment 相关联的 Activity 实例，只不过这个实例有可能为`null`。当需要上下文的时候也可以调用`getActivity()`，因为 Activity 本身就是一个`Context`对象。

另外，在通过`getActivity()`拿到 Activity 的实例以后，还可以通过这个 Activity 的实例去调用另一个 Fragment ，如此就做到了 Fragment 之间的交互。

## 生命周期

Fragment 的生命周期大致可分为 4 个阶段：

- 运行状态：当相关联的 Activity 进入该状态时，Fragment 也会进入该状态。
- 暂停状态：当相关联的 Activity 因为另一个未占满屏幕的 Activity 被添加到栈顶而进入暂停状态时，Fragment 也会进入该状态。
- 停止状态：当相关联的 Activity 进入该状态时，Fragment 也会进入该状态。或者当 Fragment 完全不可见并且处于返回栈中时，Fragment 也是停止状态的。
- 销毁状态：当相关联的 Activity 进入该状态时，Fragment 也会进入该状态。或者当调用`FragmentTransaction`类的`remove()`或者`replace()`方法将 Fragment 从 Activity 中移除时，Fragment 也会进入该状态。

Fragment 拥有除了`onRestart()`以外 Activity 所拥有的 6 个生命周期回调函数，并且还额外拥有 Activity 没有的 5 个回调函数，分别是：

- `onAttach()`：当 Fragment 和 Activity 建立关联时调用。
- `onCreateView()`：Fragment 创建视图（加载布局）时调用。
- `onActivityCreated()`：与 Fragment 相关联的 Activity 创建完毕时调用。
- `onDestroyView()`：当与 Fragment 关联的视图被移除时调用。
- `onDetach()`：当 Fragment 和 Activity 解除关联时调用。
