---
title: Compose 学习笔记 ① - 基础概念
date: 2023-11-28 19:12:27
category: 笔记
---
不知不觉 Compose Multiplatform 的版本已经来到了 1.5.10 了，最近打算在业余时间把 Compose 的知识点重新过一遍，至于跨平台的话等后续再找时间玩一下。

<!-- more -->

## 声明式 UI

Compose 的写法属于「声明式」，声明式编程是一种编程范式，而编程范式是一种编程的风格或方法论，除了「声明式」以外，还有「命令式」、「函数式」等等。

对于同一个界面需求，比如创建一个文本控件，然后修改其显示的内容，传统 View 系统的做法是先在布局文件中创建一个文本控件，然后再在代码中去获取到这个控件的实例，最后通过代码手动去修改它的属性：

```xml
<TextView
android:id="@+id/myTextView"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:text="传统 XML 写法" />
```

其中`android:text`中的内容是固定死的，如果我后续要在代码中修改文本内容的话，我要这样：

```kotlin
val textView = findViewById<TextView>(R.id.myTextView)
textView.text = "设置新的文本"
```

像传统 View 系统这样的写法就是命令式编程，命令式编程通常包含了程序状态的明确改变，以及通过语句按顺序执行的控制流。具体来说就是我们会给到计算机一系列指令，告诉它该如何执行任务。

那么到这里其实就大概可以猜到，「声明式 UI」的特点之一，或者说是 Compose 的核心写法就是不需要手动去更新界面，这就是声明式与命令式的区别体现。接下来是一个用 Compose 实现同样需求的例子：

```kotlin
val greeting = "Hello Compose!"
Text(text = greeting)  // 将上面创建的 text 变量作为参数传递进 Text() 中
```

这里的`Text()`看起来就像是一个函数一样，用官方的话来说，它是一个 Composable（中文译为「可组合项」），`Text()`其实也就是 Compose 中的文本组件，它接收多个参数，其中有一个类型为`String`的`text`参数将作为显示的内容，在这里我将上面创建的变量`greeting`作为参数传递，所以文本最终会显示「Hello Compose!」。之后如果在程序中的某个地方需要修改这个文本显示的内容，那么只需要修改变量`greeting`的值即可！也就是说，在这个例子中，Compose 的声明式体现在两个地方：

1. 自动利用给到的`text`参数初始化文本显示内容；
2. 当给到的参数发生变化时，将自动更新界面；

再简单一点说，是不是声明式 UI 与使不使用 XML 没关系，主要还是要看需不需要手动更新界面。

## 布局迁移

View 系统在进行界面布局时会用到各种 Layout ，这些 Layout 并不能直接在 Compose 中使用，但是都会有一些它们的替代品来实现相同的效果：

- FrameLayout → Box()
- LinearLayout → Column() / Row()
- RelativeLayout → Box() + Modifier
- ConstraintLayout → 一部分功能由 Box() 实现
- RecyclerView → LazyColumn() / LazyRow()
- ScrollView → Modifier.verticalScroll() / Modifier.horizontalScroll()
- ViewPager → 位于 accompanist 库中的 Pager()

## Modifier

在 View 系统中，对控件的属性设置需要在 XML 文件中进行，但是在 Compose 中没有 XML 文件，取而代之的是 Modifier 和控件函数本身自带的参数。例如想要让一个文本控件显示的内容是「Try Modifier」，同时将其内边距设置为 5 个 dp 就这可以这么做：

```kotlin
Text(text = "Try Modifier", modifier = Modifier.padding(5.dp))
```

可以看到，`text`参数负责接收需要显示的文本，而内边距属性则是由一个 Modifier 实例带进了`modifier`参数中。但是这里就会产生一个问题：「我怎么知道设置文本的时候是用到`text`参数，然后设置内边距的时候是用到 Modifier 呢？」其实很好分辨，首先 Modifier 更多是用来对一些通用属性进行设置的，例如内边距、背景颜色、点击监听等等，这些属性基本上每个控件都会有。而部分控件特有的属性，则是会单独作为一个函数参数而存在的，例如上方的文本控件中的`text`参数，因为显示文本是文本相关的控件才有的。

此外 Modifier 还有一个特性尤其需要注意，即「顺序敏感，依次执行，调用叠加」。在使用 Modifier 时，可以通过链式调用一次性为控件设置多个属性，但是这些设置属性的函数会按照编写的顺序依次执行，同时对于某些函数来说，多次调用的效果是叠加而不是覆盖。最能体现这个特性的一个案例就是内边距和外边距，在 Modifier 中，通过`padding()`来设置内边距，但是并不能通过`margin()`来设置外边距，因为 Modifier 中并没有`margin()`。同样以上面的文本控件来做一个简单的示例：

```kotlin
Text(text = "Try Modifier",
    modifier = Modifier
        .background(Color.Black)
        .padding(5.dp)
        .background(Color.Red)
        .padding(5.dp)
)
```

程序运行，效果如下：

![Modifier 设置内、外边距效果](/images/note/Modifier-内、外边距测试.png)

从效果上来看，大致可以判断出最外层这一圈黑色的好像就是文本的外边距，里面的红色背景的边界距离文本也有一定距离，这个距离就是内边距。

## 分组和依赖引入

从官方文档中得知 Compose 由`androidx`中的 7 个 Maven 组 ID 构成，每个组都包含一套特定用途的功能，这 7 个组以及其说明如下：

| 组                   | 说明                                                                                   |
|---------------------|----------------------------------------------------------------------------------------|
| compose.animation   | 在 Jetpack Compose 应用中构建动画，丰富用户体验。                                           |
| compose.compiler    | 借助 Kotlin 编译器插件，转换 @Composable functions（可组合函数）并启用优化功能。               |
| compose.foundation  | 使用现成可用的构建块编写 Jetpack Compose 应用，还可扩展 Foundation 以构建您自己的设计系统元素。   |
| compose.material    | 使用现成可用的 Material Design 组件构建 Jetpack Compose UI。这是更高层级的 Compose 入口点，旨在提供与 www.material.io 上描述的组件一致的组件。|
| compose.material3   | 使用 Material Design 3（新一代 Material Design）组件构建 Jetpack Compose UI。Material 3 中包括了更新后的主题和组件，以及动态配色等 Material You 个性化功能，旨在与新的 Android 12 视觉风格和系统界面相得益彰。|
| compose.runtime     | Compose 编程模型和状态管理的基本构建块，以及 Compose 编译器插件的目标核心运行时。                 |
| compose.ui          | 与设备互动所需的 Compose UI 的基本组件，包括布局、绘图和输入。                                  |

View 系统并没有像 Compose 这样明确分组，一定程度上导致 View 系统发展到后期越来越难扩展。这样看来 Compose 的分组带来的好处是显而易见的：首先是开发者可以根据自己的需求选择性地添加依赖项，而不是一次性引入所有的库；再者就是官方自己也方便版本管理和更新。

那么在开发过程中应该如何选择性地引入依赖项呢，这就要先对这些组进行一个补充介绍：

1. `compiler`：编译器插件，在 gradle 的`dependencies`中并不需要明确引入，因为在`dependencies`的上方就已经标注了：`kotlinCompilerExtensionVersion = "1.4.3"`。
2. `runtime`：Compose 底层的模型，包含数据结构和状态转换机制等等，例如`mutableStateOf()`、`remember()`等等。
3. `ui`：提供和 UI 相关的最基础的功能，包括绘制、测量、布局、触摸反馈等。
4. `animation`：动画相关。
5. `foundation`：提供相对完整的 UI 体系，例如`Column()`和`Row()`、`Image()`等。
6. `material`/`material3`：包含了大量 Material Design 风格的组件，例如`Button()`、`OutlinedButton()`、`TextButton()`等。值得一提的是：「按钮」这个东西并不是 Material Design 发明的，只是官方设计的具备了 Material Design 风格的按钮刚好叫作「Button」而已。

在上面的列表中，越靠前的组越接近底层，往后的组会依赖前面的组，例如 4 依赖 3 ，3 依赖 2 ，也就是说，在使用过程中，只需引入其中一个组，这个组之前的组都会被引入。例如，我想使用官方提供的具有 Material Design 风格的控件，所以我引入了`material`/`material3`，又因为`material`/`material3`在最上层，所以它依赖的组（`foundation`、`animation`等）也都会被引入。
