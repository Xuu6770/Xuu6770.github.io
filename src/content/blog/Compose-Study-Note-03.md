---
title: Compose 学习笔记 ③ - 性能优化
date: 2024-01-09 15:20:00
category: 笔记
---
本节接着说一下 Compose 为了「自动更新」这个特性所带来的性能缺陷而做了哪些优化。

<!-- more -->

除了我们之前提到的使用`remember()`来进行缓存，在保证正确性同时提高性能以外，Compose 的另一项优化措施就是在重组过程中会自动判断某些 Composable 是否需要被再次执行，不过这也分几个场景，我们一个一个来看。

## 不跳过

首先是 Compose 对于那些不会跳过的代码的例子：

```kotlin
var name by mutableStateOf("Aiden")
setContent {
	Log.d("ScopeTest", "在 Column 的上面")
	Column {
		Text(text = name)
		Log.d("ScopeTest", "在 Column 的里面")
		Button(onClick = { name = "Marcus" }) {
			Text(text = "改变名称")
		}
	}
	Log.d("ScopeTest", "在 Column 的下面")
}
```

上面这段代码就是在程序运行后先打印 3 条 Log ，然后在点击按钮更改变量`name`的值后触发重组，让`Column()`内部的代码再次被执行，从而让界面显示新的内容，同时我非常肯定是`Column()`内部发生了重组。然而实际上当我按下按钮以后，被再次执行的不止有`Column()`内部的代码，和`Column()`处于同一层级的上下两条打印 Log 的语句也被再次执行了……

这实际上是因为：

1. `Column()`是个内联函数，在编译时其所在位置的代码被其函数内的代码替换了；
2. `Column()`函数内的核心代码其实就是调用了它的函数类型的参数`content`而已；

所以上面的`Column()`在编译时大概会变成这样：

```kotlin
var name by mutableStateOf("Aiden")
setContent {
	Log.d("ScopeTest", "在 Column 的上面")
	Text(text = name)
	Log.d("ScopeTest", "在 Column 的里面")
	Button(onClick = { name = "Marcus" }) {
		Text(text = "改变名称")
	}
	Log.d("ScopeTest", "在 Column 的下面")
}
```

所以实际上是因为上下这两条打印 Log 的语句和`Column()`是位于同一层级的，所以它们也被`Column()`的重组波及到了。更糟糕的是，如果被波及的只是打印日志的代码那倒也还好，但是如果是一些很消耗性能的操作，那就会造成不必要的资源开销。

## 无参数跳过

比如对于重复执行一个自定义 Composable 函数来说，它的性能开销肯定是比重复执行一条打印日志的语句要大的，那么 Compose 对于 Composable 的跳过机制又是怎样的呢。

```kotlin
@Composable
fun HeavyFun() {
    Log.d("优化测试", "位于 HeavyFun() 内部")
    Text(text = "假设在这里进行了网络请求……")
}
```

首先自定义一个 Composable 用于模拟消耗资源的操作，其中会打印一条 Log 。

```kotlin
var name by mutableStateOf("Aiden")
setContent {
	Column {
		Text(text = name)
		HeavyFun()
		Button(onClick = { name = "Marcus" }) {
			Text(text = "改变名称")
		}
	}
}
```

随后在`Column()`中调用上面那个 Composable ，当程序运行，按下按钮后，由`Text()`引起的重组会波及到`HeavyFun()`，其内部的代码应该会被再次执行。

然而在实际运行时，`HeavyFun()`这个 Composable 中的代码并没有因为重组而被再次执行……这其实是因为 Compose 的插件检测到`HeavyFun()`并没有发生变化，所以在执行到它时跳过了它内部的代码，这就是 Compose 的性能优化措施之一。

## 一般数据类型参数跳过

接下来如果为自定义 Composable 添加参数呢，比如这样：

```kotlin
@Composable
fun HeavyFun(age: Int) {
    Log.d("优化测试", "位于 HeavyFun() 内部")
    Text(text = "年龄：$age")
}
```

并且在调用它时传递一个参数进去，同时在按钮的点击事件当中修改变量`age`的值：

```kotlin
var name by mutableStateOf("Aiden")
var age = 23
setContent {
	Column {
		Text(text = name)
		HeavyFun(age)
		Button(onClick = {
			age = 24
			name = "Marcus"
		}) {
			Text(text = "改变名称")
		}
	}
}
```

再次运行程序，点击按钮触发重组后，Compose 发现传入`HeavyFun()`中的参数有变化，于是`HeavyFun()`中的代码就会被再次执行了。

这里有两点值得一提：

1. 在上面的例子中，往`HeavyFun()`内传入的是一个普通的`Int`类型的对象，所以如果只有这个对象发生变化的话是不会触发重组的。当然如果想要往`HeavyFun()`内传入`MutableState`对象也可以，比如把传入`Text()`的变量`name`也传进`HeavyFun()`中，在这种情况下，当变量`name`发生变化时，`HeavyFun()`和`Text()`都会被标记为失效，也就是说，标记失效这个动作执行了两次，但是重组这个行为只会执行一次，这也是 Compose 的性能优化措施之一；
2. 尽管`HeavyFun()`有一个参数，但是如果`HeavyFun()`内部没有用到这个参数，那么不管在调用`HeavyFun()`时如何传递这个参数，重组时`HeavyFun()`中的代码仍然会被跳过……由此看出来 Compose 真的很聪明。

总的来说，Compose 在重组时是否会重复执行某个 Composable 中内部的代码，取决于这个 Composable 函数是不是有基本数据类型的参数，如果没有，那就会直接跳过，如果有并且参数有变，那就重复执行，否则同样跳过。

## 类对象参数跳过

但是如果 Composable 函数中的参数是非基本数据类型的话，比如说是我们自定义的一个`User`类的话，那这里就要分两种情况了，同时每种情况下又会再分两种情况，也就是说总共有四种情况。

**情况一** ，`User`类是普通类（由`class`声明），属性由`val`修饰：

```kotlin
class User(val name: String)

@Composable
fun Greeting(user: User) {
    Log.d("优化测试", "位于 Greeting() 内部")
    Text(text = "Hello ${user.name}!")
}
```

```kotlin
var str by mutableStateOf("点击按钮前")
var user = User("Aiden")
setContent {
	Column {
		Text(text = str)
		Greeting(user = user)
		Button(onClick = {
			user = User("Aiden")
			str = "点击按钮后"
		}) {
			Text(text = "改变 str 的值")
		}
	}
}
```

在这种情况下，Compose 在重组时对新旧`User`类对象进行比较时所使用的`equals()`是父类的`equals()`，也就是用来比较引用性相等的`equals()`，相当于 Kotlin 中的三等号`===`。又因为我们在按钮的点击监听中创建了一个新的`User`类对象并将其赋值给了`user`变量，导致在重组时，Compose 发现传入`Greeting()`内的`User`类对象变成了另外一个，所以`Greeting()`内的代码就被重复执行了。这也就是运行程序后点击按钮触发重组，`Greeting()`内的日志被打印了的原因。

**情况二** ，`User`类是普通类（由`class`声明），属性由`var`修饰：

```kotlin
//          ↓ 改用 var 修饰属性
class User(var name: String)

// Greeting() 没有变化，省略不写
```

```kotlin
var str by mutableStateOf("点击按钮前")
 ↓ 这里改用 val 对 user 变量进行修饰
val user = User("Aiden")
setContent {
	Column {
		Text(text = str)
		Greeting(user = user)
		// 按钮的点击监听中取消赋值的操作
		Button(onClick = { str = "点击按钮后" }) {
			Text(text = "改变 str 的值")
		}
	}
}
```

在这种情况下，Compose 发现`User`类的属性被`var`修饰，于是 Compose 决定只要发生重组，它就会无脑重复执行`Greeting()`内的代码。另外，在上面的代码中，我使用了`val`来修饰变量`user`，同时取消了按钮的点击监听中的赋值操作，我做这些只是想表达参数的变化已经不能影响 Compose 重复执行代码了，因为你在不改变参数的情况下，代码都会被重复执行，更何况你改了参数。

**情况三** ，`User`类是数据类（由`data class`声明），属性由`val`修饰：

```kotlin
// ↓ 数据类      ↓ val 修饰属性
data class User(val name: String)

// Greeting() 没有变化，省略不写
```

```kotlin
// 主要部分的代码与情况一相同，省略不写
```

在这种情况下，Compose 在重组时对新旧`User`类对象进行比较时所使用的`equals()`是数据类重写过的`equals()`，也就是用来比较结构性相等的`equals()`，相当于 Kotlin 中的双等号`==`。所以尽管我们在按钮的点击监听中创建了一个新的`User`类对象并将其赋值给了`user`变量，但是在重组时，Compose 经过比较发现新旧两个`User`类对象的属性是相等的，所以跳过了`Greeting()`内的代码，`Greeting()`内的日志就没有被打印。

**情况四** ，`User`类是数据类（由`data class`声明），属性由`var`修饰：

```kotlin
//               ↓ 改用 var 修饰属性
data class User(var name: String)

// Greeting() 没有变化，省略不写
```

```kotlin
// 主要部分的代码与情况二相同，省略不写
```

**情况四** 和 **情况二** 类似，尽管判断参数变化的条件由引用性相等变为结构性相等，但是由于类的属性被`var`修饰，所以 Compose 会无脑重复执行`Greeting()`内的代码。

综合上述四种情况能得到一个比较可靠的结论就是，当一个类的属性被`var`修饰时，这个属性就有可能在程序中的任何位置被修改，这样的类是不被信任的。对此，Compose 为了保证准确性，选择了最保险的方法，也就是无脑重复执行 Composable 中的代码。

无脑重复执行的做法最大程度上保证了程序正确，但是资源的消耗也变得不可避免了，为此，Compose 提供了一个用于修饰类或者接口的注解`@Stable`。比如在上面的例子中，我们用`@Stable`注解修饰`User`类，这样一来，无论`User`类是普通类还是数据类，就算它的属性是被`var`修饰的，只要`equals()`返回`true`，Compose 就会跳过`Greeting()`。