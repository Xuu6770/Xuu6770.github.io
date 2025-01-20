---
title: Kotlin 协程学习笔记 2.2
date: 2024-12-04 00:38:00
category: 笔记
---
「结构化并发」与父、子协程的生命周期关联有很大的关系，并且在讨论父、子协程的生命周期时，往往离不开 Job 。前一节也有提到，当涉及到一些「流程」相关的事务管理时，就要用到 Job 。例如使用 Job 对象的`parent`属性来获取父协程，用`children`属性来获取子协程。接下来就简单来看看父、子协程是如何体现的。

<!-- more -->

一个协程之所以能成为「父」协程通常是因为它创建了「子」协程，这可能是通过`launch()`创建的：

```kotlin
fun main() = runBlocking {
    var childJob: Job? = null
    val parentJob = CoroutineScope(EmptyCoroutineContext).launch {
        childJob = launch {
            delay(1000)
        }
    }

    // 获取子协程集合
    val childrenJobs = parentJob.children

    // 打印子协程数量
    println(childrenJobs.count())

    // 看一下这个子协程与集合里的是不是同一个
    println(childJob === childrenJobs.first())
}
```

程序运行输出：

> 1
> true

在上面的例子中，我在`parentJob`内部创建了一个子协程，并将其赋值给一开始就声明的变量`childJob`方便后续调用。在`parentJob`外部，我调用了它的`children`属性来获取它的子协程，由于`parentJob`可能不止有一个子协程，所以`children`属性实际上返回的是一个集合，不过在这个例子中，这个集合内应当只有一个对象。接下来我尝试打印这个集合中的数量，结果是`1`，和猜想的一样，同时我还使用了引用相等来判断通过`children`属性获取到的这个子协程与我在`parentJob`内部创建的是不是同一个，结果是`true`。同样的，如果利用引用相等来判断`childJob`的父协程是不是`parentJob`，得到的结果也会是`true`。

也就是说，一个协程通过`launch()`创建子协程，双方的父子关系也经由`launch()`形成，就像上面的例子一样。不过如果要从更加严谨的角度来看的话，父子关系还是要取决于`launch()`是由谁调用的，且看下面这个例子：

```kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(EmptyCoroutineContext)
    val parentJob = scope.launch {
        //  ↓ 此时 childJob 还是 parentJob 的子协程吗？
        val childJob = scope.launch {
            
        }
    }
}
```

在最上面的例子中，创建子协程时，`launch()`前面并没有写明调用者，事实上它的调用者是`this`，只不过被省略了。而在当前的例子中，尽管`childJob`嵌于`parentJob`中，但是由于它们都是由`scope`这个 CoroutineScope 对象（这个对象也是有 Job 的，如果我们没有传入 Job ，它就会自己创建一个）创建的协程，所以它们两个其实是兄弟关系而不是父子关系。

不过这还没完，`launch()`可以接收一个`CoroutineContext`类型的参数，对于这个参数，我们可以传入一个 Job 对象，而传入的这个 Job 对象，将会被指定为`launch()`新创建的子协程的父协程，也就是无论`launch()`是谁调用的，我们都可以通过传入参数的方式来手动指定父协程：

```kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(EmptyCoroutineContext)
    val parentJob = scope.launch {
        // scope.coroutineContext[Job] 可能为空，此处用双感叹号来断言
        val childJob = launch(scope.coroutineContext[Job]!!) {

        }
    }
}
```

例如这种写法和上面的例子的写法的效果是一样的，`parentJob`和`childJob`依旧是由`scope`创建的协程，它们仍然是兄弟关系。

在简单看完一个协程是如何成为父协程以及一个子协程是属于哪个父协程以后，我们再回过头来看结构化并发，这其中包含了结构化取消和结构化结束，对于前者来说，父协程一旦取消，那么它下面的子协程也会全部取消执行，而对于后者来说，父协程会等待所有的子协程执行完成之后，它才会结束。这里可以再看一个例子：

```kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(EmptyCoroutineContext)
    val parentJob = scope.launch {
        launch {
            delay(100)
        }
    }
    val startTimestamp = System.currentTimeMillis()
    parentJob.join()
    val duration = System.currentTimeMillis() - startTimestamp
    println("Duration: $duration ms")
}
```

程序运行，打印的时间大概是一百零几毫秒，但是如果此时创建的协程不属于`parentJob`的子协程的话：

```kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(EmptyCoroutineContext)
    val parentJob = scope.launch {
        //     ↓ 传入一个自定义 Job 对象作为这个协程的父协程
        launch(Job()) {
            delay(100)
        }
    }
    val startTimestamp = System.currentTimeMillis()
    parentJob.join()
    val duration = System.currentTimeMillis() - startTimestamp
    println("Duration: $duration ms")
}
```

打印的时间就只有几毫秒了，由此可以验证父协程在等待子协程执行完成。

而像这种「等待子协程完成后才会接着往下做」的特性，往往也可以应用于实际业务中的应用初始化的场景。

最后还有一点：在协程中，无论父子协程兄弟协程，所有的协程都是并行执行的，虽然父子协程的代码不能同时在`Dispatchers.Main`中运行，但是在流程上，父子协程依旧是并行关系。