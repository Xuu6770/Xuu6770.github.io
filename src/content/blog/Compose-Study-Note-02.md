---
title: Compose 学习笔记 ②
date: 2024-01-05 19:12:27
category: 笔记
---
这一节进一步讲一些 Compose 的概念，包括 Composable 、界面自动更新、架构原则、可被订阅内部变化的集合、CompositionLocal 等。

<!-- more -->

## Composable

首先是一些有关 Composable 的基础信息：

- Composable 有点像传统 XML 布局写法中的 View ，例如 Compose 中的文本控件`Text()`和按钮控件`Button()`就是一个 Composable 。
- 从写法上来看，Composable 就是一个被`@Composable`注解修饰的函数。
- View 可以自定义，Composable 也可以自定义（但是两者不一样），具体做法就是编写一个函数，然后给它添加上`@Composable`注解即可。
- 被`@Composable`注解修饰的函数在编译时，Compose 的编译器插件（Compiler Plugin）会在幕后进行一些操作，例如为这些函数添加额外的参数（比如会添加一个`Composer`类型的参数）。
- Compose 的编译器插件（Compiler Plugin）在这里体现了 AOP（面向切面编程）的理念，并且以“插件”的形式来实现 AOP ，不仅做的事情更多，也可以更好地支持跨平台。
- Composable 官方翻译为“可组合项”，并且 Composable 只能在 Composable 中被调用。

### setContent()

首先要提到的就是`setContent()`，`setContent()`就像是 Compose 代码的入口，我们平时就在它的 Lambda（大括号）内通过调用 Composable 来设计界面。值得注意的是：`setContent()`这个函数并没有被`@Composable`注解修饰，这并不符合前面提到的“Composable 只能在 Composable 中被调用”的规则……在这里不妨先来看一下`setContent()`的函数签名：

```kotlin
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
)
```

可以看到，`setContent()`是一个高阶函数，在`MainActivity`中，`setContent()`的 Lambda 内的代码都被打包成了一个函数类型的参数，也就是这些代码实际上是被“传递”而不是被“调用”的，所以`setContent()`本身并不需要被`@Composable`注解修饰。但是问题就来了，Compose 要求我们在 Composable 中调用 Composable ，但是`setContent()`又不是 Composable ，既然如此，最初的那个 Composable 是在哪里被调用的呢？

`setContent()`不是那个最初调用 Composable 的，真正作为一般函数和 Composable 函数之间的桥梁的是`internal fun invokeComposable(composer: Composer, composable: @Composable () -> Unit)`，它成为了调用 Composable 的特例。在这个函数中，`composable`这个函数类型的参数被强转成了一般函数并在最后被调用了，而一旦有一个 Composable 被调用了，那么接下来就在这个 Composable 中调用其它的 Composable 就行了。

### 自定义 Composable

自定义 Composable 是什么？它做到了什么传统 View 写法做不到的事情吗？

前面提到 View 可以自定义，Composable 也可以自定义，但是两者不一样，至于为什么，且看下面这个例子：

```kotlin
@Composable
fun Foo() {
    Column {
        Text(text = "Name: Aiden")
        Button(onClick = { /*TODO*/ }) {
            Text(text = "Click")
        }
    }
}
```

上面是我自定义的一个 Composable ，主要有 3 部分：

1. 调用`Column()`创建一个纵向布局；
2. 在`Column()`内调用`Text()`创建一个文本控件显示一串文本；
3. 在`Text()`下方调用`Button()`创建一个按钮，按钮上显示“Click”；

这是个很简单的自定义 Composable ，我只要在需要的地方调用`Foo()`就可以重用这个界面。但是同样的需求，如果是在传统的 View 系统中我要怎么实现呢？我并不会去使用自定义 View ，因为它太复杂了，有点牛刀杀鸡的感觉。相对的，我只需要写一个 XML 布局文件就可以了，因为这只是涉及到简单的显示而已，这就是自定义 View 和自定义 Composable 之间不能直接划等号的原因。

那自定义 Composable 可以和 XML 文件划等号吗？再看下面这个例子：

```kotlin
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}
```

这是新版 Android Studio 创建的 Compose 项目中的示例代码，可以看到相比我上面那个自定义 Composable 来说，`Greeting()`多了两个函数参数，这就意味着你在调用`Greeting()`的时候需要向其中传入参数，并且`Greeting()`还会根据传入的参数来显示不同的内容。这种动态显示内容甚至可以根据业务逻辑来显示内容的功能是 XML 写法所不具备的，XML 就是单纯的标记语言，是写死的，这也是前面说到为什么“涉及到简单的显示”的时候会使用 XML 。所以如果我想要在 View 系统中具备这种“动态显示”的能力，我就只能回到自定义 View 上了。

总结来说：

- 在传统 View 系统中并没有一个能和自定义 Composable 完全等价的存在；
- 在显示简单内容时，自定义 Composable 比自定义 View 要方便多了；
- 在显示复杂内容时，自定义 Composable 可以做到 XML 做不到的事情；

自定义 Composable == 自定义 View + XML 。

## MutableState

之前说过 Compose 的写法属于声明式，特点在于 UI 会自动更新，接下来就大致说说这个自动更新是什么情况。

### 被订阅的对象

首先是一个很简单的例子，有一个文本控件`Text()`，当向其中传入的文本发生变化时，界面就会自动更新，随后显示新的内容：

```kotlin
var name = "Aiden"
setContent {
	Text(text = name)
}
```

上面的代码在逻辑上看起来好像没问题，定义一个变量`name`并将其作为参数传递到文本控件`Text()`中，之后我只需要更改变量`name`的值，文本控件应该就会自动更新显示相应的新内容……然而实际并不是这样。

这是因为我们一直以来写代码时用到的这些类型（无论是`String`、`Int`等等）是没办法直接和 Compose 进行合作的。简单来说，`name`确实作为参数传递给`Text()`了，后期你也确实修改`name`的值了，但是`Text()`并不知道它发生了改变，不仅是`Text()`，所有的 Composable 都不知道，也就没办法刷新界面了。

这里就需要引入`MutableState`这个对象，它可以被 Compose 监听，当它的值发生变化时，界面会去进行响应：

```kotlin
val name = mutableStateOf("Aiden")
setContent {
	Text(text = name.value)
}
```

上面的代码中使用了`mutableStateOf()`创建了一个`MutableState`的对象，这个对象当中包裹着一个`String`类型的值，当这个的值发生变化时，`MutableState`让它的变化可以被调用它的地方感知到，从而去更新界面——也就是上面例子中的`Text()`。此外就是向`Text()`中传入的参数不再是`name`而是`name.value`了，这是因为变量`name`已经不再是`String`类型了。

### 初探自动更新

现在我们知道了当我们需要读取或是修改一个`MutableState`对象的值的时候，实际上是要调用它的`value`属性，同时我们也知道，读写一般会涉及到 getter 和 setter 方法，接下来我们就通过`value`属性的 getter 和 setter 方法来简单看一下界面是怎么知道该什么时候更新的。

首先是要定位到 getter 和 setter 方法的位置，先看`mutableStateOf()`这个函数：

```kotlin
@StateFactoryMarker
fun <T> mutableStateOf(
    value: T,
    policy: SnapshotMutationPolicy<T> = structuralEqualityPolicy()
): MutableState<T> = createSnapshotMutableState(value, policy)
```

可以看到它是通过调用`createSnapshotMutableState(value, policy)`来创建并返回一个`MutableState<T>`对象的，接下来就点进`createSnapshotMutableState(value, policy)`看一看：

```kotlin
internal actual fun <T> createSnapshotMutableState(
    value: T,
    policy: SnapshotMutationPolicy<T>
): SnapshotMutableState<T> = ParcelableSnapshotMutableState(value, policy)
```

同样是一个等号，继续往下：

```kotlin
internal class ParcelableSnapshotMutableState<T>(
    value: T,
    policy: SnapshotMutationPolicy<T>
) : SnapshotMutableStateImpl<T>(value, policy), Parcelable {

// 省略中间代码

}
```

`ParcelableSnapshotMutableState`类内的代码就是一些对`Parcelable`接口中的方法的重写而已，这里就省略不贴上来了。也就是说，`ParcelableSnapshotMutableState`类的作用就是在继承`SnapshotMutableStateImpl`类的基础上实现`Parcelable`接口而已，关键还得看`SnapshotMutableStateImpl`类，点进去：

```kotlin
internal open class SnapshotMutableStateImpl<T>(
    value: T,
    override val policy: SnapshotMutationPolicy<T>
) : StateObject, SnapshotMutableState<T> {
    @Suppress("UNCHECKED_CAST")
    override var value: T
        get() = next.readable(this).value
        set(value) = next.withCurrent {
            if (!policy.equivalent(it.value, value)) {
                next.overwritable(this, it) { this.value = value }
            }
        }

    private var next: StateStateRecord<T> = StateStateRecord(value)

    override val firstStateRecord: StateRecord
        get() = next

// 省略剩余代码

}
```

可以看到，这个类首先实现了`SnapshotMutableState`接口，从而间接实现了`MutableState`接口。而在`MutableState`接口中，有一个`value`属性，这就是我们在读写时调用的那个`value`。根据 Kotlin 接口的特性：接口中可以声明属性，但是属性不能被赋值，实现接口的同时需要实现接口中属性的 getter 和 setter 方法（如果属性被`var`修饰的话）。所以回到上面的代码中可以看到，`MutableState`接口中的`value`属性已经被重写了，或者说已经被实现 getter 和 setter 方法了，这样一来，我们就算找到了 getter 和 setter 方法的位置，接下来就是简单看一看这两个方法。

但是在那之前还有一点要提一下，就是到目前为止虽然都在讲`MutableState`接口，但实际上它并不具备被订阅的能力，也就是说界面的刷新和它的关系并不大，真正与之有关的是被`SnapshotMutableStateImpl`实现的`StateObject`接口，接下来先说一些和这个接口有关的内容。

点进`StateObject`这个接口中我们就可以看到这样一个属性：

```kotlin
/**
 * The first state record in a linked list of state records.
 */
val firstStateRecord: StateRecord
```

到这里又要稍微提一下，在 Compose 中，一个 state object 的状态其实并不只有一个，Compose 会将多个状态保存在一个链表中，以方便后续去进行添加或是合并的操作。从这个属性的注释中也大概可以得知这是“状态记录链表中的第一个记录”，也就是链表中的头节点。

接着我们回到`SnapshotMutableStateImpl`类中，由于`SnapshotMutableStateImpl`类实现了`StateObject`接口，所以它要实现`firstStateRecord`的 getter 方法，我们看到这个 getter 方法也很简单，就一句话`get() = next`，但是这个`next`是什么意思呢？往上看一下：

```kotlin
//                                      ↓ 直接调用构造函数实例化
private var next: StateStateRecord<T> = StateStateRecord(value)
```

可以看到`next`是一个`StateStateRecord`类型的属性，接着我们点进`StateStateRecord`看一下：

```kotlin
private class StateStateRecord<T>(myValue: T) : StateRecord() {
    override fun assign(value: StateRecord) {
        @Suppress("UNCHECKED_CAST")
        this.value = (value as StateStateRecord<T>).value
    }

    override fun create(): StateRecord = StateStateRecord(value)
    //    ↓ 把构造函数传递进来的参数也保存了一份
    var value: T = myValue
}
```

可以看到`StateStateRecord`的构造函数接收一个泛型参数，在源码中，官方是直接将`SnapshotMutableStateImpl`构造函数中同样为泛型参数的`value`作为参数传递进`StateStateRecord`的构造函数中的，并且`StateStateRecord`自己还创建了一个属性用于保存传递进来的这个 value（传递进来后的名字是`myValue`），然后这个`myValue`（其实就是`value`），其实就是最开始我们在调用`mutableStateOf()`创建一个可以被监听/订阅的对象时传递进去的值！

又因为`next`直接调用了构造函数，所以在`SnapshotMutableStateImpl`类对象实例化的时候，`next`自己也实例化了，并且在实例化后还持有着我们最开始传递进去的那个`value`，这么一看，这时候的`next`其实就是 state object 的第一个状态，或者说是链表的头节点！如此一来，`firstStateRecord`的 getter 方法为什么直接返回`next`就变得很好理解了！

还有一点就是尽管`next`是`StateStateRecord`类型，但是`StateStateRecord`也是继承自`StateRecord`的，所以`next`可以被直接赋值给`firstStateRecord`。不过既然都说到这了，不如点进`StateRecord`看一看：

```kotlin
abstract class StateRecord {

    internal var snapshotId: Int = currentSnapshot().id

    internal var next: StateRecord? = null

    abstract fun assign(value: StateRecord)

    abstract fun create(): StateRecord
}
```

可以看到不仅有一个整型属性代表 ID ，同时还有一个名为`next`的属性代表下一个节点，由此不妨大胆推测`StateStateRecord`/`StateRecord`就是存储在链表里的节点的数据类型。

接下来终于要开始看 getter 方法了，虽然它就一条语句。其中`next`已经大概知道是什么东西了，后面的`readable(this)`则是主要做了两件事：

1. `snapshot.readObserver?.invoke(state)`：通知快照观察系统（read observers）这个 state object 被读取了，并让系统对读取它的位置进行记录，后续当状态发生变化时，这个位置需要进行重组。
2. `readable(this, snapshot.id, snapshot.invalid)`：在链表中找到最新且可用的状态并返回这个状态。

最后的`.value`就简单了，因为返回的这个状态是和`next`的类型是相同的，也就是`StateStateRecord`类型，这个`.value`就是`StateStateRecord`类里的一个属性，这个属性保存着我们传递进去的值。

getter 看完了，接着来看一下 setter ，在 setter 中首先调用了`StateStateRecord`/`StateRecord`类的一个扩展函数`withCurrent()`，这个扩展函数同时还是个高阶函数，函数内部做的事情就是调用了它的函数类型的参数，并传入了一个同为`StateStateRecord`/`StateRecord`类型的对象——这个对象是怎么来的呢？实则是调用了在 getter 方法第二步那里用到的那个三参数的`readable()`，从而达到在不通知 read observers 的情况下获取链表中最新且可用的状态。

拿到状态以后我们把视角从`withCurrent()`内部拉回到 setter 方法中的`withCurrent()`的 Lambda 表达式中，在这里，我们拿到的那个状态用变量`it`代表，Compose 接下来会做下面这些事：

1. `if (!policy.equivalent(it.value, value))`：判断我们拿到的这个状态的值与 setter 方法要写的值是否相等，如果相等那就不用再赋值了，直接结束即可。
2. 如果不相等，那就需要写入新的值了，但是这又会分为两种情况：
    - 首先将拿到的这个状态的 snapshot ID 和当前快照（Snapshot）的 ID 进行比较，如果相等，则直接把新值写进拿到的这个状态中。
    - 如果不相等，Compose 会去创建一个新状态，这个状态的 snapshot ID 会和当前快照（Snapshot）的 ID 相同，最后再将新值写进这个新状态中。
3. `notifyWrite(snapshot, state)`：找到那些用过这个`MutableState`的位置，并将它们标记为失效，以便在下一次界面刷新时进行重组。

### 利用委托简化写法

到目前为止，每次在使用`MutableState`对象时都要加上`.value`才能获取到它包裹住的那个值，对此，Compose 也提供了利用 Kotlin 的委托来进行简化后的写法：

```kotlin
var name by mutableStateOf("Aiden")
setContent {
    Text(text = name)
}
```

同样是之前的例子，在使用了`by`关键字进行委托以后，每次在读写时就可以直接用变量名而不需要加上`.value`了。

当然这样的便利并不是 Compose 的特权，Compose 只是使用了 Kotlin 中的自定义委托，在`MutableState`接口的代码附近可以看到为`State`接口和`MutableState`接口编写的扩展函数：

```kotlin
inline operator fun <T> State<T>.getValue(thisObj: Any?, property: KProperty<*>): T = value

inline operator fun <T> MutableState<T>.setValue(thisObj: Any?, property: KProperty<*>, value: T) {
    this.value = value
}
```

这两个函数符合了自定义委托的规则，所以在 import 以后就可以直接使用`by`关键字了。

## remember

到目前为止我写出来的例子中，`MutableState`对象的创建都是放在 setContent 代码块之外的，如果我放在 setContent() 代码块内，就会得到 Android Studio 的一个警告：

```kotlin
setContent {
    //          ↓ mutableStateOf 会被划红色波浪线
    var name by mutableStateOf("Aiden")
    Text(text = name)
}
```

这个警告并不会阻止程序编译运行，那既然如此它为什么还要警告呢？

这是因为 setContent() 代码块内是一个 Composable ，而 Composable 是有可能会经历重组的，所谓 **重组（Recompose）** 就是界面刷新过程中的一个步骤，比如对象`name`的值由`Aiden`改成`Marcus`，这个变化就会触发重组，重组会让文本控件`Text(text = name)`显示了新的文本`Marcus`。这看起来是好的，毕竟我改了东西，界面上就应该要显示出来，但是问题在于重组这个步骤会导致 **重组作用域（Recompose scope）** 中的代码被再次执行。所谓重组作用域就是在 Composable 中的某段代码，尽管它并没有非常明确的范围提示，但是我们知道它大概就是在那个读取了发生变化的`MutableState`对象的代码的附近，就比如`Text(text = name)`这行代码的上下几行都在重组作用域中，也就是说，你明明把对象`name`的值由`Aiden`改为了`Marcus`，但是由于重组，`var name by mutableStateOf("Aiden")`这条语句又被执行了一次，导致对象`name`又被创建了一次，它的值还是`Aiden`，文本控件最终显示的也是`Aiden`……

也许你会想：“那我不在 setContent() 里面创建变量不就好了吗？”是，这样确实可以避免被 setContent() 里的 Composable 在重组时波及到，但是你并不能保证你在其它的 Composable 当中没有创建变量的需求，比如在自定义 Composable 中……

所以这时候就要用到`remember()`这个解决方案了，当我们引入`remember()`以后，Android Studio 的警告也会随之消失：

```kotlin
setContent {
    var name by remember { mutableStateOf("Aiden") }
    Text(text = name)
}
```

`remember()`其实是起到一个缓存的作用，在初次调用时会去保存函数内的对象，在后续调用中就直接返回保存的这个对象了，这样也就避免了因为再次创建对象从而产生的问题，之后如果修改了`name`的值，文本控件也能正确显示新内容了。

但是`remember()`还是很灵活的，它缓存的内容还是可以刷新的，这就要提到`remember()`还有带其它参数的重载版本了。具体来说就是可以向`remember()`中传入一个或多个 key ，这些 key 可以是任意类型，比如`String`或者`Int`等等，这些 key 起到一个控制缓存有效性的作用，当`remember()`检测到 key 发生变化时，它就会重新执行 Lambda 中的代码。来看个例子：

```kotlin
@Composable
fun HttpRequestSample(url: String) {
    //                  ↓ 传入 1 个 key
    val foo = remember(url) { "// 在这里模拟网络请求" }
    // 在这里模拟使用请求到的结果
}
```

上方的自定义 Composable 主要是模拟了一个 HTTP 请求的场景，请求的 URL 作为 key 参数传递进`remember()`中，这样一来，初次执行代码时，请求会被发起，并且请求获取到的结果会被缓存。后续如果再次执行到这段代码时，如果请求的 URL 没变化，那么`remember()`就直接返回缓存的结果就可以了，如果 URL 变了，`remember()`就会再次发起请求然后缓存结果以便下次使用。这样一来就可以避免多次发起相同的请求，节约了资源。

## 架构原则

单向数据流是 Compose 中的一个很重要的概念，但是在讲它之前，还是要先讲点铺垫的内容。

### 状态提升

首先是“状态”，状态的体现在 View 系统和 Compose 中是不同的。在之前的 View 系统中，widget（控件）都有着自己内部的状态（属性），这些状态既可以在前期通过 XML 设置，也可以在后期通过在代码中利用`findViewById()`等形式获取到对象实例后，再通过 getter / setter 方法来设置。例如 TextView ，它有着`text`这一状态，这个状态可以通过`getText()`来读取，也可以通过`setText()`来设置。

但是 Compose 中的 widget 却不是这样的，在 Compose 中同样用于显示文本的`Text()` widget 是没有内部状态的，就更别提使用 getter / setter 方法来维护状态了。这一时间也许你会觉得 Compose 很没用，因为它连获取一个文本 widget 显示的文本内容这件事都做不到。

事实倒也不是这样，前面也提到 Compose 的 widget 只是没有内部状态而已，并不是说它没有状态，而是它的内部状态被提到外部来了。就以`Text()` widget 为例：

```kotlin
val greeting = "Hello! Aiden"
setContent {
    Text(text = greeting)
}
```

`Text()` widget 中的状态`text`表示显示的文本内容，它被提到了外部变成了`Text()`这个函数的参数，当我们在调用`Text()`的时候需要向其中传入相应的内容，又因为这个参数是由我们主动传进去的，所以状态的后期维护也是可行的，就比如我现在要读取`Text()` widget 显示的内容，其实我只要读取`greeting`变量就行了，又比如我想让显示的内容变成“Hello! Marcus”，那我只需要修改变量`greeting`即可。

这种将内部的状态提到外部的模式在 Compose 中叫 **“State hoisting”** / **“状态提升”** ，这是所有 Compose widget 甚至自定义 Composable 都要遵循的模式。

### SSOT

接下来以`TextField(value = , onValueChange = )`作为例子来讲一个概念，TextField 是一个 Compose 内置的具有 Material 3 风格的文本输入框，调用它的时候会要求我们填入两个参数，根据函数签名以及注释可以知道，参数`value`的类型是`String`，它将作为显示在文本框中的文本，注意，它并不是 placeholder ，而是你在文本框中输入的文本的载体，你输入的文本会存储在这个对象中。参数`onValueChange`是一个函数类型的参数，作用是用户在文本框中进行输入操作时的回调，接下来看一个实际用例：

```kotlin
// 创建一个可被订阅的空字符串
var text by mutableStateOf("")
setContent {
    TextField(value = text, onValueChange = {
        //     ↓ it 在这里为用户输入的内容
        text = it
    })
}
```

首先`value`和`onValueChange`这两个参数都是状态提升的体现。其次是这个 widget 的工作逻辑：TextField 显示的内容被存储在`String`类型的变量`text`中，当用户进行输入操作后，新的值会通过`onValueChange`这个回调赋值给变量`text`，变量`text`发生了变化，触发重组，最后由 TextField 显示新的内容，由此形成闭环。但是如果 onValueChange 回调里面什么都不写，那么无论怎么输入，文本框都是没反应的。

一个文本框不就是让用户来进行输入操作的吗？输入的时候为什么不直接显示出来？显示内容的更新逻辑还要我自己写？这就要提到 Compose 所提倡的另一个架构原则—— **Single source of truth** / **单一数据源** / **单一可信来源** 。这个模式的最大优势在于将数据的更改行为集中到一个地方 / 保护数据不会被其它地方修改 / 易于跟踪数据的修改源，从而更易于发现 BUG 。这个理念在 TextField 等 Compose widget 上体现得淋漓尽致，接下来讲一个很简单的例子你一定会明白。

假设我要让用户在一个 TextField 中输入年龄，并且假设 Compose 已经帮我实现了 TextField 显示内容自动更新的逻辑，即用户输入的内容不会经过我，而是直接显示在 TextField 中。现在我有一个需求，我要求这个 TextField 只能输入阿拉伯数字，因为年龄就是阿拉伯数字。然后当程序运行时你就会发现出问题了，当用户输入非阿拉伯数字时，TextField 也是会接收并且显示的。对此我想到两个方案：

1. 再添加一个监听事件，把 TextField 里面的非阿拉伯数字删掉就行了；
2. 等用户把所有的表单填完，点“提交”的时候我在统一审查，不合格就不给提交；
3. ......

对于第 1 个方案，用户在输入完以后，TextField 中可能会短暂出现非法字符，虽然最后都被我删掉了，但是这个过程的体验就不好了，尽管只是一个很细小的问题。对于第 2 个方案看起来好像比第一个方案好，但是这两个方案无论是哪一个，顶多只能算是补救措施，并不满足我的需求。我的需求是这个 TextField 中完全不能出现非阿拉伯数字，之所以会出现，就是因为我假设 Compose 帮我实现了 TextField 显示内容自动更新的逻辑，导致 TextField 的数据源除了我以外，还多了个“用户输入”。我是可以在代码上去修改 TextField 显示的内容的，但是用户也可以在运行时通过和 TextField 交互来改变其显示的内容——这就乱套了。

好在实际情况是 Compose 把 TextField 内容更新的回调交由开发者来实现，这样一来，TextField 的数据来源就永远只有开发者一个了。

### UDF

在 Compose 中的 widget 一般有状态（State）和事件（Event）这么两个概念，同样以前面的 TextField 为例，`value`就是状态，而`onValueChange`就是事件。我们在使用 TextField 时会把状态`value`传递给 TextField 让它去显示内容，然后 TextField 又会把输入回调事件`onValueChange`提取出来反向传递给我们，让我们来自定义，这么一来一回的过程在 Compose 中就被称为 **Unidirectional data flow** / **单向数据流** 。

Compose 推荐将 Single source of truth（单一数据源）与 Unidirectional data flow（单向数据流）模式一起使用，此模式可以更好地保证数据一致性，不易出错、更易于调试，并且具备 SSOT 模式的所有优势。

## 可变状态集合

接下来借一个例子来简单讲一下在大多数时候应该怎样使用集合才能正常触发重组，先看下面这段代码：

```kotlin
val numberList by mutableStateOf(mutableListOf(1, 2, 3))
setContent {
	Column {
		Button(onClick = { numberList.add(numberList.last() + 1) }) {
			Text("加 1")
		}
		for (number in numberList) {
			Text(text = number.toString())
		}
	}
}
```

先解释一下代码：

1. 首先使用了`mutableStateOf()`创建了一个可变列表`numberList`，目的是希望当列表中的内容发生变化时就触发重组然后刷新界面；
2. 接着创建了一个纵向布局；
3. 在纵向布局中创建了一个显示着“加 1”的按钮，点击按钮后会把一个数字添加到`numberList`列表末尾，这个数字的值是在`numberList`列表的最后一个数字的基础上加 1 ；
4. 在按钮下面遍历`numberList`列表，并把内容显示出来；

程序运行的理想效果是：一开始会显示一个按钮以及三行数字，每当我点击一次按钮，就会多显示一行数字。

但是程序实际运行起来并不是这样，这同样是因为以往我们所使用的这些集合是没办法和 Compose 合作的，它们并不具备被订阅的功能，自然也就无法触发重组，所以如果要创建一个可被订阅的列表，应该要使用`mutableStateListOf()`：

```kotlin
val numberList = mutableStateListOf(1, 2, 3)
// 其余代码不变
```

对于这个改动主要有几点要注意：

- 一般情况下我们只会修改集合的内部，集合本身是不需要被重新赋值的，所以改为使用`val`来修饰变量而不是`var`；
- 可变状态集合类并没有实现委托，所以无法使用`by`关键字了，改为使用等号`=`；
- `mutableStateListOf()`内部不需要再调用`mutableListOf()`来创建一个可变列表了，直接把初始内容写进去即可；

这样一来，当`numberList`列表中的内容发生改变后就会正常触发重组了。

另外，除了使用`mutableStateListOf()`来创建可以被订阅的列表外，还可以使用`mutableStateMapOf()`来创建可以被订阅的 Map 。

## CompositionLocal 

CompositionLocal 中的【Composition】指的是 Compose 中的那些布局和组合，【Local】指的是局部的意思，所以 CompositionLocal 就可以理解为是在 Composable 中使用的特殊的局部变量。

### 使用场景

CompositionLocal 的使用场景目前大致有两个：

1. 提供上下文。在 Android Studio 中，当我们键入【Local】时，IDE 会为我们提供补全建议，在列表中我们可以看到很多由 Compose 官方定义好的 CompositionLocal 实例，例如`LocalContext`就是用来提供应用上下文的。
2. 主题。以具有 Material Design 风格的`Button()`为例（不是`material3`包中的`Button()`），在它的函数签名中有一个叫`colors`的参数，这个参数默认拿的就是`LocalColors`提供的颜色。

CompositionLocal 和函数参数也有一定的区别，举一个可能不太恰当的例子：比如有一个定制裤子的函数，它接收一个参数，参数是你的身高，函数需要通过你的身高来定制裤子，而此时你的身高可以通过 CompositionLocal 来获取。也就是说，函数参数更像是函数作者规定好的东西，使用函数时一定要遵守，至于参数怎么用，由函数作者来决定。而 CompositionLocal 就像是提供好的一个属性，提供者在提供时就已经和使用者达成了共识，大家都知道这个 CompositionLocal 是干嘛用的（比如`LocalContext`），接下来就是在有需要的地方去使用就好了。

### 自定义 CompositionLocal

除了使用官方提供的 CompositionLocal ，我们也可以创建自己的 CompositionLocal ：

首先需要声明一个 CompositionLocal 实例并提供默认值：

```kotlin
val LocalName = compositionLocalOf { "Aiden" }
```

随后在需要使用的地方直接调用即可：

```kotlin
@Composable
fun User() {
    Text(LocalName.current)
}
```

如果需要覆盖默认值，就这么做：

```kotlin
setContent {
	CompositionLocalProvider(LocalName provides  "Marcus") {
		User()
	}
}
```

当没有像上面这样使用`CompositionLocalProvider()`来提供值时，系统就会使用声明 CompositionLocal 实例时提供的默认值。

`CompositionLocalProvider()`可以进行嵌套使用，每一层嵌套都只会受到那一层提供的值的影响：

```kotlin
setContent {
	CompositionLocalProvider(LocalName provides "Aiden") {
		User()  // 这里显示的是 Aiden
		CompositionLocalProvider(LocalName provides "Marcus") {
			User()  // 这里显示的是 Marcus
		}
	}
}
```

`CompositionLocalProvider()`可以一次性`provides()`多个值，每个值以逗号`,`隔开：

```kotlin
val LocalName = compositionLocalOf { "Aiden" }
val LocalAge = compositionLocalOf { 24 }
```

```kotlin
setContent {
	CompositionLocalProvider(LocalName provides "Aiden", LocalAge provides 24) {
		User()
	}
}
```

使用`compositionLocalOf()`创建的 CompositionLocal 实例会被订阅，当提供用来覆盖默认值的值发生变化时，就会精准地触发重组，例如：

```kotlin
var name by mutableStateOf("Aiden")
setContent {
    //                                           ↓ 当 name 发生变化时会精准地触发`User()`的重组
	CompositionLocalProvider(LocalName provides name) {
		User()
	}
}
```

而如果是使用`staticCompositionLocalOf()`创建的 CompositionLocal 实例则不会被订阅，但是仍然会触发重组，只不过没那么精准，重组涉及的范围会变大。

也就是说，当实例会经常发生变化时，我们应该使用`compositionLocalOf()`来创建这个实例，尽管订阅实例的变化会消耗一定的性能，但是其带来的精准范围重组可以帮我们避免进行更多的计算，例如`LocalContentColor`就是用`compositionLocalOf()`创建的。而当一个实例压根不会变化时，我们应该使用`staticCompositionLocalOf()`来创建它，这样一来，重组时虽然会波及到较大的范围，但是发生重组的可能性并不大。`LocalContext`就是用`staticCompositionLocalOf()`创建的，因为上下文基本不会发生变化。