---
title: Android 学习笔记 - Activity
date: 2021-12-29 18:37:10
category: 笔记
---
主要记录一下 Activity 的生命周期、启动模式，以及 Intent 。

<!-- more -->

## 生命周期

Activity 存储在栈这种先进后出的结构中，位于栈顶的 Activity 将会展示给用户，执行「返回」操作的时候栈顶的 Activity 也会先出栈。

Activity 在生命周期中会存在 4 种状态：

- 运行：当打开一个 Activity 的时候，它就会位于栈顶，并且处于运行状态。
- 暂停：不是每一个 Activity 都会占满整个屏幕，例如对话框。而当我们将一个对话框形式的 Activity 入栈时，位于其之下的 Activity 也就是原来位于栈顶的 Activity 就会进入暂停状态。因为这个 Activity 被对话框挡住了，但又没完全不可见，尤其是对话框还导致这个 Activity 变得不可交互，所以就进入了暂停状态。
- 停止：当一个 Activity 不再处于栈顶位置时，且完全不可视时，将会进入此状态。处于该状态的 Activity 有可能会被系统回收。
- 销毁：当 Activity 出栈后就会变成这个状态。

这 4 种状态伴随着 7 个回调方法：

1. `onCreate()`：在 Activity 在第一次创建的时候调用，在里面执行一些初始化操作。
2. `onStart()`：在 Activity 由不可见变为可见时调用。
3. `onResume()`：在 Activity 准备好和用户进行交互的时候调用。此时的 Activity 一定位于栈顶。
4. `onPause()`：在系统准备去启动或者恢复另一个 Activity 的时候调用。通常会在这个方法中将一些消耗 CPU 的资源释放掉，以及保存一些关键数据。这个方法的执行速度一定要快，不然会影响到新的栈顶 Activity 的使用。
5. `onStop()`：在 Activity 完全不可见的时候调用。和`onPause()`的主要区别在于，如果启动的新 Activity 是一个对话框形式的 Activity ，那么`onPause()`会得到执行。
6. `onDestroy()`：在 Activity 被销毁之前调用。
7. `onRestart()`：这个方法在 Activity 由停止状态变为运行状态之前调用。

以上 7 个方法中除了`onRestart()`，其它都是两两对应的，可以这样简单理解：Activity 的创建和销毁对应 1 和 6 ，可见和完全不可见对应 2 和 5 并且 7 穿插在 5 到 2 之间，4 是部分可见，3 是从部分可见恢复到完全可见。

![Android 生命周期](/images/note/Activity_Life_Cycle.jpg)

## Intent

Intent 是 Android 程序中各组件之间进行交互的一种重要方式，它不仅可以指明当前组件想要执行的动作，还可以在不同组件之间传递数据。Intent一般可用于启动 Activity 、启动 Service 以及发送广播等场景。

### 显式 Intent

最简单的用于启动 Activity 的方式，只需要提供两个参数，一是启动 Activity 的上下文，二是指定要启动的目标 Activity 。假设当前项目中有一个 Activity 的名称为「SecondActivity」。

```kotlin
val intent = Intent(this, SecondActivity::class.java)  // 构建一个 Intent 对象
startActivity(intent)  // 通过 Intent 对象启动 Activity
```

Kotlin 中的`SecondActivity::class.java`相当于 Java 中的`SecondActivity.class`。

### 隐式 Intent

隐式 Intent 并不明确指出想要启动哪一个 Activity ，而是预先在`AndroidManifest.xml`的`<intent-filter>`中设置`<action>`、`<catagories>`，来让同时满足这两个条件的 Activity 响应。

如果有多个应用可处理 Intent ，系统会为用户显示一个对话框，供其选择要使用的应用。

#### 启动自己程序的 Activity

假设在程序中有一个叫「SecondActivity」的 Activity ，那么此时需要在`AndroidManifest.xml`中对名为「.SecondActivity」的 Activity 标签进行设置，具体操作就是为其增加`<intent-filter>`标签，并且在其中按需指定 action 和 category ：

```xml
<activity
    android:name=".SecondActivity"
    android:exported="false">
    <intent-filter>
        <action android:name="Start Second Activity" /> <!-- 指定 action -->
        <category android:name="android.intent.category.DEFAULT" /> <!-- 指定 category -->
    </intent-filter>
    <meta-data
        android:name="android.app.lib_name"
        android:value="" />
</activity>
```

接下来需要在调用处新建 Intent 对象并指定 action 和 category 信息，然后如果存在某个 Activity ，它在`AndroidManifest.xml`中注册了 Intent 对象中一摸一样的 action 和 category （比如上面提到的「SecondActivity」），那么该 Activity 将会响应`startActivity()`：

```kotlin
val intent = Intent("Start Second Activity")  // 指定 action
startActivity(intent)
```

不同于显式 Intent 使用的构造函数，此处构建 Intent 对象传入的参数是一个 action 。并且此处没有明确指定 category ，是因为默认就会使用`android.intent.category.DEFAULT`这个 category 。除了默认的 category ，也可以通过`intent.addCategory()`手动为 Intent 对象添加多个 category ，但是 action 只能有一个。当`startActivity(intent)`执行后，将成功启动「SecondActivity」。

#### 启动其它程序的 Activity

根据需求在构建 Intent 对象时指定相应的 action ，就可以启动其它程序的 Activity。

- 网络浏览器加载网址

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("http://www.baidu.com"))
startActivity(intent)
```

- 发起通话

```kotlin
val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:10086"))
startActivity(intent)
```

## Activity 间传递数据

Intent 提供了一系列`putExtra()`方法的重载，可以把想要传递的数据暂存在 Intent 中，在启动另一个 Activity 后，再把这些数据从 Intent 中取出就可以了。

### 向下一个 Activity 传递

存储消息并启动 Activity ，其中`putExtra()`需要两个参数，可以视为键值对，第一个是键，第二个才是值。

```kotlin
/* MainActivity */

val data = "Greetings to SecondActivity!"
val intent = Intent(this, SecondActivity::class.java).apply {
    putExtra("dataFromMainActivity", data)
}
startActivity(intent)
```

接下来取出消息并打印，取消息时需要根据数据类型来使用不同的 getXxxExtra 方法。

```kotlin
/* SecondActivity */

val data = intent.getStringExtra("dataFromMainActivity")
Log.d("SecondActivity", "传递过来的消息是：$data")
```

在「SecondActivity」中并没有定义名为`intent`的 Intent 对象，`intent.getStringExtra("dataFromMainActivity")`中的`intent`实际上是调用了父类的`getIntent()`，该方法会获取用于启动「SecondActivity」的 Intent ，然后再通过 getXxxExtra 方法并传入键（dataFromMainActivity），就可以取出相应的值。

除了上方代码中的`getStringExtra()`以及其它用于接收基本数据类型的方法之外，还有`getSerializableExtra()`、`getParcelableExtra()`、`getBundleExtra()`、`getCharSequenceExtra()`这四个方法用于接收相对应的对象。

### 返回给上一个 Activity

在`androidx.appcompat:appcompat:1.3.0`以及更高的版本当中，`startActivityForResult()`已被废弃，同时被废弃的还有`requestPermissions()`。取而代之的是 Activity Result API ，通过这个 API 来实现两个 Activity 间的数据交换以及请求运行时权限。

```kotlin
/* MainActivity */

private val requestDataLauncher =
    // 调用 registerForActivityResult() 来注册一个对 Activity 结果的监听
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
        // 若有结果发返回则会回调此处的 Lambda 表达式对结果进行处理
        if (it.resultCode == RESULT_OK) {
            val data = it.data?.getStringExtra("dataFromSecondActivity")
            Log.d("dataFromSecondActivity", "$data")
        }
    }

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val intent = Intent(this, SecondActivity::class.java)
    setContent {
        // 使用 ActivityResultLauncher 对象的 launch() 来启动 Intent
        Button(onClick = { requestDataLauncher.launch(intent) }) {
            Text(text = "启动 SecondActivity")
        }
    }
}
```

```kotlin
/* SecondActivity */

Button(onClick = {
    val intent = Intent()
    intent.putExtra("dataFromSecondActivity", "Greetings to MainActivity!")
    setResult(RESULT_OK, intent)
    finish()  // 销毁当前 Activity
}) {
    Text(text = "结束 SecondActivity 并返回数据")
}
```

在 MainActivity 中已经完全移除了对`onActivityResult()`的重写，而是调用`registerForActivityResult()`方法来注册一个对 Activity 结果的监听。`registerForActivityResult()`方法接收两个参数，第一个参数是一种 Contract 类型，由于是从另外一个 Activity 中请求数据，因此这里使用了 StartActivityForResult 这种 Contract 。第二个参数是一个 Lambda 表达式，当有结果返回时则会回调到这里，然后在这里获取并处理数据即可。

`registerForActivityResult()`方法的返回值是一个`ActivityResultLauncher`对象，这个对象当中有一个`launch()`可以用于去启用 Intent。这样就不需要再调用`startActivityForResult()`了，而是直接调用`launch()`并把 Intent 传入即可。

## 启动模式

某个 Activity 的启动模式是在清单文件中其对应的`<activity>`标签内进行设置的，例如此处将 MainActivity 的启动模式设置为`singleTask`：

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="singleTask"><!-- 将启动模式设置为 singleTask -->
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

启动模式可分为两大类，`standard`和`singleTop`是一类，`singleTask`、`singleInstance`和`singleInstancePerTask`是另一类。如此分类的依据是前者可以对 Activity 进行多次实例化。

### standard

`standard`是 Activity 默认的启动模式，对于使用该模式的 Activity ，系统每次都会在当前任务栈顶创建一个新实例而不会理会栈中是否已经存在实例。每个实例可以属于不同的任务，而一个任务可以有多个实例。

### singleTop

在该模式下，如果 Activity 已经处于当前任务栈栈顶，那么系统将不会创建新的实例，而是复用已有的实例，并调用其`onNewIntent()`。

`onNewIntent()`可以在 Activity 中被重写，其在`onPause()`之后，`onResume()`之前被调用，并且接收一个`Intent`类型的参数，这个 Intent 就是在调用`startActivity()`时传递的 Intent 。

### singleTask

如果要启动的 Activity 已经存在于某个任务栈中，那么系统会把该任务栈调到前台，并将其之上的所有 Activity 通通出栈，然后再调用其`onNewIntent()`。如果要启动的 Activity 不存在于任何任务栈中，那么系统会创建一个新的任务栈并在其根位置创建实例。

接下来是一个简单的测试用例，假设有两个 Activity 分别为 MainActivity 和 SecondActivity ，且 MainActivity 的`launchMode`设置为`singleTask`。

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Button(onClick = {startActivity(Intent(this, SecondActivity::class.java))}) {
                Text(text = "启动 SecondActivity")
            }
        }
    }

    override fun onPause() {
        super.onPause()
        Log.i("@@@", "MainActivity onPause() 被调用")
    }

    override fun onResume() {
        super.onResume()
        Log.i("@@@", "MainActivity onResume() 被调用")
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        Log.i("@@@", "MainActivity onNewIntent() 被调用")
    }
}
```

```kotlin
class SecondActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Button(onClick = { startActivity(Intent(this, MainActivity::class.java)) }) {
                Text(text = "启动 MainActivity")
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.i("@@@", "SecondActivity onDestroy() 被调用")
    }
}
```

程序运行，在 MainActivity 中点击按钮启动 SecondActivity 会得到如下打印：

> MainActivity onPause() 被调用

随后在 SecondActivity 中点击按钮启动 MainActivity 会得到如下打印：

> MainActivity onNewIntent() 被调用
MainActivity onResume() 被调用
SecondActivity onDestroy() 被调用

### singleInstance

在该模式下，只允许一个 Activity 的实例存在，并且该实例会运行在一个单独的任务栈中。如果要启动的 Activity 已经存在于该任务栈中，那么系统会直接调用其 onNewIntent() 方法。

接下来是一个简单的测试用例，假设程序中有三个 Activity ：

- MainActivity 用来启动 SecondActivity 。
- SecondActivity 的启动模式设置为`singleInstance`且用来启动 ThirdActivity 。
- 三个 Activity 的`onCreate()`中都会打印`taskId`。

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val context = this
        Log.i("@@@", "MainActivity 启动，任务 ID 为：$taskId")
        setContent {
            Column(modifier = Modifier.fillMaxSize()) {
                Text(text = "目前位于 MainActivity")
                Button(onClick = { startActivity(Intent(context, SecondActivity::class.java)) }) {
                    Text(text = "启动 SecondActivity")
                }
            }
        }
    }
}
```

```kotlin
class SecondActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val context = this
        Log.i("@@@", "SecondActivity 启动，任务 ID 为：$taskId")
        setContent {
            Column(modifier = Modifier.fillMaxSize()) {
                Text(text = "目前位于 SecondActivity")
                Button(onClick = { startActivity(Intent(context, ThirdActivity::class.java)) }) {
                    Text(text = "启动 ThirdActivity")
                }
            }
        }
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        Log.i("@@@", "SecondActivity onNewIntent() 被调用")
    }
}
```

```kotlin
class ThirdActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val context = this
        Log.i("@@@", "ThirdActivity 启动，任务 ID 为：$taskId")
        setContent {
            Column(modifier = Modifier.fillMaxSize()) {
                Text(text = "目前位于 ThirdActivity")
                Button(onClick = { startActivity(Intent(context, SecondActivity::class.java)) }) {
                    Text(text = "启动 ThirdActivity")
                }
            }
        }
    }
}
```

程序运行，会得到第一个打印：

> MainActivity 启动，任务 ID 为：55

接着点击按钮启动 SecondActivity 会得到第二个打印：

> SecondActivity 启动，任务 ID 为：56

接着点击按钮启动 ThirdActivity 会得到第三个打印：

> ThirdActivity 启动，任务 ID 为：55

可以看到，MainActivity 和 ThirdActivity 的`taskId`是一致的，说明它们位于同一个栈内，而 SecondActivity 由于将启动模式设置成了`singleInstance`，使其单独被放置在一个栈中进行管理。这时候在设备上进行「返回」操作，会发现 ThirdActivity 会直接返回到 MainActivity ，然后再返回到 SecondActivity ，最后才退出程序。

而当在 ThirdActivity 中点击按钮启动 SecondActivity 时，由于其已经实例化过了，所以会直接被复用并且调用其`onNewIntent()`。

### singleInstancePerTask

真的是永远有学不完的新东西……`singleInstancePerTask`是 Android 12 新增的第五种启动模式，它的作用是让 Activity 只能作为任务栈的根 Activity 存在，并且每个任务栈内只能有一个实例。但是在不同的任务栈中可以有不同的实例，如果想要这么做，就需要在 Intent 内设置`Intent.FLAG_ACTIVITY_MULTIPLE_TASK`或`Intent.FLAG_ACTIVITY_NEW_DOCUMENT`标志。而`singleTask`是在整个应用内只有一个实例，但是它可以是也可以不是一个任务栈的根，这将由任务的亲和性决定，也就是清单文件中的`<activity>`标签内`taskAffinity`属性。

## 参考资料

- [Activity Result API详解，是时候放弃startActivityForResult了](https://mp.weixin.qq.com/s/C62WVau-AU0MH7S4Ix85ew)
- [通用 Intent | Android 开发者 | Android Developers](https://developer.android.com/guide/components/intents-common)
- 《第一行代码 Android 第3版》郭霖 著
