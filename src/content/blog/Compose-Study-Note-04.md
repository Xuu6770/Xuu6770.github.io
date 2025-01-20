---
title: Compose 学习笔记 ④ - derivedStateOf
date: 2024-01-10 18:50:00
category: 笔记
---
本节主要讲一下`derivedStateOf()`这个函数是什么，怎么用。

<!-- more -->

简单概括一下`derivedStateOf()`的功能：状态对象 A 的更新被委托给了`derivedStateOf()`，`derivedStateOf()`又会去订阅另一个状态对象 B ，当 B 的状态变化后，`derivedStateOf()`会根据已给出的逻辑去更新 A 。接下来我们看一下官方文档中的例子：

```kotlin
@Composable
fun TodoList(highPriorityKeywords: List<String> = listOf("Review", "Unblock", "Compose")) {

    val todoTasks = remember { mutableStateListOf<String>() }

    // Calculate high priority tasks only when the todoTasks or highPriorityKeywords
    // change, not on every recomposition
    val highPriorityTasks by remember(highPriorityKeywords) {
        derivedStateOf { todoTasks.filter { it.containsWord(highPriorityKeywords) } }
    }

    Box(Modifier.fillMaxSize()) {
        LazyColumn {
            items(highPriorityTasks) { /* ... */ }
            items(todoTasks) { /* ... */ }
        }
        /* Rest of the UI where users can add elements to the list */
    }
}
```

文档地址：[https://developer.android.com/jetpack/compose/side-effects?hl=zh-cn#derivedstateof](https://developer.android.com/jetpack/compose/side-effects?hl=zh-cn#derivedstateof)

首先代码的用意其实也就是根据函数参数中给到的字符串列表`highPriorityKeywords`去对列表`todoTasks`进行过滤，得到一个新列表`highPriorityTasks`，然后在一个纵向布局中先展示`highPriorityTasks`中的内容，后展示`todoTasks`的内容。其中重点在于`highPriorityKeywords`和`todoTasks`这两个列表：

- `highPriorityKeywords`作为参数被传递进了`remember()`中，而`remember()`这个函数，我们前面提到过它大概是起到一个缓存的作用。在这个场景中，如果`remember()`的参数发生变化，Compose 在重组时就会重新执行`remember()`中的代码，然后将执行后得到的新结果进行缓存。
- `todoTasks`在`derivedStateOf()`中被`derivedStateOf()`订阅，当`todoTasks`内部发生变化时，`derivedStateOf()`会重新执行 Lambda 表达式中的代码，通过`filter()`计算出新的值（当然在别的场景中不一定是`filter()`）。

无论是`remember()`被重新执行还是`derivedStateOf()`被重新执行，它们重新执行的部分都是一样的，也就是`todoTasks.filter { it.containsWord(highPriorityKeywords) }`这条语句，并且最终都会去更新变量`highPriorityTasks`的值。

官方所提供的这个例子可以说是`derivedStateOf()`使用场景的“最终形态”，因为官方将`derivedStateOf()`和带参数的`remember()`结合起来使用了。但是为了更好理解，我们需要先从几个简单的场景来渐进到官方的这个“最终形态”。

## 场景一

前面提到`derivedStateOf()`的作用就是根据一个状态的变化计算出另一个状态，接下来就这个定义来编写一个最简单的用例：

```kotlin
setContent {
	var name by remember { mutableStateOf("Aiden") }
	val upperCaseName by remember {
		derivedStateOf { name.uppercase() }
	}
	Column {
		Text(text = upperCaseName)
		Button(onClick = { name = "Marcus" }) {
			Text(text = "修改名字")
		}
	}
}
```

程序的功能很简单，用`derivedStateOf()`订阅一个字符串类型的 state object 的变化，将这个字符串转换成大写然后赋值给变量`upperCaseName`，最后再把`upperCaseName`显示出来。此外还有几个要点：

1. 变量`upperCaseName`被使用`by`关键字委托给了`derivedStateOf()`，尽管`derivedStateOf()`函数本身返回的是一个`State`对象，但是因为委托，变量`upperCaseName`在这里只会是`String`类型；
2. 变量`upperCaseName`被`val`修饰，这是因为`derivedStateOf()`会帮我们去更新`upperCaseName`的值，我们并不需要也并不能去手动更新`upperCaseName`，如果手动更新，那就违背了使用`derivedStateOf()`的目的；
3. `derivedStateOf()`返回的对象和`mutableStateOf()`返回的对象都是会被订阅状态变化的；

程序运行，`derivedStateOf()`和`remember()`都会执行一次，前者是为了进行第一次的委托计算并赋值（也就是将变量`name`的字符串转换为大写），而后者是为了缓存这个结果。于是界面上会显示一行大写的文本“AIDEN”以及在它下方的一个按钮，当我点击按钮改变`name`的值后，由于`derivedStateOf()`内部使用（订阅）到了`name`，所以`derivedStateOf()`中的代码会再次执行，从而计算出新的值，也就是“MARCUS”，并将其赋值给`upperCaseName`，最后`Text()`这边也会进行重组，界面上最终显示着一行大写的文本“MARCUS”以及在它下方的一个按钮。

这个例子看起来还挺简单的，而且这个“根据变化重复执行代码”的特性好像在之前也遇到过……好像用带参数的`remember()`也可以实现上面代码相同的效果来着……我们来看一下：

```kotlin
setContent {
	var name by remember { mutableStateOf("Aiden") }
	//                     ↓ 使用带参数的 remember() 来更新
	val upperCaseName = remember(name) { name.uppercase() }
	Column {
		Text(text = upperCaseName)
		Button(onClick = { name = "Marcus" }) {
			Text(text = "修改名字")
		}
	}
}
```

程序运行，由于点击按钮改变了`name`的值，这就导致在重组时 Compose 发现传入`remember()`中的参数有变化，于是就刷新了缓存（也就是重复执行了`remember()`中的代码），`upperCaseName`的值得到更新，程序最终的运行效果和上面使用`derivedStateOf()`时是一样的……带参数的`remember()`和`derivedStateOf()`运行的效果竟然是一样的……这就不禁让人开始思考这两者的区别在哪里……

## 场景二

在上面的场景中可以看到带参数的`remember()`和`derivedStateOf()`都可以根据变化在重组时重新执行代码，这是它们相同的地方，而不同的地方就在于它们判断变化的依据：

- `remember()`在重组时是否会重新执行代码，是依据于它的参数本身有没有发生变化，比如从一个`String`变成另一个`String`，这种对象的变化往往伴随着赋值操作。也就是说，当参数是集合类型时，就属于特殊情况了，因为集合内部发生的变化（例如增加元素或者移除元素）是不会引起`remember()`的注意的，从而也就不会刷新缓存；
- `derivedStateOf()`是否会重新执行代码，是依据于它所订阅的 state object 的变化，比如说当订阅的是`mutableStateOf()`创建的 state object 时，一旦这个 state object 发生了赋值操作，那就会引起`derivedStateOf()`内部代码的再执行。包括`mutableStateListOf()`创建的集合对象也是一样，当集合内发生变动时，就会触发`derivedStateOf()`内部代码的再执行；

接下来根据上面的结论再来看个例子：

```kotlin
setContent {
	val names = remember { mutableStateListOf("Aiden", "Marcus") }
	val upperCaseNames by remember {
		derivedStateOf { names.map { it.uppercase() } }
	}
	Column {
		for (name in upperCaseNames) {
			Text(text = name)
		}
		Button(onClick = { names.add("Wrench") }) {
			Text(text = "添加名字")
		}
	}
}
```

这一次使用了`mutableStateListOf()`创建了一个可被订阅内部状态变化的列表`names`，接着同样使用`derivedStateOf()`对`names`进行订阅，当`names`的内部发生变化时，`derivedStateOf()`内的代码就会被再次执行，从而计算出新的值并赋给变量`upperCaseNames`。

程序的功能就是把列表中的名字全部变成大写然后展示出来。程序跑起来以后，会先显示两行大写的名字，分别是`AIDEN`和`MARCUS`，当我点击按钮后，列表中就会添加一个新的名字，列表这个变化会引起`derivedStateOf()`内代码的再次执行，原本只显示两行大写名字的界面在点击按钮后显示了第三行大写名字，运行效果很理想。

接着再来看看如果将`names`作为参数传递进`remember()`以后会怎么样：

```kotlin
setContent {
	val names = remember { mutableStateListOf("Aiden", "Marcus") }
	//                     ↓ 使用带参数的 remember() 来更新
	val upperCaseNames = remember(names) {
		names.map { it.uppercase() }
	}
	Column {
		for (name in upperCaseNames) {
			Text(text = name)
		}
		Button(onClick = { names.add("Wrench") }) {
			Text(text = "添加名字")
		}
	}
}
```

程序运行，点击按钮，界面是没反应的。原因其实就是尽管列表`names`中添加了新的元素，但是`remember()`并不关心这样的变化，除非`names`被用等号`=`赋值成了另一个列表。

到此为止，我们似乎就对什么时候该使用`derivedStateOf()`和什么时候该使用带参数的`remember()`有一个初步的概念了，那就是“当涉及到集合内部变化时，我们使用`derivedStateOf()`，当涉及到赋值操作时，我们就使用带参数的`remember()`”，对吧？对，但是还不够。

## 场景三

把话挑明了说，`remember()`和`derivedStateOf()`的本质从头到尾都没变过，`remember()`的作用其实就是我们前面说到的“缓存”，它的参数只是作为一个“是否要刷新缓存”的依据；而`derivedStateOf()`的作用就是监听其订阅对象的状态变化，当变化发生时，重新执行代码，执行的结果是根据其订阅对象的变化而变化的。`remember()`加`derivedStateOf()`这个组合其实与`remember()`加`mutableStateOf()`这个组合类似，`remember()`和它们都在各司其职。

最后来看一个类似于官方例子的例子：

```kotlin
@Composable
fun UpperCaseNames(names: List<String>, onAddElement: () -> Unit) {
    //                             ↓ 我该添加参数吗？
    val upperCaseNames by remember( ) {
        // ↓ 我该使用 derivedStateOf 吗？
        derivedStateOf { names.map { it.uppercase() } }
    }
    Column {
        for (name in upperCaseNames) {
            Text(text = name)
        }
        Button(onClick = { onAddElement.invoke() }) {
            Text(text = "增加元素")
        }
    }
}
```

首先可以看到上面的例子是一个自定义的 Composable 函数，这意味着这个例子具有一定的通用性，包括官方给的例子也是一个自定义的 Composable ，因为在实际开发中我们不可能像前面的例子那样把代码都写在`setContent()`里。

接着再来看看这个函数的功能，我的想法很简单，就是在调用这个 Composable 的时候传入一个储存着名字的列表，函数把列表里的名字全部变成大写再展示出来，并且当我点击按钮以后，我希望往列表里添加一个名字，最后函数要把更新后的列表里的名字全部变成大写再展示出来。那么现在问题就来了，对于这个需求，我是应该使用带参数的`remember()`还是应该使用`derivedStateOf()`？还是说两个都用？

首先我在点击按钮往列表里增加新的名字的时候，我需要这个变化能被感知到，也就是要能触发重组刷新界面，所以这个列表应该是一个由`mutableStateListOf()`创建的可被订阅内部变化的列表才行。接着我把这个列表作为参数传递进`UpperCaseNames()`中：

```kotlin
setContent {
	//                            ↓ 可被订阅内部变化的列表
	val names = remember { mutableStateListOf("Aiden", "Marcus") }
	//                       ↓ 传递参数
	UpperCaseNames(names = names) {
		names.add("Wrench")
	}
}
```

既然我传递的是一个可以被订阅的列表，那我直接用`derivedStateOf()`订阅它不就好了吗？我试试只用`derivedStateOf()`而不用带参数的`remember()`。

```kotlin
@Composable
fun UpperCaseNames(names: List<String>, onAddElement: () -> Unit) {
    val upperCaseNames by remember {
		//                 ↓ 订阅列表的内部变化
        derivedStateOf { names.map { it.uppercase() } }
    }
    Column {
        for (name in upperCaseNames) {
            Text(text = name)
        }
        Button(onClick = { onAddElement.invoke() }) {
            Text(text = "增加元素")
        }
    }
}
```

运行程序，当我点击按钮后，列表`names`增加了一个新名字，`names`的内部变化让`derivedStateOf()`重新执行了 Lambda 中的代码，从而计算出了新的列表，最后再刷新界面把新的名字的大写形态展示出来，整体效果理想。这样看来，我似乎真的就不需要带参数的`remember()`了……

那倒还真不是，我们再来看一个场景：我现在有两个名字列表和一个代表年龄的变量，我现在不想往列表里增加新的名字了，我也不 care 名字是不是大写了，我现在的新需求就是要根据年龄显示不同的名字列表，当年龄大于或等于 18 岁时，我显示一个列表，否则我显示另一个。

```kotlin
setContent {
	Column {
		var age by remember { mutableIntStateOf(17) }
		val names = remember(age) {
			if (age >= 18) mutableStateListOf("Aiden", "Marcus")
			else mutableStateListOf("想不出别的英文呢名了", "随便写两个字符串")
		}
		UpperCaseNames(names = names) { names.add("Wrench") }
		Button(onClick = { age = 18 }) {
			Text(text = "成年了")
		}
	}
}
```

`UpperCaseNames()`函数的代码暂时不动。程序运行，因为`age`初始化为`17`，所以先展示`"想不出别的英文呢名了", "随便写两个字符串"`这个列表，当我点击按钮后，`age`被改为`18`，触发重组，又因为`age`是`remember()`的参数，所以`remember()`中的代码也会重新执行，所以不用担心`names`不更新。程序继续往下跑，`names`被更新为`"Aiden", "Marcus"`这个列表以后被作为参数传递进`UpperCaseNames()`函数中，照理说`UpperCaseNames()`接下来应该会根据新的参数来显示新的列表，但实际上界面并没有变化。

我们回看`UpperCaseNames()`函数其实也很容易发现问题，问题就在于`remember()`没参数。尽管重组带着新的名字列表进入了`UpperCaseNames()`函数内部，但是因为`remember()`没参数导致缓存没刷新，`upperCaseNames`这个变量所持有的列表仍然是之前缓存过的`"想不出别的英文呢名了", "随便写两个字符串"`这个列表，这也就是界面显示没变化的原因了。

解决方案也很简单，为`remember()`添加参数即可，并且这个参数就是`UpperCaseNames()`函数的参数：

```kotlin
@Composable
fun UpperCaseNames(names: List<String>, onAddElement: () -> Unit) {
    //                               ↓ 添加参数
    val upperCaseNames by remember(names) {
        derivedStateOf { names.map { it.uppercase() } }
    }
    Column {
        for (name in upperCaseNames) {
            Text(text = name)
        }
        Button(onClick = { onAddElement.invoke() }) {
            Text(text = "增加元素")
        }
    }
}
```

这样一来，当列表内部元素发生变动的时候，就由`derivedStateOf()`来负责重新执行代码，而当列表本身变成另一个对象的时候，就由`remember()`来负责重新执行代码。这也符合了我前面说到的两者各司其职并不冲突。

## 总结

说了那么多只是为了搞清楚`remember()`和`derivedStateOf()`的本质而已，前者首先是起到一个缓存作用，其次是根据参数是否变化决定是否要刷新缓存；后者则是用于监听状态变化从而重新执行代码的。剩下的就是要结合实际场景来决定使用哪一个以及是否需要搭配使用了。