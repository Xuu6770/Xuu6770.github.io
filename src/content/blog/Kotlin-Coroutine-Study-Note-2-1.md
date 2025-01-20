---
title: Kotlin 协程学习笔记 2.1 - 什么是「一个协程」
date: 2024-11-22 19:11:00
category: 笔记
---
所谓结构化并发就是父子协程之间的生命周期关系的管理。

<!-- more -->

Job 是协程中的一个关键概念，它是在执行 CoroutineScope().launch() 后返回的一个对象，CoroutineScope().launch() 则是用于启动一个协程，那么它返回的这个 Job 对象是不是就是它所启动的协程呢？之所以这么问，是因为在 Java 中，启动线程后，是可以拿到这个线程对象的，利用这个对象，我们可以执行`Thread`类的 start(), run(), interrupt() 等函数来管理线程的行为，同时在线程运行过程中，我们还能通过`name`等属性去查看线程的各种元数据。所以我才会将一个`Thread`对象理解为一个线程，这是一个实体对应关系。

进一步说，启动线程前，是需要先创建一个线程对象的，例如：

```kotlin
//    ↓ 这就是实例化后的线程对象
val myThread = Thread {
    println("Hello from thread")
}
//        ↓ 调用线程对象的函数来启动它
myThread.start()
```

但是启动协程前并没有「创建协程对象」这一步骤，而是直接调用协程作用域中的 launch() 函数即可。当然了 launch() 是会返回一个 Job 对象的前面也说了，但是这个 Job 对象确实不能代表协程本身，如果进一步看 launch() 的内部，也可以发现系统其实是创建了协程对象的，只是返回的不是协程对象而已：

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    //    ↓ 协程对象在这里被创建
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

在 launch() 的源码中可以看到系统会根据条件创建不同类型的协程对象，即要么是`LazyStandaloneCoroutine`，要么是`StandaloneCoroutine`，但是最终返回的值的类型被限定为 Job ，而 Job 是一个接口，它们之间的关系用 Java 语法来表达就是：

```text
LazyStandaloneCoroutine extends StandaloneCoroutine extends AbstractCoroutine implements Job
```

返回 Job 对象的目的之一是明确责任和分工，其二是限制我们所能做的事情，也就是我们只能调用 Job 的 API 了。而 Job 的 API 其实和线程的也很相似：

- `Job`同样有管理协程行为的函数：`start()`、`cancel()`、`join()`……
- 同样也有查看协程状态的属性：`isActive`（类似于线程的`isAlive`）、`isCancelled`、`isCompleted`（查看协程是否结束，无论是正常结束还是异常中止）……
- 此外它还可以管理协程的父子关系：`parent`（拿到父协程的 Job 对象）、`children`（拿到子协程的 Job 对象），`cancelChildren()`（取消所有子协程）……

这么看来，Job 的 API 其实就涵盖了我们对协程操作的大部分需求，甚至会让人认为 Job 就代表了协程……

不过这种理解也并非绝对错误的，我们确实可以将 Job 理解为一个由协程作用域启动的协程（无论是用 launch() 启动还是 async() 启动），但是我们还是要知道，Job 只是包含了跟协程的流程相关的功能，并非协程的全部，所以在这里还要再提一下 CoroutineScope 。

CoroutineScope 和 Job 同样都是被 AbstractCoroutine 实现的接口，它与 Job 各自有着不同的职责。例如前面提到 Job 有一个 start() 函数用来启动协程（用于启动那些创建好但是没立刻启动的，即需要手动启动的协程），尽管如此，我们平时也都还是用 CoroutineScope 的 launch() 来启动协程的。除此之外，在创建协程时，我们还可以在 CoroutineScope 中设置这个协程运行在哪个线程或者线程池上，例如：

```kotlin
//             ↓ 使用 IO 调度器来让这个协程在 IO 线程池中的线程上运行
CoroutineScope(Dispatchers.IO).launch { }
```

而在 launch() 的 Lambda 表达式内，我们也是可以通过`coroutineContext[ContinuationInterceptor]`拿到当前协程作用域的调度器的，不过值得一提的是，launch() 里面和外面的 CoroutineScope 并不是同一个：

```kotlin
fun main() = runBlocking {
    // ↓ 外部 CoroutineScope
    CoroutineScope(Dispatchers.IO).launch {
        // ↓ 内部 CoroutineScope
        this.println(coroutineContext[ContinuationInterceptor])
    }.join()
}
```

在上面的代码中，外部的 CoroutineScope ，也就是调用 launch() 创建并启动新协程的 CoroutineScope 与 launch() 的 Lambda 表达式内的`this`所指代的 CoroutineScope 并不是同一个，只不过由于 launch() 所启动的协程会默认继承调用 launch() 的 CoroutineScope 的上下文，所以打印出来的内部的 CoroutineScope 的上下文会和外部调用 launch() 的 CoroutineScope 的上下文一样都是`Dispatchers.IO`。

> 但是内部的 CoroutineScope 是哪来的呢？

这是我自己遇到的问题，这里也记录一下。首先还是先来简单看看 launch() 函数内部都做了什么：

```kotlin
public fun CoroutineScope.launch(
    // ↓ 指定协程作用域
    context: CoroutineContext = EmptyCoroutineContext,
    // ↓ 设置协程启动模式，例如 LAZY 就只是创建而不启动协程
    start: CoroutineStart = CoroutineStart.DEFAULT,
    // ↓ 挂起函数类型的参数，需要由 CoroutineScope 对象调用，内部是协程要在后台执行的代码
    block: suspend CoroutineScope.() -> Unit
): Job {
    //  ↓ 使用参数重的上下文来创建一个新的上下文
    val newContext = newCoroutineContext(context)
    //  ↓ 根据参数`start`的类型来决定创建什么类型的协程
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    //        ↓ 启动协程，将「启动模式、协程本体、协程中要执行的代码」这三个参数传入
    coroutine.start(start, coroutine, block)
    //     ↓ 将协程以 Job 接口形式返回，通过 Job 接口的 API 来管理协程的生命周期和流程
    return coroutine
}
```

其实我写的这些注释有些多余了，源码中都有，或者是我自己上面也有写。我直接说重点吧，launch() 函数中的`block`是挂起函数类型的参数，根据签名和 Kotlin 语法来看，只有 CoroutineScope 对象能调用这个`block`，而调用这个`block`的 CoroutineScope ，就是我们前面所说的「内部的 CoroutineScope」，也就是大括号内的，或者说是 Lambda 表达式内的 CoroutineScope 。但是这个 CoroutineScope 是在哪里被创建的呢？或者说是在哪里被实例化的呢？然后`block`又是在哪里被调用了呢？

CoroutineScope 是在这里被创建的：

```kotlin
val coroutine = if (start.isLazy)
    LazyStandaloneCoroutine(newContext, block) else
    StandaloneCoroutine(newContext, active = true)
```

> LazyStandaloneCoroutine 和 StandaloneCoroutine 不是协程本体吗？我没看到`CoroutineScope()`这个字眼啊？

哈！那是因为我前面已经说了，CoroutineScope 和 Job 同样都是被 AbstractCoroutine 实现的接口，而 AbstractCoroutine 是 StandaloneCoroutine 的父类，然后 StandaloneCoroutine 又是 LazyStandaloneCoroutine 的父类，也就是说，它们都是实现了 CoroutineScope 接口的 CoroutineScope 对象！所以实例化一个 LazyStandaloneCoroutine 对象或者实例化一个 StandaloneCoroutine 对象，都等于是实例化了一个 CoroutineScope 对象，既然它们都是 CoroutineScope 对象，那调用`block`就是可行的了，那么好了，请问`block`在哪呢？我猜是在`coroutine.start(start, coroutine, block)`这个语句的 start() 函数内部，但是我点进去看了，内部又调用了另一个 3 参数的 start() 函数，而这个 start() 函数可能是重载函数，我也无法再点进去看了……既然如此那就这样吧，我觉得应该是不会错的了，在最终的实现里面可能是有一行代码是用 CoroutineScope 来调用这个 block 的。

OK ，现在再拐回来，前面说到可以从 CoroutineScope 中通过`coroutineContext`拿到调度器，其实还可以通过它拿到 Job 对象，而这个 Job 对象就是这个协程的 Job 对象，也是创建这个协程的 launch() 函数要返回的 Job 对象，返回给调用处，用来对这个协程的流程进行管理：

```kotlin
fun main() = runBlocking {
    var innerJob: Job? = null
    val outerJob = CoroutineScope(Dispatchers.IO).launch {
        innerJob = coroutineContext[Job]
    }
    outerJob.join()
    print(outerJob === innerJob)
}
```

上面的代码的打印结果为`true`，代表 launch() 的 Lambda 表达式内的 Job 与 launch() 要返回的 Job 是同一个对象。

总结一下，Job 是创建协程时返回的一个句柄，用于控制协程的流程，而 CoroutineScope 则是类似于「大总管」的角色，它管理的是协程更加顶层的东西。从代码上来看，Job 和 CoroutineScope 其实就是一个对象：在 launch() 函数中，调用`block`参数的 CoroutineScope 对象是变量`coroutine`，然后 launch() 函数返回的 Job 对象还是变量`coroutine`，一个对象以两个身份出现，其目的就是让责任和分工更加明确，避免 API 污染，

让需要用到 CoroutineScope 的地方出现的就是 CoroutineScope ，例如获取协程上下文信息和开启子协程：

```kotlin
fun main() = runBlocking {
    val myJob = CoroutineScope(Dispatchers.IO).launch {
        // ↓ 打印上下文信息
        print(coroutineContext[ContinuationInterceptor])
        // ↓ 启动子协程
        launch {
            
        }
    }
}
```

让需要用到 Job 的地方出现的就是 Job ，例如控制协程的流程：

```kotlin
fun main() = runBlocking {
    val myJob = CoroutineScope(Dispatchers.IO).launch {
        
    }
    // ↓ 通过调用 Job 对象的 API 来取消协程
    myJob.cancel()
}
```

在理解层面上，不严谨地来说，我们既可以认为 launch() 返回的 Job 对象是协程的代表，也可以认为 launch() 的 Lambda 表达式内的 CoroutineScope 是协程的代表。这其实也是本节的主题：从理解「一个协程」的含义切入，简单介绍一下 Job 和 CoroutineScope 的作用以及它们的关系。所以 Java 线程就不像 Kotlin 协程了，线程没有这种功能拆分，线程本身就是「大总管」。