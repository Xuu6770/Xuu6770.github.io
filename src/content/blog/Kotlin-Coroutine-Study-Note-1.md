---
title: Kotlin 协程学习笔记 1 - 基础知识
date: 2024-06-23 22:32:00
category: 笔记
---
协程是 Kotlin 的一个亮点，内容也比较多，故而拎出来单独做一篇笔记。本节只是对协程进行一个基础性的概述。

<!-- more -->

> 本篇内容主要还是以基于 JVM 的 Kotlin 和 Android 开发的角度为主。

## 协程的历史

Kotlin 的协程是 2017 年初在 1.1 版本加入进来的，那时协程还只是实验性的（Experimental）。等到 2018 年底，Kotlin 更新到 1.3 版本的时候，协程才成为 Kotlin 的正式特性。接着到 2019 年 Kotlin 协程才推出 Flow 相关的 API。

事实上早在 1967 年的 Simula 语言当中，就已经出现了协程。不过在之后的几十年里，协程并没有被推广开，后续的 C、C++、Java 之类的语言，更多的是使用线程来进行异步和并发。直到 2012 年左右，C# 重新拾起了协程这个特性，实现了 async、await、yield。之后，JavaScript、Python、Kotlin 等语言才继续跟进实现了对应的协程。

## 相关概念

首先需要明确，虽然 Kotlin 现在已经是跨平台的了，但是协程在不同平台的底层实现并不完全一样。

协程和线程的作用是一样的，都是用来管理并发，有的语言用线程来管理并发，有的则是用协程，有的则两个都用。例如 Java 就是用线程的，而 Kotlin 则是用协程。并且 Kotlin 的协程在使用体验上要比线程好，这就是为什么在 JVM 上明明都已经有 Java 线程了，Kotlin 也是可以直接用线程的，可是 Kotlin 还是要封装一下线程，发明出个协程出来。协程中最大的亮点就在于它可以用线性的结构编写异步的代码，例如需要在子线程中进行网络请求，然后在主线程中显示结果时，用传统的线程的做法也好，用第三方库例如 RxJava 也好，难免要进行回调嵌套或者是链式调用，但是用协程就可以做到所有的代码都在同一个层级上，结构清晰明了。

虽然协程和线程的作用大致相同，但是它们的概念模型却不太一样，以下是对协程的几句简单介绍：

- 可以简单理解为「相互协作的程序」，例如针对某个协程，可以通过`yield()`将其挂起，让出执行权。
- 可以理解为更加轻量的线程，成千上万个协程可以同时运行在一个线程当中，但是成千上万个线程就可能会出现内存溢出。
- 可以理解为是运行在线程当中的轻量的 Task ，这些 Task 支持挂起和恢复，当 Task 挂起时，就不会阻挡到后续的 Task 。
- 不会与特定的线程绑定，可以在不同的线程之间灵活切换。
- 协程是非阻塞式的，比如`delay()`，而线程是阻塞式的，比如`Thread.sleep()`。

协程说白了就是用线程来实现的并发管理库，它要做的无非这么 3 件事：

1. 线程间的切换；
    1. 例如：切换到 并行/子 线程；
    2. 或者：在 Android 开发中，我们需要切换到 UI 线程，也就是主线程去更新界面（一般使用 Handler 或者 View 来操作）；
2. 线程间的配合，包括在各个线程的执行过程中去等待别的线程；
3. 线程安全，也就是锁，也就是对共享资源访问的互斥，以达到保护资源的目的，确保资源正确性的作用；

## 引入协程库

在开始之前，还需要在项目中对协程进行进入，毕竟它是个库。通过[官方 GitHub 页面](https://github.com/Kotlin/kotlinx.coroutines)的指示可以得知，如果项目是使用 Gradle 进行构建，那么就需要在其配置文件中引入：

```groovy
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC")
    // ↓ 如果是 Android 开发，则额外引入以下这条依赖
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0-RC")
}
```

## launch()

首先还是先来看一下在基于 JVM 的 Kotlin 中如何创建线程：

```kotlin
Thread {
    // 这是第一种方式
}.start()

// ↓ 这是 Kotlin 封装的函数，本质上和第一种方式的实现是一样的
thread {
    // 这是第二种方式
}
```

还算是挺简单挺简洁的，不过在实际工作中，我们可能更多地会去使用线程池来管理和复用线程（而不是总是使用以上两种方法来独立创建线程）：

```kotlin
val executor = Executors.newCachedThreadPool()
executor.execute {
    // do something
}
```

一种启动协程的方式是通过协程作用域的扩展函数`launch()`，这种方式是 **非阻塞式** 的，但是 **拿不到** 执行的结果。

什么是「协程作用域」？协程作用域（CoroutineScope）首先是一个接口，用于管理协程的生命周期。其次它定义了一个作用域，所有在这个作用域内启动的协程都将受到它的控制。例如 ViewModel 库给我们提供了一个`viewModelScope`，这就是一个和 ViewModel 的生命周期绑定的作用域，当 ViewModel 的生命周期结束，该作用域也会随之结束，作用域内的所有协程都会被终止。

`launch()`又是干什么的？`launch()`是协程作用域的扩展函数，协程作用域有很多种，但是`launch()`的目的只有一个，那就是在调用它的作用域内启动协程。

接下来先来看个例子：

```kotlin
fun main() {
    // ↓ 作用域    ↓ 启动协程
    GlobalScope.launch {
        delay(1000)
        println(Thread.currentThread().name)
    }
    Thread.sleep(2000)
    println(Thread.currentThread().name)
}
```

程序运行，打印结果如下：

```
DefaultDispatcher-worker-1 @coroutine#1
main
```

在上面的代码中，`GlobalScope`虽然是一个可以启动协程的协程作用域，但是并不推荐使用它。一个更加贴合实际的做法是使用`CoroutineScope()`这个函数来创建一个协程作用域对象，并向其传递相应的协程上下文作为参数，就像这样：

```kotlin
fun main() {
    //                         ↓ 传递参数，用于指定上下文
    val scope = CoroutineScope(Dispatchers.Default)
    scope.launch {
        println("协程启动了")
    }
}
```

「协程上下文」是什么？协程上下文（CoroutineContext）是用于提供协程执行所需的信息和功能的，它和协程作用域是协程中两个很重要的概念。在上面的例子中，我们向构造函数中传入了`Dispatchers.Default`，其中`Dispatchers`可以理解为“调度器”，而`Default`则是协程提供的 4 个调度器中的一个，完整的调度器表格如下：

| 名称 | 用途 | 线程池 |
|:---:|:---:|:-----:|
| Default | 默认调度器，用于执行计算密集型任务（一般指的是不与内存之外进行交互的操作，例如排序、拼接字符串等等）。 | 使用共享的后台线程池，为了效率最大化，该线程池的线程数通常等于 CPU 核心数。 |
| Main | 主线程调度器，通常用于更新 UI 。 | 该调度器不会使用线程池，而是直接把操作扔到主线程中去执行，比如在 Android 开发中就是这样。而如果在 Spring 或者 Ktor 这种服务端开发环境中使用了这个调度器的话就会报错，因为服务端没有这个需求。 |
| IO | I/O 调度器，用于执行 I/O 密集型任务（例如文件读写、网络请求等等）。 | 使用共享的 I/O 线程池，一般情况下，线程数为固定的 64 条，但是当 CPU 的核心数大于 64 核时，线程数将被调整为与核心数相等。 |
| Unconfined | 不限制调度器，开始时在调用者线程中运行，随后在遇到第一个挂起点时根据具体的挂起函数决定运行线程。 | 无特定线程池，适用于不需要指定特定线程的任务，或在某些情况下为了避免线程切换的开销；该调度并不如前面三个那么常用。 |

除了使用已有的调度器（每种调度器决定了使用怎么样的线程池）以外，还可以通过`newFixedThreadPoolContext()`这个函数来创建自定义的线程池：

```kotlin
fun main() {
    //                                   ↓ 线程数 ↓ 线程名称
    val pool = newFixedThreadPoolContext(8, "myPool")
    //                          ↓ 传入创建好的线程池
    val scope = CoroutineScope(pool)
    scope.launch {
        println("协程启动了")
    }
    //     ↓ 不用了要记得关掉线程池
    pool.close()
}
```

`newFixedThreadPoolContext()`接收两个参数，一个是线程的数量（整型），一个是线程的名字（字符串）。此外这个函数还被`@DelicateCoroutinesApi`修饰，意在传达这是一个需要谨慎使用的函数，原因在于创建出来的自定义线程池当不再被使用时，要记得调用`close()`将其关闭。

那为什么`Dispatchers.Default`和`Dispatchers.IO`不需要手动关闭呢？这是因为它们是由 Kotlin 协程库内部管理的全局调度器，它们的生命周期与应用程序的生命周期同步，系统会在适当的时候自动管理这些调度器的资源。

`launch()`用于启动协程并返回一个`Job`对象，要注意，该对象仅代表协程的句柄（Handle），而不是协程的执行结果。进入`launch()`内部会发现该函数接收 3 个参数，分别是：

1. 协程上下文`CoroutineContext`，默认为`EmptyCoroutineContext`，代表空上下文。
2. 协程启动模式`CoroutineStart`，默认为`CoroutineStart.DEFAULT`，代表立即执行。
3. 一个函数类型的参数：`suspend CoroutineScope.() -> Unit`，该参数就是我们要让协程执行的具体代码，在调用`launch()`的地方以 Lambda 表达式传入。此外，该参数的函数类型是挂起函数，并且是`CoroutineScope`的成员函数或者扩展函数，最后就是该函数没有返回值。

在`CoroutineScope()`中可以传入具体的上下文，在`launch()`中也可以，就像这样：

```kotlin
fun main() {
    //                         ↓ 这里先传入空的上下文
    val scope = CoroutineScope(EmptyCoroutineContext)
    //           ↓ 此处传入具体的上下文
    scope.launch(Dispatchers.Default) {
        println("协程启动了")
    }
}
```

在`launch()`中传入上下文与在`CoroutineScope()`中传入上下文的区别就在于传入的上下文是否可以被复用。具体来说，在`CoroutineScope()`创建的协程作用域内启动的协程默认都是使用这个作用域的上下文，例如我使用`CoroutineScope(Dispatchers.Default)`创建协程作用域，那么每次调用`launch()`启动的协程的上下文都是`Dispatchers.Default`。但是如果我在调用`launch()`时特地往里面传递上下文，那么启动的这个协程就会忽略所在作用域提供的上下文，并以我传递的上下文为准，非常灵活。

## 挂起函数

在 Kotlin 协程中，可以用`suspend`关键字来修饰函数，被修饰的函数被称为“挂起函数”，挂起函数只能在协程里或者是被别的挂起函数调用。协程之所以在切换线程后还能自动“切回来”，主要依靠的就是这种函数。例如：

```kotlin
fun main() {
    CoroutineScope(Dispatchers.Main).launch {
        foo()
        // 做一些界面更新操作
    }
}

// 自定义挂起函数
suspend fun foo() {
    delay(1000)  // 模拟网络请求
}
```

以上是一个定义并使用了挂起函数的简单用例，其中`foo()`为挂起函数，函数为了模拟网络请求（或者其它耗时操作），在函数体内调用了`delay()`进行了 1 秒钟的延迟。`foo()`执行的延时操作并不会阻塞主线程，这是因为它是挂起函数，它的执行伴随着协程的挂起。而在主函数中，我们首先在主线程中启动了一个协程，并在其中调用了挂起函数`foo()`，在`foo()`下面，我们可以执行一些界面更新操作（这里将这些代码省略了）。假设`foo()`返回了一些请求的结果，当我们需要将这些结果展示在界面上时，我们就可以在调用`foo()`的位置的下面去执行界面的更新操作，这是因为协程是运行在主线程中的。

### 客户端场景

上面的例子也算是协程在涉及界面（例如 Android 客户端开发）的项目中一个比较常见的做法：在主线程启动一个协程，然后在其中通过调用挂起函数来执行耗时任务，执行完成后再回到主线程，这样就可以把耗时操作扔到后台去，从而不影响界面交互和响应。

值得注意的是，在这个过程中，我们通常也不会通过`CoroutineScope(Dispatchers.Main)`来创建位于主线程的协程，而是通过 Jetpack 库里已有的接口。例如`lifecycleScope`——这是`LifecycleOwner`这个接口的扩展属性，而`LifecycleOwner`这个接口有两个很典型的实现类，一个是位于`androidx`之下的`ComponentActivity`，也就是 Activity ，另一个则是 Fragment ，也是位于`androidx`之下的。也就是说，在这两个类里，我们可以直接调用`lifecycleScope`。

`lifecycleScope`是一个现成的协程作用域（可以用来直接启动协程），其特点之一在于它启动的协程默认使用的是`Dispatchers.Main`上下文，这和我们上面手动指定`Dispatchers.Main`的例子的效果是一样的。但是严谨来说，`lifecycleScope`内部使用的上下文其实是`Dispatchers.Main.immediate`，它与`Dispatchers.Main`的区别在于后者启动的协程不管当前是在什么线程，都会把任务以 Handler.post() 的方式发布到主线程的消息队列中；而前者则会先判断一下当前所在线程，如果不在主线程，那么就和后者做一样的事，如果当前已经在主线程中了，那就会直接执行协程中的代码。总之结论就是尽可能使用`Dispatchers.Main.immediate`，因为它是经过性能优化的，换句话说，可以的话，请使用`lifecycleScope`而不是`CoroutineScope(Dispatchers.Main)`。

`lifecycleScope`的特点之二是其会与所在组件的生命周期进行绑定，例如在 Activity 里调用，那么它就会与 Activity 进行绑定，当 Activity 销毁时，`lifecycleScope`会自动去取消该作用域内的所有协程，从而达到避免内存泄漏和不必要资源浪费的目的。

除了`lifecycleScope`以外，Jetpack 中的 ViewModel 组件也提供了类似的作用域——`viewModelScope`。`viewModelScope`是 ViewModel 的扩展属性，所以只能在 ViewModel 中使用，这在我们进行界面数据管理时帮助很大。除此之外，它和`lifecycleScope`基本一样，`viewModelScope`使用的上下文也是`Dispatchers.Main.immediate`，并且也会绑定生命周期，自动释放资源。

### withContext

`withContext()`就是一个挂起函数，一般用于在协程中切换上下文。它与`launch()`的区别在于，`launch()`启动的是并行的协程，例如：

```kotlin
CoroutineScope(Dispatchers.Main).launch { 
    launch(Dispatchers.IO) { 
        
    }
    println()
}
```

所谓「并行」，就是`launch(Dispatchers.IO)`这行代码执行完后，下面的打印就会接着立刻执行，而不会理会上面启动协程的代码的运行情况，无论启动的这个协程是否执行完成，下面的代码都会继续执行。

而，如果想要串行地切换协程上下文，就要使用挂起函数，就像前面讲到的客户端开发场景一样，使用挂起函数去执行耗时操作，执行完成后，再回到主线程执行更新界面的操作。这就是「串行」，一件事情到一件事情。例如：

```kotlin
CoroutineScope(Dispatchers.Main).launch {
    // ↓ 把 launch() 改成了 withContext()
    withContext(Dispatchers.IO) {

    }
    println()
}
```

`withContext()`也接收一个协程上下文的参数，用于指定它切换到哪个调度器上。在上面的例子中，`println()`以及它以下的代码（如果它下面还有代码的话）就会在`withContext()`中的代码执行完成后再执行，并且`withContext()`中的代码的上下文是`Dispatchers.IO`，而`withContext()`外的代码的上下文还是`Dispatchers.Main`。

`withContext()`的用法基本上就是这样，还有一点值得注意的是，`withContext()`是有返回值的，它的 Lambda 表达式的最后一行将作为协程执行结果返回。

接下来再讲一下使用类似于`withContext()`这种官方提供的挂起函数的时候的一个比较好的编码习惯，且看下面这个例子：

```kotlin
CoroutineScope(Dispatchers.Main).launch {  
    val data = withContext(Dispatchers.IO) {  
        delay(2000)  // 模拟网络请求  
        "data"  // 返回请求到的结果  
    }  
    println("请求结果：$data")  // 打印请求结果  
}
```

在上面的代码中，我首先创建了一个使用主线程上下文的协程作用域并启动了一个协程，随后在这个协程中，我又通过`withContext()`切换到另一个基于 IO 上下文的协程中去进行网络请求，最后将请求到的结果进行打印。

上面的代码在功能上来说是没问题的，但是当我们为了追求更好的可读性和复用性的时候，可能会考虑对结构进行以下优化：

```kotlin
CoroutineScope(Dispatchers.Main).launch {  
    //         ↓ 将切换到 IO 上下文做网络请求的操作单独提取至一个函数中  
    val data = getData()  
    println("请求结果：$data")  // 打印请求结果  
}
```

```kotlin
private suspend fun getData() = withContext(Dispatchers.IO) {  
    delay(2000)  // 模拟网络请求  
    "data"  // 返回请求到的结果  
}
```

这个改动的重点在于，将`withContext()`连同其它操作一并提取出来，而非是像下面这样，只提取业务逻辑（以下代码仅作为示例，并非推崇的做法）：

```kotlin
CoroutineScope(Dispatchers.Main).launch {  
    val data = withContext(Dispatchers.IO) {  
        getData()  
    }  
    println("请求结果：$data")  // 打印请求结果  
}
```

```kotlin
private suspend fun getData(): String {  
    delay(2000)  // 模拟网络请求  
    return "data"  // 返回请求到的结果  
}
```

这样的改动除了提高可读性以外，还可以确保网络请求正确无误地在 IO 上下文中执行，并且就算是在`Dispatchers.IO`中调用`getData()`，也不会出现性能问题。

### 运行原理

前面提到挂起函数可以自动切换到后台线程，然后在执行完任务后又切换回主线程执行剩下的任务，挨在一起的上下两行代码就有可能出现上面的代码执行在后台线程，下面的代码执行在主线程的情况，这样的特性虽然带来了简洁和高效的写法，但是却也容易误导开发者，因为它隐藏了一些细节。

```kotlin
CoroutineScope(Dispatchers.Main).launch {  
    val data = withContext(Dispatchers.IO) {  
        delay(2000)  // 模拟网络请求  
        "data"  // 返回请求到的结果  
    }  
    println("请求结果：$data")  // 打印请求结果  
}
```

比如还是之前的这个例子，`withContext()`中的代码就是在 IO 上下文中执行的，赋值给变量`data`的操作和下面打印的操作是在 Main 上下文中执行的，这些在 Main 上下文的操作都会等到 IO 上下文的操作执行完成后再执行，原因前面也说过了（`withContext()`是串行切换协程的）。这就带来一个问题：这些在 Main 上下文的操作既然要等，那这样的等待难道不会导致主线程阻塞吗？那肯定是不会的，答案我们都能猜到，但是原因是什么呢？

首先我们要知道，在 Java 中，我们是可以在主线程去启动一个子线程的，另外，我们也可以指定某个线程池去执行某个任务，这些我们都可以做到。但是有那么几件事我们是实现不了的：

1. 从主线程切换到某个指定的子线程中；
2. 从任意子线程切换回主线程中；
3. 我们虽然能指定线程池去执行某个任务，但是具体是线程池中的哪个线程去执行，这是由线程池自己决定的，我们也无法人为干预；

但是在 Android 开发中，我们又确实存在从后台线程回到主线程的能力，用 Handler 呗，对吧。但是我们之所以能这么做，也是由 Android 本身的特性决定的，Android 的主线程或者说 UI 线程是一个无限循环的线程，它始终都在工作，比如刷新界面，然后检查消息队列里有没有需要执行的任务，所以我们才能在有需要的时候回到主线程去执行类似于更新界面之类的工作。

说到这里，我们就已经对挂起函数为什么能回到主线程有一个非常初步的认知了：不是因为它创造了 Java 中不存在的东西，而是它利用了 Android 系统的特性而已。我们接着往下说。

其实在 Android 开发中，我们还是能指定某个子线程来执行任务的，那个子线程就是`HandlerThread`，但它终究是个特例，因为它就是被官方定制过的，我们能切换到它上面也是因为它也是个无限循环的线程。

说回协程，我们直接从`launch()`入手。`launch()`是个高阶函数，我们在它的大括号里写的代码其实是被封装起来作为参数传递了，传递的目的地就是协程的上下文，传递后如何被执行则取决于上下文的实现。接下来我们就来看看协程的上下文，首先是`Dispatchers.Default`和`Dispatchers.IO`，这两个上下文都间接继承了`Executor`，表明它们内部都有着自己维护的线程池，我们在`launch()`大括号中的代码最终会被它们的`dispatch()`函数扔到它们自己的线程池当中去执行，这也就是我们能切换到后台线程的大致原理了。事实上，`Dispatchers.Main`也是类似这样工作的，因为它也有一个`dispatch()`函数，在调用这个函数前，它会去获取主线程的 Looper ，然后在这个函数中，再通过 handler.post() 去将代码传递到主线程中。

所以说白了协程并没有跳出 Java ，Java 里切线程是基于回调的，协程里切线程也是基于回调的，只不过协程通过封装，隐藏了细节，让开发者做的事情变少了，它自己则是在后期的编译过程中去把这些操作加上。具体的做法，我们也可以参考 Retrofit ，当我们调用了`enqueue()`后，我们的网络请求任务就会被丢到线程池中，线程池将从后台启动线程来执行我们的网络请求，这就是第一次线程切换。当网络请求完成后，线程池还会把请求的结果（无论是成功还是失败）通过 Handler 扔回主线程，让主线程去执行相应的回调，这就是第二次线程切换。协程也是这样，在挂起函数执行前切换一次线程，在挂起函数执行后又切换一次线程。我们甚至可以说挂起函数和`launch()`是相似的，都是通过线程池切到后台，然后通过平台机制（Android 的 Handler）切回前台。

最后我们再回到最初的问题上，在 Main 上下文的操作要等到 IO 上下文的操作执行完成后再执行，这里的「等」只是我们的口头表述而已，实际上挂起函数里的代码早都已经离开主线程，被扔到后台线程中去执行了，等执行完了，再把结果扔回主线程，主线程在这个过程中该干嘛干嘛，等到后台线程执行完了，将结果扔到主线程的消息队列里的时候，主线程再去里面取出来就好了，所以也就不会存在阻塞主线程一回事了。

## 协程更轻？

Kotlin 官方文档将协程描述为一个轻量级的线程，并且还附上了代码来证明：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(50_000) { // launch a lot of coroutines
        launch {
            delay(5000L)
            print(".")
        }
    }
}
```

在这里我们启动了五万个协程，程序依旧能正常运行并完成任务，但是如果我们启动的是五万个线程的话，程序就会出现内存溢出的情况。

但是这并不能说明什么，毕竟我们都知道协程终究还是跳不出线程的，这里之所以能创建五万个协程而不会导致程序崩溃主要还是因为`runBlocking()`。究其原因就是这个函数将五万个协程都放进一个线程中去工作了，在上面的例子中就是都放进主线程去了。也就是说，如果我在 Java 线程也能有把这五万个任务都放到一个线程里去工作的方法的话，那我也不会内存溢出。

官方文档甚至还告诉我们，如果我们把示例代码中的`runBlocking()`移除，然后把`launch`换成`thread`，然后再把`delay`换成`Thread.sleep`，程序运行就会消耗大量的内存，但是文档压根就不提`delay()`和`Thread.sleep()`的区别。`delay()`是一个挂起函数，可以将协程暂停执行一段时间，它和 Java 的`Thread.sleep()`有点像，但又不一样，因为`delay()`并不会阻塞线程。`delay()`的本质让它只能成为一个「延时操作」而非「耗时操作」，但是`Thread.sleep()`是会堵塞线程的，在 sleep 的这段时间里线程什么也干不了，所以`delay()`和`Thread.sleep()`不一样。

综上所述，在线程中，如果我们既要执行类似于协程的延时操作，又要把任务都放到一个线程中去执行，那我们就可以使用`ScheduledExecutorService`的`schedule()`。就像这样：

```kotlin
val executor = Executors.newSingleThreadScheduledExecutor()  
repeat(50_000) {  
    executor.schedule({ print(".") }, 5, TimeUnit.SECONDS)  
}
```

也就是说，官方的这个文档似乎又有点误导人的嫌疑……但是我们再换个角度想，在线程中，如果要做延时操作，我们要用到`schedule()`，这个函数本身就已经有点麻烦了，更别提实际场景中还有更复杂的并发逻辑。但是在协程中，我们就只需要一句`delay()`就行了……这种简洁高效的写法，相比线程来说，也不失为一种「轻量」。

## 取消协程

取消协程可以被归纳到结构化并发的知识中，所谓结构化并发（Structured Concurrency）是一种编程模型和设计原则，旨在使并发代码更具可读性、更易于管理，并减少常见的并发错误。其核心思想是将并发任务的生命周期绑定到特定的作用域中，从而确保这些任务在作用域结束时要么完成，要么被取消。

比如，我们在启动协程时需要有一个协程作用域（CoroutineScope），这玩意儿既可以提供协程所需要的上下文，同时也具备取消协程的能力：

```kotlin
val scope = CoroutineScope(Dispatchers.Default)  
scope.launch {  
    delay(5000)  
}  
scope.cancel()
```

在客户端项目中，我们能很容易联想到一个需要取消协程的场景：例如在用户发起网络请求后，这个请求的结果将被用于更新界面，但是在获取到结果前，用户离开了，那为了避免出现资源浪费和内存泄漏的风险，我们就可以在 Activity 的`onDestroy()`中去取消协程。

除了能通过协程作用域去取消协程，对于通过`launch()`启动的单个协程，我们也可以调用它的`cancel()`来取消它：

```kotlin
//   ↓ 我们前面提到通过`launch()`启动的协程是无法返回执行结果的
//   ↓ 相对的它会返回一个`Job`对象用来管理这个协程
val job = CoroutineScope(Dispatchers.Default).launch {  
    delay(5000)  
}  
job.cancel()
```

这两个`cancel()`的区别在于，协程作用域的`cancel()`会把其作用域内启动的所有协程都取消掉，而后者只会取消其对应的那个协程。

除了这种「作用域管理」外，「父子协程关系」也是在结构化并发中需要了解的。简单来说，协程可以嵌套启动，形成父子关系：

```kotlin
//                                   ↓ 启动一个协程  
CoroutineScope(Dispatchers.Default).launch {  
    // ↓ 在内部又启动一个子协程  
    launch {   
          
    }  
}
```

父协程需要等待其所有子协程完成，然后才会自己完成。如果父协程被取消，它的所有子协程也会被递归地取消。这种关系有助于保持并发操作的一致性和可预测性。

## async() 和 join()

先来回顾一下`launch()`和挂起函数，`launch()`用来启动协程，多个相同层级的`launch()`会启动多个并行的协程，但是这些协程之间无法进行交互（比如一个协程获取另一个协程的执行结果）。而挂起函数相比`launch()`来说则更多是用在串行的工作流当中，因为它会把当前协程挂起，等后台任务执行完成后再返回到当前协程执行接下来的代码，这样一来，我们就可以轻松拿到并使用后台任务的执行结果了（比如前面提到的将网络请求得到的结果用于更新界面的例子）。

所以现在的问题在于并行的协程该如何交互，比如有一个场景是要同时发起多个网络请求，并将这些请求的结果进行统一处理，虽然`launch()`是并行的，但是它并不能返回执行结果。面对这种情况，我们就需要引入`async()`了。

`async()`也是一种启动协程的方式，它和`launch()`一样是非阻塞式的，不同的地方在于`async()`会返回一个`Deferred`对象，这个对象可以在将来某个时间点通过调用它的`await()`来获取协程的执行结果，所以`async`通常与`await()`一起使用。例如：

```kotlin
// 模拟第一个网络请求，获取用户名  
suspend fun getUserName(): String {  
    delay(1000)  
    return "Aiden"  
}  
  
// 模拟第二个网络请求，获取用户年龄  
suspend fun getUserAge(): Int {  
    delay(1000)  
    return 24  
}
```

上面的代码定义了两个挂起函数，用于模拟两个网络请求。

```kotlin
val deferred = lifecycleScope.async {  
    getUserName()  
}  
lifecycleScope.launch {  
    val age = getUserAge()  
    val name = deferred.await()  
    println("Name: $name, Age: $age")  
}
```

1. 接下来首先使用`async`启动一个协程，并在其中调用获取用户名的挂起函数，同时定义一个变量用来持有`async`返回的`Deferred`对象。
2. 然后再用`launch()`启动一个并行的协程，并在其中先调用获取用户年龄的挂起函数，此时两个网络请求都在运行，当获取用户年龄的挂起函数执行完成后会返回到当前线程进行赋值。
3. 赋值完成便来到下一行，通过调用前面保存的`Deferred`对象的`await()`来拿到获取用户名的挂起函数的执行结果，此时如果这个挂起函数已经执行完成，那这个地方就能正常拿到结果，否则`await()`就会将协程挂起，并等待获取用户名的挂起函数执行完成，因为`await()`也是个挂起函数。
4. 当用户年龄和用户名都拿到以后，最后就整合并打印信息。

事实上，第一个部分的代码（那两个模拟网络请求的挂起函数）在不变的情况下，第二部分的代码还是可以再调整一下，使其可读性会高一些且易于管理：

```kotlin
lifecycleScope.launch {  
    val name = async { getUserName() }  
    val age = async { getUserAge() }  
    println("Name: ${name.await()}, Age: ${age.await()}")  
}
```

-----

接下来再说说`join()`，这个函数是个挂起函数，它的功能就好像它的名字一样，在调用它的地方将协程挂起，等待被调用者（也是个协程）里的任务执行完成后，再恢复协程。

可以想象这么一个开发场景：某个程序启动了多个并行的协程，其中一个用于初始化工作，并且在后续的工作中有需要依赖这个初始化工作的结果的地方，那么在开始后续的工作之前，我可以先通过调用`join()`来确保这个负责初始化工作的协程已经执行完成：

```kotlin
lifecycleScope.launch {  
    val initJob = launch {  
        delay(3000)  // 模拟初始化工作  
    }  
    launch {  
        // 假设在这里开启了其它的并行协程  
    }  
    //       ↓ 等待初始化完成  
    initJob.join()  
    //  进行后续的工作  
}
```

其实这里也可以把`launch()`换成`async()`，把`join()`换成`await()`，但是我们的初始化工作并没有返回值，就没必要使用`async()`。说到底设计`async()`和`join()`的目的就是不一样的，当我们需要等待某个协程执行完成，但是这个协程并不返回执行结果时，就可以使用`join()`来确保程序执行的正确性。

## suspendCoroutine()

`suspendCoroutine()`是一个挂起函数，其作用就是包裹住那些使用传统回调机制的代码，使其整体上成为协程代码，以适配项目其它地方的协程风格。

`suspendCoroutine()`的使用要点就是把原本的回调机制结构的代码直接放进`suspendCoroutine()`中，然后在回调函数中对结果的返回方式进行修改，前者很简单，问题在于后者应该怎么改。比如使用传统的回调机制结构的代码进行了网络请求，请求成功要返回请求结果，请求失败要返回 Exception ，这是第一个点。第二个点是在传统的回调机制的代码中，我们有时候会在回调函数里去进行接下来的工作，比如回调函数返回了网络请求的结果，我们拿到结果后就直接在回调函数里更新界面了。

基于以上这两个点，我们将引入`Continuation`接口的`resume(value: T)`，这个函数接收一个泛型参数，其功能用于恢复已挂起的协程，并将接收到的参数作为返回值进行返回。所以在上面提到的场景中，对于回调函数提供的网络请求成功的结果，我们应该直接使用`resume(value: T)`对其进行返回，随后在挂起函数的调用处在对返回的结果进行应用。

除了`resume(value: T)`以外，`Continuation`接口中还有一个函数是`resumeWithException(exception: Throwable)`，这个函数同样用来恢复挂起函数，并且也有返回值，只不过它所返回的是异常。在网络请求等场景中，我们通常都会有成功和失败两种情况，对于失败这条分支而言，我们通常就会调用`resumeWithException(exception: Throwable)`将异常抛出，随后在挂起函数的调用处在对异常进行捕获。

接下来是一个极度简化的例子，首先是在没有使用`suspendCoroutine()`的情况下：

```kotlin
/**  
 * 网络请求回调接口  
 */  
interface NetworkCallback {  
    fun onSuccess(response: String)  
    fun onFailure(error: Throwable)  
}  
  
/**  
 * 模拟网络请求  
 */  
fun makeNetworkRequest(callback: NetworkCallback) {  
    val success = true  // 假设请求成功或失败  
  
    if (success) {  
        // 模拟成功的响应  
        val response = "请求成功，这是结果："  
        callback.onSuccess(response)  
    } else {  
        // 模拟失败的响应  
        val error = Exception("请求失败，这是原因：")  
        callback.onFailure(error)  
    }  
}
```

接着我在 Activity 中去开启一个子线程然后调用函数进行网络请求：

```kotlin
thread {  
    makeNetworkRequest(object : NetworkCallback {  
        override fun onSuccess(response: String) {  
            println(response)  
        }  
  
        override fun onFailure(error: Throwable) {  
            println(error.message)  
        }  
    })  
}
```

现在我们再来看看使用`suspendCoroutine()`的版本，网络请求回调接口和模拟网络请求的代码都不需要变更，我们首先新增一个挂起函数：

```kotlin
suspend fun suspendMakeNetworkRequest() = suspendCoroutine {  
    makeNetworkRequest(object : NetworkCallback {  
        override fun onSuccess(response: String) {  
            it.resume(response)  
        }  
  
        override fun onFailure(error: Throwable) {  
            it.resumeWithException(error)  
        }  
    })  
}
```

可以看到，我直接把之前网络请求的代码复制粘贴进来了，并且修改了成功和失败两个回调函数对于结果的处理方式，如果成功，就返回结果，如果失败，就抛出异常。

Activity 中的代码也需要改成：

```kotlin
lifecycleScope.launch {  
    try {  
        val result = suspendMakeNetworkRequest()  
        println("Result: $result")  
    } catch (e: Exception) {  
        println("Error: ${e.message}")  
    }  
}
```

其中`val result = suspendMakeNetworkRequest()`这一行代码就跟前面很多使用挂起函数的例子一样了，如此一来便完成了将使用线程的代码封装成挂起函数，从而实现了传统回调机制结构的代码风格到协程代码风格的转变。

值得注意的是，在这里我还是手动编写了 try catch 语句块用于捕获异常，这个异常如果不在这里捕获，那么它就会被丢给更外层的协程去处理。另外对于协程异常的捕获，存在一种错误的写法，同样以上面的代码为例：

```kotlin
lifecycleScope.launch {  
    try {  
        // ↓ 额外启动了一个协程  
        launch {  
            val result = suspendMakeNetworkRequest()  
            println("Result: $result")  
        }  
    } catch (e: Exception) {  
        println("Error: ${e.message}")  
    }  
}
```

在上面的代码中，launch 代码块内的异常，try 是捕获不到的，这是因为 launch 只是用来启动协程，它一执行完，try catch 语句块也跟着结束了。

-----

我们最后再来看一个相关的函数——`suspendCancellableCoroutine()`，看名字我们就大概能猜到，这是一个支持取消的`suspendCoroutine()`。

前面我们提到，想要取消一个协程，直接调用它的`cancel()`就行了。但是有一个前提我们也需要知道，那就是哪怕是线程的取消，它也只是一个标记位而已，至于能不能立刻取消，那也得看 JVM 怎么操作。同样的，协程的取消，也要挂起函数配合，所以如果在协程中遇到线程的 API ，那就算你调用`cancel()`，也无法保证协程一定能取消。

不仅如此，`suspendCoroutine()`它虽然是个挂起函数，但是它也不配合协程的取消，所以我们才要使用到`suspendCancellableCoroutine()`，接下来我们以上面的代码为基础写一个示例：

```kotlin
//          ↓ 首先是这个函数这里需要把 suspendCoroutine 替换成 suspendCancellableCoroutine
suspend fun suspendMakeNetworkRequest() = suspendCancellableCoroutine {  
    // 省略其余代码 
}
```

```kotlin
//   ↓ 随后定义一个变量持有这个作用域  
val job = lifecycleScope.launch {  
    try {  
        val result = suspendMakeNetworkRequest()  
        println("Result: $result")  
    } catch (e: Exception) {  
        println("Error: ${e.message}")  
    }  
}  
job.cancel()  // 在这里调用作用域的 cancel() 用于取消协程
```

这样一来，假设网络请求需要消耗一定的时间，比如说几秒钟，那么上面的代码就会在网络请求还没完成前就把协程取消掉了。

另外，当我们需要`suspendCancellableCoroutine()`在被取消前执行一些收尾工作时，就可以这么写：

```kotlin
suspend fun suspendMakeNetworkRequest() = suspendCancellableCoroutine {  
    it.invokeOnCancellation {  
        // 在这里做一些取消前的收尾工作  
    }  
  
    // 省略其余代码
}
```

最后总结一下：

- 当我们需要将传统的回调机制的代码封装成挂起函数时，可以使用`suspendCoroutine()`，大致做法就是将回调机制的代码都复制进去，然后改一下返回结果的方式。
	- 正常返回使用`resume(value: T)`；
	- 抛出异常使用`resumeWithException(exception: Throwable)`；
- 当我们将传统的回调机制的代码封装成挂起函数，并且希望其支持取消功能时，就可以使用`suspendCancellableCoroutine()`。
	- 取消仍然是通过调用`cancel()`；
	- 可以通过`invokeOnCancellation()`设置取消前的收尾工作，注意这是在协程被取消时才会执行的工作，如果是正常的收尾工作则该写哪就在哪写。

## runBlocking()

`runBlocking()`也是一个用于启动协程的函数，它和前面`launch()`、`async()`的区别有两点，一是它的执行不需要协程作用域，之所以如此，是因为它不需要我们前面提到的协程作用域提供的功能（协程上下文和取消协程）。第二个区别就是它的运行会阻塞当前线程，无论它是运行在线程的代码里还是协程的代码里，阻塞一直持续到它自己代码块内的代码执行完成后才会恢复线程。

`runBlocking()`之所以这样设计也是和它的定位有关。前面我们提到可以通过`suspendCoroutine()`将线程风格代码封装成协程风格的代码，而`runBlocking()`的作用就刚好反过来。这么一来，无论是线程的代码还是协程的代码，我们都能在它们之间来回交互了。

在后端开发中，还可以通过等号将`runBlocking()`与`main()`函数相连，以此为整个`main()`函数提供协程作用域：

```kotlin
fun main() = runBlocking<Unit> {  
    launch {   
// 可以直接启动协程  
    }  
}
```

除此之外，`runBlocking()`还可以用来测试代码。总之，`runBlocking()`是一个特殊的启动协程的函数，但是其主要目的不在于启动协程，而是将协程风格的代码封装成阻塞式的，以便传统线程风格代码调用。