---
title: Android 学习笔记 - 广播&通知
date: 2022-01-06 13:23:39
category: 笔记
---
Android apps 可以发送广播，亦可接受来自系统或者其他 apps 的广播并针对其做一些工作。广播还可以作为一个跨应用程序和正常用户流之外的信息传递系统使用；通知是指在应用界面之外显示的消息，其旨在向用户提供提醒、来自他人的通信信息或应用中的其他实时信息。用户可以点按通知来打开应用，或直接从通知中执行操作。

<!-- more -->

## 广播

### 接收广播

程序可以通过两种方式来接受广播：

- 注册上下文接收器（context-registered receivers）（动态注册）
- 基于 manifest 的接收器（manifest-declared receivers）（静态注册）

其中动态注册一般是需要在程序启动之后才能接收广播，因为上下文接收器的「上下文」必须要有效，例如在 Activity 上下文中注册，只要 Activity 没有被销毁，就可以收到广播。如果在应用上下文中注册，那么只要应用还在运行，就可以收到广播；而静态注册可以在广播发出后由系统来启动程序。不过无论哪种方式，都需要通过 BroadcastReceiver 来实现。进一步说就是创建一个类，使其继承于`BroadcastReceiver`类，然后重写`onReceive()`，这样当程序接收到对应的广播时，`onReceive()`中的代码就会执行。

#### 静态注册

因为静态注册可以在程序未启动的情况下接收广播，从 Android 8.0 (API level 26) 开始，静态注册将不再能接收大部分的隐式广播，所谓隐式广播就是没有特别指定由哪一个程序接收的广播。

静态注册的大致步骤：

- 创建一个类，使其继承于`BroadcastReceiver`类，然后重写`onReceive()`。
- 在`AndroidManifest.xml`对这个 receiver 进行注册（创建`<receiver>`标签，标签的`android:name`属性为第一步中创建的类的类名）。

如果是通过 Android Studio 直接创建一个 BroadcastReceiver 则可以省去以上两步，因为 Android Studio 已经自动完成了，此时在`AndroidManifest.xml`中可以看到在`<application>`标签中，也就是和`<activity>`标签同级的位置多出来一个`<receiver>`（假设类名为「BatteryChangeReceiver」）：

```xml
<receiver
android:name=".BatteryChangeReceiver"
android:enabled="true"
android:exported="true"></receiver>
```

- 接下来在`<receiver>`标签中的`<intent-filter>`声明需要接收的广播，例如通过`<action android:name="android.intent.action.BOOT_COMPLETED" />`便可以收到手机开机完成后发送的广播：

```xml
<receiver
    android:name=".BatteryChangeReceiver"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

不过如果是要接受开机完成的广播，则还需要声明权限：`<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />`。

- 最后在`onReceive()`中编写非耗时操作即可。

#### 动态注册

此处以接收电池电量的变化为例记录一下动态注册的主要步骤及代码：

```kotlin
class MainActivity : ComponentActivity() {
    // 3. 创建一个第一步中定义的类的实例对象
    private val batteryChangeReceiver = BatteryChangeReceiver()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 2. 创建一个`IntentFilter`的实例对象并根据广播类型为其添加相应的 action
        val intentFilter =
            IntentFilter().apply { addAction("android.intent.action.BATTERY_CHANGED") }
        // 4. 调用 registerReceiver() 并将第三步和第二步中的对象作为参数传入
        registerReceiver(batteryChangeReceiver, intentFilter)
    }

    override fun onDestroy() {
        super.onDestroy()
        // 5. 在适当的位置调用`unregisterReceiver()`以取消注册释放资源
        unregisterReceiver(batteryChangeReceiver)
    }

    // 1. 定义一个类继承自`BroadcastReceiver`并重写`onReceive()`
    class BatteryChangeReceiver : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            // 简单地显示一个吐司
            Toast.makeText(context, "当前电量：%${intent?.getIntExtra("level", 0)}", Toast.LENGTH_SHORT)
                .show()
        }
    }
}
```

此时当程序接收到来自系统发出的电量变化的广播时将会弹出一个吐司。此外，还可以在`/<计算机的 Android SDK 目录>/platforms/<相应的Android API 版本>/data/broadcast_actions.txt`中找到完整的广播列表。

### 发送广播

#### 普通广播

通过使用`sendBroadcast(Intent)`来发送普通广播，普通广播是一种异步执行的广播，所有接收器几乎会在同一时间收到该广播信息。

```kotlin
button.setOnClickListener {
    Intent().also { intent ->
        intent.action = "com.risingsun.learnandroid.MY_BROADCAST"  // 设置 intent
        intent.putExtra("info", "This is my broadcast")  // 存储额外信息
        intent.setPackage(packageName)  // 指定接收广播的应用
        sendBroadcast(intent)  // 发送广播
    }
}
```

此处的`com.risingsun.learnandroid.MY_BROADCAST`属于自定义广播，自定义广播往往是隐式广播，前面提到，Android 8.0 开始将不再能通过静态注册接收隐式广播（哪怕是自己给自己广播），所以这里还需要通过`setPackage(packageName)`来指定接收广播的应用，其中`packageName`属于语法糖，`setPackage(packageName)`的意思其实就是广播给应用自己。

然后在程序中添加一个静态注册的 BroadcastReceiver 并重写`onReceive()`来接收这个广播：

```kotlin
override fun onReceive(context: Context, intent: Intent) {
    val info = intent.getStringExtra("info")
    Toast.makeText(context, info, Toast.LENGTH_LONG).show()
}
```

运行程序点击按钮将会看到一个内容为「This is my broadcast」的吐司。

#### 有序广播

通过使用`sendOrderedBroadcast(Intent, String)`来发送有序广播，有序广播是一种同步执行的广播，一次只向一个接收器发送广播，接收器在处理完之后可以将结果传播给下一个接收器，也可以完全中止广播。静态注册的接收器可以通过在`<intent-filter>`内定义`android:priority`的值来决定优先级，动态注册的接收器直接设置`IntentFilter`对象的`priority`即可。当存在多个相同优先级的接收器时，将随机执行。

在发送普通广播的代码基础上，只需要将`sendBroadcast(intent)`更改为`sendOrderedBroadcast(intent, null)`即可发送有序广播，其中第二个参数对应权限相关的字符串，若将`null`改为`android.permission.INTERNET`，那么只有在`AndroidManifest`中声明`<uses-permission android:name="android.permission.INTERNET" />`的程序的接收器才能收到广播。

#### 向下一个接收器传递结果

在当前接收器收到广播以后可以在`onReceive()`中使用 setResultXxx() 来设定结果信息，当下个接收器收到广播时可以从中取出信息，同时也可以为再下一个接收器继续设置结果信息。

例如在当前接收器中的`onReceive()`中写上`resultData = "result from MyBroadcastReceiver"`，那么下个接收器可以直接使用`resultData`这个变量，其值就是`result from MyBroadcastReceiver`。

#### 阻断广播

在`onReceive()`中使用`abortBroadcast()`可以终止广播的发送，即下一个接收器无法收到广播。

### 通过权限限制广播

在调用`sendBroadcast()`或`sendOrderedBroadcast()`时可以向其中传入权限参数，此时只有在`AndroidManifest.xml`申请了相关权限的 apps 的接收器可以收到这个广播。

相反，如果在注册接收器时填入指定权限，那么就只有在`AndroidManifest.xml`申请了相关权限的 apps 可以向这些接收器所属的 app 发送广播，例如在静态注册时：

```xml
<receiver android:name=".MyBroadcastReceiver"
            android:permission="android.permission.SEND_SMS">
    <intent-filter>
        <action android:name="android.intent.action.AIRPLANE_MODE"/>
    </intent-filter>
</receiver>
```

或者在动态注册时：

```kotlin
var filter = IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED)
registerReceiver(receiver, filter, Manifest.permission.SEND_SMS, null )
```

那么发送方必须申请`<uses-permission android:name="android.permission.SEND_SMS"/>`权限才能向这些接收器发送广播。

## 通知

### NotificationChannel

「NotificationChannel」意为「通知渠道」，简单理解就是站在应用的角度来对通知进行分类，例如 Android 端的 YouTube 会创建很多个通知类别，其中包括「订阅内容」、「直播」、「评论和回复」等等。从 Android 8.0 API 26 开始，必须为所有通知分配渠道，否则通知将不会显示。将通知进行分类可以让用户知道每个通知的意图，也可以让用户对每个类别的通知的提醒方式以及是否允许提醒进行自定义，不至于一个开关直接关掉所有的通知。然而在 Android 7.1 API 25 及更低版本的设备上，用户就真的只能一刀切（只能单独对每个应用是否允许通知进行设置）。

分配渠道之前要先创建一个`NotificationChannel`对象，然后在创建`Notification`对象也就是构建通知本体时才将渠道传递过去。创建通知渠道一般有这么几步：

```kotlin
// 1
val mannager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager

// 2
val channel = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    NotificationChannel("testChannel", "通知测试", NotificationManager.IMPORTANCE_DEFAULT)
} else {
    TODO("VERSION.SDK_INT < O")
}

// 3
mannager.createNotificationChannel(channel)
```

1. 通过`getSystemService()`创建一个`NotificationManager`对象。
2. 创建一个`NotificationChannel`对象，构造函数接收 3 个参数，分别是：
    1. 渠道的 ID ，字符串类型。此处设置的 ID 会在创建`Notification`对象的时候用到，设置了这个 ID 的通知就会归属到这个渠道（类别）下。
    2. 渠道的名称，字符串类型。名称会在系统设置的应用详情中的通知管理里显示，例如此处将会显示「通知测试」，然后后面会跟上一个允许通知的开关。
    3. 渠道的重要性，整型。该值会影响通知出现时形式，包括是否在界面上弹出以及是否发出声音等。需要注意，无论在此处将其设置为什么级别，用户都可以在安装应用后手动修改，并且重要性一旦设置好以后，就算后期在代码上进行修改，改动也无法生效。例如一开始设置为`NotificationManager.IMPORTANCE_DEFAULT`，后期修改为`NotificationManager.IMPORTANCE_HIGH`（重要性为`NotificationManager.IMPORTANCE_HIGH`的通知将会以提醒式通知的形式出现，也就是像微信收到联系人消息一样，在手机顶部弹出，这是 5.0 引入的特性），实际运行的时候，通知的出现形式仍然和原来一样，这种情况下就算清除应用缓存和数据也没用，就算调用`NotificationManager`对象的`deleteNotificationChannel()`将通知渠道删除然后再重新创建也没用，解决方案只有卸载重装应用或者新建一个通知渠道。尽管重要性无法再被修改，但是渠道的名称和说明仍然可以修改。
3. 最后通过调用`NotificationManager`对象的`createNotificationChannel()`方法并将`NotificationChannel`对象传入即可创建通知渠道。

创建好的通知渠道会保留在系统设置的应用详情中的通知管理里，如需删除则需要调用`NotificationManager`对象的`deleteNotificationChannel()`。另外，系统并不会响应重复创建现有的通知渠道的操作，所以这段代码允许被重复执行。

除了将每条通知添加到相应的通知渠道之外，还应考虑通过调用`.setCategory()`为拥有特别用途的通知设置对应的分类，例如在构建闹钟通知时设置`.setCategory(CATEGORY_ALARM)`。

在后期如果想知道用户对某个通知渠道做了怎样的设置，可以通过调用`NotificationManager`对象的`getNotificationChannel()`并传入对应通知渠道的 ID 来获取到一个`NotificationChannel`对象，然后再调用这个`NotificationChannel`对象的`vibrationPattern`、`sound`、`importance`等属性就可以获取渠道的震动、声音、重要性设置，从而判断当前设置是否满足开发者的期望，如果不满足，可以尝试向用户发出消息，请求用户修改相关设置。

### Notification

创建好通知渠道后，就可以着手于创建通知的本体（`Notification`对象）了。一般为了兼容性考虑，创建`Notification`对象时会使用位于`androidx`中的`NotificationCompat`而不是`Notification`。`NotificationCompat`允许在构建通知时向其添加仅在较新版本 Android 上可用的功能，同时仍向后兼容至 Android 4.0 API 14 ，尽管这会导致诸如内嵌回复操作等部分新功能在旧版本 Android 上会变成空操作。

```kotlin
val notification = NotificationCompat.Builder(this, "testChannel").build()
```

`Builder()`接收两个参数，分别是上下文和通知渠道的 ID（这里填入了前面设置的`testChannel`），然后再调用`build()`就可以创建出一个`Notification`对象。不过这样创建的通知没有什么内容，所以一般会在调用`build()`之前调用其它的设置方法来完善这个通知，例如：

```kotlin
val notification =
    NotificationCompat.Builder(this, "testChannel")
        .setContentTitle("通知标题")  // 设置标题，可选
        .setContentText("通知内容")  // 设置内容，可选
        .setSmallIcon(R.drawable.ic_baseline_notifications_none_24)  // 设置小图标，必须
        .setLargeIcon(  // 设置大图标，可选
            BitmapFactory.decodeResource(
                Resources.getSystem(),
                R.drawable.ic_baseline_notifications_none_24
            )
        )
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)  // 设置通知在 Android 7.1 和更低版本上的干扰程度
        .build()
```

最后通过调用`NotificationManager`对象的`notify()`来发送通知，`notify()`接收两个参数，分别是应用内唯一的通知 ID 以及`Notification`对象。

### 展开式通知

#### 长文本

在一般的通知中，通知的内容就是一小句话，简明扼要，但是有时候会出现需要在`setContentText()`中增加大量的文字说明的情况，这时通知会在显示的时候将过多的文字用省略号代替。所以当通知需要显示大量文本的时候，正确的方法是在构建通知时使用`setStyle()`：

```kotlin
.setStyle(
    NotificationCompat.BigTextStyle()
        .bigText("这是一个非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常非常长的通知")
)
```

另外，如需对文本添加格式（粗体、斜体、换行等等），还可以使用 HTML 标记添加样式。

#### 内容是图片

`setStyle()`除了设置长文本的通知，还能在通知里增加图片，例如在`drawable`文件夹内有一个图片文件`pic.jpg`，那么这么做便可以将这张图片显示在通知中：

```kotlin
.setStyle(
    NotificationCompat.BigPictureStyle()
        .bigPicture(BitmapFactory.decodeResource(resources, R.drawable.pic))
)
```

拥有图片的通知可以展开或收起，当通知展开时，将在内容区域显示图片，而当通知收起时，可以通过调用`bigLargeIcon()`和`setLargeIcon()`来在通知的右方区域显示图片的缩略图。例如，现在假设在`drawable`文件夹内有一个图片文件`pic.jpg`，那么在构建通知时需要向其添加`setLargeIcon()`并传递图片，同时调用`BigPictureStyle.bigLargeIcon()`并传递`null`：

```kotlin
val pic = BitmapFactory.decodeResource(resources, R.drawable.pic)
val notification =
    NotificationCompat.Builder(this, "testChannel").setContentTitle("通知标题")
        .setContentText("通知内容")
        .setSmallIcon(R.drawable.ic_baseline_notifications_none_24)
        .setLargeIcon(pic)  // 注意
        .setStyle(BigPictureStyle().bigPicture(pic).bigLargeIcon(null))  // 注意
        .build()
```

#### addLine()

通过多次调用`addLine()`可以在通知中显示多行文本，每行文本之间都会再空一行，最多显示前 6 行：

```kotlin
val cs1: CharSequence = "第一行内容"
val cs2: CharSequence = "第二行内容"
val notification =
    NotificationCompat.Builder(this, "testChannel").setContentTitle("通知标题")
        .setContentText("通知内容")
        .setSmallIcon(R.drawable.ic_baseline_notifications_none_24)
        .setStyle(
            NotificationCompat.InboxStyle().addLine(cs1).addLine(cs2)
        ).build()
```

#### 在通知中显示对话

[在通知中显示对话](https://developer.android.com/training/notify-user/expanded#message-style)

#### 使用媒体控件创建通知

[使用媒体控件创建通知](https://developer.android.com/training/notify-user/expanded#media-style)

### 可交互通知

#### 通知点击操作

在日常使用中，用户往往可以和应用发出的通知进行交互，例如收到微信联系人的消息，微信会弹出通知，此时点击通知就可以进入聊天界面。想要让通知对用户的点击做出响应，就需要用到 PendingIntent 。

要使用 PendingIntent ，需要先创建一个`Intent`对象，这个对象指向在用户点击通知时需要启动的 Activity 或者 Service 或者 Broadcast 。随后就是创建`PendingIntent`对象，可以通过`PendingIntent`类的`getActivity()`或者`getService()`或者`getBroadcast()`来创建`PendingIntent`对象（3 个方法各自对应`Intent`对象的目标），这里以`getActivity()`为例，`getActivity()`接收 4 个参数，分别是：

1. 上下文。
2. 请求代码，不知道怎么用，一般填入`0`。
3. `Intent`对象。
4. 标记位，不知道怎么用，一般填入`0`。但是在 Android 12 API 31 及以上版本中，这个值必须要是`FLAG_IMMUTABLE`或者`FLAG_MUTABLE`二者之一。

最后在构建通知时添加`setContentIntent()`并将`PendingIntent`对象传入即可。

当用户点击了设置好 PendingIntent 的通知后，程序会去执行对应的操作，但是通知仍然存在于抽屉式通知栏，此时如果想让通知在用户点击后自动消失的话，就需要在构建通知时添加`setAutoCancel(true)`，或者手动调用`NotificationManager`对象的`cancel()`，`cancel()`接收一个整型参数，这个整型参数是通知的 ID 。

#### 通知中增加按钮

一个通知最多可以提供三个操作按钮供用户对通知进行快速响应，例如暂停闹钟提醒、回复短信，只不过这些按钮对应的操作不应该和用户在点按通知时执行的操作一样。想要添加操作按钮，就需要在构建通知时添加`addAction()`，`addAction()`接收 3 个参数，分别是代表这个操作的图标、这个操作的描述，以及一个 PendingIntent 。

#### 添加回复操作

Android 7.0 API 24 引入的直接回复操作允许用户直接在通知中输入文本以响应通知，这种行为在收到短信时很常见，当收到一条短信时，可以直接点击「回复」按钮进行回复，而无需打开短信。并且当用户完成输入后，系统会将文本附加到为通知操作指定的 Intent 中，然后将 Intent 发送给应用。

[添加直接回复操作](https://developer.android.com/training/notify-user/build-notification#reply-action)

#### 添加进度条

通知可以包含动画形式的进度指示器，向用户显示正在进行的操作的状态。

[添加进度条](https://developer.android.com/training/notify-user/build-notification#progressbar)

### 打开通知渠道设置

一个很好的例子是，在微信设置中，如果想要修改收到消息时通知的震动和响铃，微信会直接打开系统设置中的通知渠道设置，而不是在微信中进行修改，因为微信本身已经无法修改通知渠道的视觉和听觉行为，微信能做的就只有重定向至系统设置，然后让用户去修改。

想要实现这种重定向，仍然需要使用到 Intent ，通过构造一个`Intent`对象，并设置其动作为`Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS`，同时设置两个额外参数，分别是应用的包名和通知渠道的 ID ，最后调用`startActivity()`并将`Intent`对象传入即可。

```kotlin
val intent = Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS).apply {
    putExtra(Settings.EXTRA_APP_PACKAGE, packageName)
    putExtra(Settings.EXTRA_CHANNEL_ID, "testChannel")
}

startActivity(intent)
```

### 即时通讯应用的通知

[有关即时通讯应用的最佳做法](https://developer.android.com/training/notify-user/build-notification#messaging-best-practices)

## 参考资料

- [Broadcasts overview | Android Developers](https://developer.android.com/guide/components/broadcasts)
- [通知概览 | Android 开发者 | Android Developers](https://developer.android.com/guide/topics/ui/notifiers/notifications)
- 《第一行代码 Android 第3版》郭霖 著
