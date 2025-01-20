---
title: 学习 Android 自定义 View - 第 1 节
date: 2023-04-14 00:30:24
category: 笔记
---
写写停停，总共花了 4 天时间，效率有点低了。这篇笔记包括了简单的图形绘制，以及如何进行定位。

<!-- more -->

## 基本要素

- `onDraw()`：这是`View`类中的一个重要函数，用于绘制 View 的内容。在自定义 View 开发中，通常需要重写这个函数来实现自定义的绘制逻辑。`onDraw()`接收一个`Canvas`对象作为参数，它代表了当前 View 的绘图区域。
- `onMeasure()`：这是一个用于测量 View 的尺寸的函数，当自定义 View 有特殊的测量需求时，就需要重写这个方法来指定 View 的尺寸。
- `onLayout()`：该函数用于确定 View 中子 View 的位置，通常只有自定义 ViewGroup（如自定义 LinearLayout、RelativeLayout 等）时需要重写它。对于不包含子 View 的自定义 View（例如自定义 TextView），通常不需要重写这个方法。
- `Canvas`（画布）：该类是 Android 系统提供的一个用于绘图的类，它是一个 2D 绘图表面，提供了一系列绘制方法，它的实例通常作为参数传递给`onDraw()`。Canvas 不仅支持绘制基本图形，还支持矩阵变换（平移、缩放、旋转、倾斜）以及裁剪等操作。
- `Paint`（画笔）：Canvas 仅负责提供绘图方法，而具体的绘制属性（如颜色、线宽、样式等）是由`Paint`类定义的。当使用 Canvas 绘制时，需要传递一个`Paint`对象作为参数，以确定绘制的样式和属性。
- 坐标系：相比上学时学习的平面直角坐标系（笛卡尔坐标系），Android 坐标系存在以下几点不同：
  - 原点位置（0,0）位于屏幕或 View 的左上角；
  - x 轴的正方向仍然向右，但是 y 轴的正方向变成了向下；
  - 笛卡尔坐标系中第四象限的位置变成了第一象限，并且以顺时针方向增加角度，也就是以顺时针的方向定义了第二、三、四象限的位置；
  - Android 坐标系中的使用像素（px）作为坐标和尺寸的度量单位。

接下来是一个简单用例，`TestView`类是一个自定义 View ，它继承了`View`类并实现了`View`类的其中一个构造函数，随后在`onDraw()`进行了绘制：

```kotlin
// 自定义 View
class TestView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)  // Paint.ANTI_ALIAS_FLAG：开启抗锯齿
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        // 从 x=100 y=100 到 x=200 y=200 绘制一条直线
        canvas.drawLine(100f, 100f, 200f, 200f, paint)
        // 在屏幕中心绘制一个半径为 200 的圆
        canvas.drawCircle(
            width / 2f,
            height / 2f,
            TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 50f, resources.displayMetrics),
            paint
        )
    }
}
```

其中需要解释的就是`TypedValue.applyDimension()`这个方法，这是一个主要用于将其它尺寸单位转换为像素单位的方法，例如将 dp（密度无关像素）、sp（缩放无关像素）转换为 px 。该方法接收 3 个参数：

- `int unit`：要转换的尺寸单位，如`TypedValue.COMPLEX_UNIT_DIP`（表示 dp 单位）或`TypedValue.COMPLEX_UNIT_SP`（表示 sp 单位）等。
- `float value`：要转换的值。
- `DisplayMetrics metrics`：当前设备的显示度量信息，通常可以通过`getResources().getDisplayMetrics()`方法获得。

如果需要在程序启动后显示这个 View 的话，可以修改`activity_main.xml`中的代码：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.xuu6770.androidlab.TestView
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

可以看到，要使用自定义 View 就只需要在布局文件中进行引入即可，这个操作和使用内置的 View 类似。

## Path

`Path`（路径）表示一组描述图形轮廓的点和线段的集合，通过组合各种线段、曲线和形状，可以创建复杂的图形。`Path`类通常与`Canvas`和`Paint`类一起使用。

接下来看一个 Path 的用例：

```kotlin
class TestView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)  // Paint.ANTI_ALIAS_FLAG：开启抗锯齿
    private val path = Path()  // 创建一个空的 Path 对象
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        canvas.drawPath(path, paint)
    }

    // 重写该方法，该方法会在 View 的大小改变时调用
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        path.reset()
        path.addCircle(
            width / 2f,
            height / 2f,
            TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 50f, resources.displayMetrics),
            Path.Direction.CW
        )
    }
}
```

代码运行的效果同样是在屏幕中间绘制了一个圆，只不过区别于前一个例子，绘制圆的参数是定义在 Path 中的，而这个过程又是编写在了`onSizeChanged()`中，而不是在`onDraw()`中，这是因为`onDraw()`会被频繁调用，而`onSizeChanged()`只会在 View 的大小改变时调用，而绘制时需要提供的 x 坐标和 y 坐标参数也与 View 的大小（宽和高）有关。

需要注意的是`addCircle()`中的第四个参数`Path.Direction.CW`，这个参数用于指定绘制的方向，`CW`表示顺时针，逆时针则用`CCW`表示。例如在图形重叠时，绘制的方向一定程度上决定了图形重叠的部分该如何填充。来看这个用例：

```kotlin
// 省略重复代码

// 将半径单独定义出来，方便复用
private val radius =
    TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 50f, resources.displayMetrics)

override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
    super.onSizeChanged(w, h, oldw, oldh)
    path.reset()
    path.addCircle(
        width / 2f,
        height / 2f,
        radius,
        Path.Direction.CCW
    )
    // 以圆的下半部分为起点绘制了一个矩形
    path.addRect(
        width / 2f - radius,
        height / 2f,
        width / 2f + radius,
        height / 2f + 2 * radius,
        Path.Direction.CW
    )
}

// 省略重复代码
```

上面的代码在原来的基础上多绘制了一个矩形，这个矩形与圆的下半部分重叠，如果两个图形的绘制方向的参数一致的话，那么默认情况下重叠部分也会被填充，而一旦不一样，就不会填充。之所以说是默认情况，是因为是否填充还与`Path`对象的`fillType`属性相关，该属性有 4 个可选值：

- `Path.FillType.WINDING`（非零环绕数规则）：默认值。个人理解，简单来说，就是在一个封闭区域内任意取一个点，以这个点水平向右做一条射线，射线与其它图形相交时，若该图形的绘制方向为顺时针，则计数器加 1 ，否则减 1 ，最终如果计数器的值不为 0 ，则表示该封闭区域在路径内部，需要被填充。
- `Path.FillType.INVERSE_WINDING`（反向非零环绕数规则）：与`Path.FillType.WINDING`相反，顺时针减 1 ，逆时针加 1 。
- `Path.FillType.EVEN_ODD`（奇偶规则）：个人理解，该规则同样是在一个封闭区域内任意取一个点，以这个点水平向右做一条射线，计数器在每次射线与图形相交时加 1 ，最终如果交点个数为偶数，则表示该封闭区域在路径外部，否则在内部，需要被填充。
- `Path.FillType.INVERSE_EVEN_ODD`（反向奇偶规则）：与`Path.FillType.EVEN_ODD`相反，如果交点个数为偶数，则该点在路径内部，否则在外部。

当`fillType`属性取默认值时，图像的绘制方向将参与到是否要填充的决定当中，而当属性取奇偶规则时，则不关心绘制方向。

## 绘制仪表盘

接下来通过绘制一个类似于汽车的仪表盘的图形来练练手。

### 仪表盘轮廓

先绘制一个圆弧：

```kotlin
class TestView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 3f.px
    }
    private val openAngle = 120f

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        canvas.drawArc(
            width / 2f - 150f.px,
            height / 2f - 150f.px,
            width / 2f + 150f.px,
            height / 2f + 150f.px,
            90 + openAngle / 2,
            360 - openAngle,
            false,
            paint
        )
    }
}

/**
 * 为 Float 类型的数据定义一个扩展属性，将这个值的 dp 单位转换为 px 并返回
 */
val Float.px
    get() = TypedValue.applyDimension(
        TypedValue.COMPLEX_UNIT_DIP,
        this,
        Resources.getSystem().displayMetrics
    )
```

在上面的代码中，首先在对 Paint 对象初始化完成以后，设置了它的`style`属性，该属性接收一个`Paint.Style`枚举类，该类有 3 个可选值：

- `Paint.Style.FILL`：绘制几何形状时，填充其内部区域，而不绘制轮廓线条。
- `Paint.Style.STROKE`：绘制几何形状轮廓时，只绘制轮廓线条，而不填充内部区域。
- `Paint.Style.FILL_AND_STROKE`：同时填充内部区域并绘制轮廓线条。

在代码中传入了第二个参数，又因为不填充，所以需要设置`strokeWidth`这个属性（代表轮廓线条的宽度），否则看不到图形。这里的`3f.px`使用了 Kotlin 的扩展属性，并把前面将 dp 单位转 px 单位的逻辑抽了出来。最后就是使用`drawArc()`绘制一个弧线，该函数有多个重载，但传递的都是必要的参数。比如必须要有一个 RectF 对象用来限定绘制区域，在上面的代码中是通过传递矩形的四个顶点的坐标来“创建”一个 RectF 对象的。接下来的第五、六个参数则代表了圆弧的起始角度和旋转角度，这里的计算涉及到最开始提到的坐标系，`90 + openAngle / 2`的值为 150° ，是个钝角，最终定位在第二象限（左下角）。旋转（顺时针）了`360 - openAngle`也就是 240° 最终停留在第一象限（右下角）。最后传入的`false`和`paint`，前者代表了是否绘制弧线的两端点和中心点之间的连线，后者就是绘制用的 Paint 对象。

### 刻度

在原来的基础上进行改动：

```kotlin
class TestView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    private val openAngle = 120f  // 仪表盘底部的开口角度
    private val scaleWidth = 3f.px  // 刻度的宽度
    private val scaleLength = 15f.px  // 刻度的高度

    private val path = Path()  // 改用 Path 来绘制圆弧
    private lateinit var pathDashEffect: PathDashPathEffect  // 延迟初始化一个路径特效

    // 使用 Path 来绘制刻度
    private val scale = Path().apply {
        addRect(0f, 0f, scaleWidth, scaleLength, Path.Direction.CW)
    }

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 3f.px
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        // 画弧
        canvas.drawPath(path, paint)

        // 应用特效
        paint.pathEffect = pathDashEffect

        // 使用特效画出刻度
        canvas.drawPath(path, paint)

        // 取消特效
        paint.pathEffect = null
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        path.reset()
        path.addArc(
            width / 2f - 150f.px,
            height / 2f - 150f.px,
            width / 2f + 150f.px,
            height / 2f + 150f.px,
            90 + openAngle / 2,
            360 - openAngle
        )
        val pathMeasure = PathMeasure(path, false)
        pathDashEffect =
            PathDashPathEffect(
                scale,
                (pathMeasure.length - scaleWidth) / 20f,  // 假设要画 21 个刻度
                0f,
                PathDashPathEffect.Style.ROTATE
            )
    }
}

// 省略 val Float.px
```

主要有以下几点改动：

- 原来是采用`Canvas.drawArc()`来绘制圆弧，现在采用 Path 来绘制，这是因为只有这样，才能使用 PathMeasure 来对其进行测量。因为绘制圆弧涉及到 View 的宽度和高度，所以还是将绘制的具体操作放在了`onSizeChanged()`中。
- 接下来定义了一个`PathDashPathEffect`对象，`PathDashPathEffect`是`PathEffect`的一个子类，用于在路径上重复绘制一个子路径，通过这样的操作来达到某种“效果”或者是“特效”。
- 接下来同样是创建了一个 Path 对象，并向其中添加了矩形，用来表示刻度。
- 接下来在`onDraw()`中将`drawArc()`改为`drawPath()`来绘制弧，随后调用了 Paint 对象的`pathEffect`属性来为其设置一个路径特效，然后在拥有了特效的情况下再画一个弧（画出来的是一圈的刻度），最后将`pathEffect`属性的值置为`null`。Paint 对象的`pathEffect`属性用于指定绘制路径时的效果，设置属性时需要传递一个`PathEffect`参数，常用的 PathEffect 类型包括：
  - `DashPathEffect`：用于创建虚线效果。构造函数接收两个参数：一个表示虚线段长度的数组和一个表示虚线段起始位置的偏移量。数组中的值应该成对出现，例如第一个值表示实线部分的长度，第二个值表示间隙部分的长度，以此类推。
  - `CornerPathEffect`：用于将路径中的锐角变成圆角。构造函数接收一个参数，表示圆角的半径。
  - `DiscretePathEffect`：用于将路径分割成多个小段，并对每个小段进行随机偏移。构造函数接收两个参数：一个表示每段的平均长度，一个表示偏移量的最大值。可以用于创建类似于细鱼网的效果。
  - `PathDashPathEffect`：用于在路径上重复绘制一个子路径。这个类接受一个 Path 对象作为子路径，以及一个表示子路径之间间距的值。
  - `ComposePathEffect`：用于组合两个 PathEffect ，使它们在同一个路径上按顺序作用。构造函数接收两个 PathEffect 对象作为参数。
  - `SumPathEffect`：用于同时应用两个 PathEffect 到同一个路径上。这个类接受两个 PathEffect 对象作为参数。

接下来是最关键的`onSizeChanged()`的重写，首先是`PathMeasure`：

- `PathMeasure`是一个用于测量路径的类，其主要功能是对 Path 对象的长度、分割点、切线等几何信息的计算和获取。主要的函数有以下几个:
  - `PathMeasure(Path path, boolean forceClosed)`：构造函数，其中`path`为要测量的 Path 对象，`forceClosed`代表是否要强制将路径进行闭合。
  - `getLength()`：返回关联路径的总长度。
  - `getPosTan(float distance, float[] pos, float[] tan)`：获取路径上指定距离的点的位置和切线。第一个参数为路径上的距离，第二个参数为长度为 2 的数组，用于存储路径上指定距离的点的 x 和 y 坐标。如果不需要这个值，可以传入`null`。第三个参数也是长度为 2 的数组，用于存储路径上指定距离的点的切线的 x 和 y 分量。如果不需要这个值，可以传入`null`，该函数本身则返回一个布尔值。
- 然后是对前面定义的 PathDashPathEffect 对象进行初始化，构造函数接收 4 个参数：
  - `Path shape`：这是一个子路径，表示要沿着路径重复绘制的形状，这个形状可以是线段、圆形、矩形等。
  - `float advance`：由于是重复绘制，所以这个值代表了重复绘制之间的间距。如果这个值比子路径的长度大，那么在重复绘制的时候，图形之间会有间隙；如果这个值比子路径的长度小，那么图形就会发生重叠。
  - `float phase`：这是子路径的起始位置偏移量。通过改变这个值，可以控制子路径在路径上的起始位置。如果这个值为 0，则第一个子路径将从路径的起点开始绘制。增加这个值将使子路径沿着路径移动。
  - `PathDashPathEffect.Style style`：这是用于控制子路径与原路径相交时的表现方式的枚举值，包含三个可选值：
    - `Style.TRANSLATE`：子路径会平移，使其与原路径相交。
    - `Style.ROTATE`：子路径会旋转，使其与原路径的切线对齐。
    - `Style.MORPH`：子路径会变形，使其与原路径相交，并沿着原路径的曲率变化。

参照以上函数的说明和参数的意义，代码将变得好理解，唯一还需要解释的就是 PathDashPathEffect 构造函数中的第二个参数`(pathMeasure.length - scaleWidth) / 20f`。这个参数首先通过`pathMeasure.length`拿到被测量的 Path 对象的长度，然后减去一个刻度的宽度，再分成 20 份，每一份的长度就是每个刻度之间的间距，这里要注意，每一份的长度实际上包括了刻度本身的宽度在里面，这也是为什么上面说“值比子路径的长度大，那么在重复绘制的时候，图形之间会有间隙；如果这个值比子路径的长度小，那么图形就会发生重叠”。这 20 份长度，每一份的头都会连着前一份的尾，这样一来，第 21 个刻度所在的位置，就是第 20 份刻度的尾巴，这就保证了圆弧的最尾巴的位置会始终有一个刻度，这样绘制出来的仪表盘就更像现实中的仪表盘。

### 指针

加上绘制指针后的完整代码如下：

```kotlin
class TestView(context: Context?, attrs: AttributeSet?) : View(context, attrs) {
    private val openAngle = 120f  // 仪表盘底部的开口角度
    private val scaleWidth = 3f.px  // 刻度的宽度
    private val scaleLength = 15f.px  // 刻度的高度
    private val needleLength = 120f.px  // 仪表盘指针长度
    private val pointTo = 10  // 指针指向的刻度（不包括起始刻度）

    private val path = Path()  // 改用 Path 来绘制圆弧
    private lateinit var pathDashEffect: PathDashPathEffect  // 延迟初始化一个路径特效

    // 使用 Path 来绘制刻度
    private val scale = Path().apply {
        addRect(0f, 0f, scaleWidth, scaleLength, Path.Direction.CW)
    }

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 3f.px
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        // 画弧
        canvas.drawPath(path, paint)

        // 应用特效
        paint.pathEffect = pathDashEffect

        // 使用特效画出刻度
        canvas.drawPath(path, paint)

        // 取消特效
        paint.pathEffect = null

        // 画指针
        canvas.drawLine(
            width / 2f,
            height / 2f,
            (width / 2f + needleLength * cos(Math.toRadians((90 + openAngle / 2f + (360 - openAngle) / 20f * pointTo).toDouble()))).toFloat(),
            (height / 2f + needleLength * sin(Math.toRadians((90 + openAngle / 2f + (360 - openAngle) / 20f * pointTo).toDouble()))).toFloat(),
            paint
        )
    }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        path.reset()
        path.addArc(
            width / 2f - 150f.px,
            height / 2f - 150f.px,
            width / 2f + 150f.px,
            height / 2f + 150f.px,
            90 + openAngle / 2,
            360 - openAngle
        )
        val pathMeasure = PathMeasure(path, false)
        pathDashEffect =
            PathDashPathEffect(
                scale,
                (pathMeasure.length - scaleWidth) / 20f,
                0f,
                PathDashPathEffect.Style.ROTATE
            )
    }
}

/**
 * 为 Float 类型的数据定义一个扩展属性，将这个值的 dp 单位转换为 px 并返回
 */
val Float.px
    get() = TypedValue.applyDimension(
        TypedValue.COMPLEX_UNIT_DIP, this, Resources.getSystem().displayMetrics
    )
```

首先添加了两个变量`needleLength`和`pointTo`，重点还是来看在`onDraw()`中的改动。在`onDraw()`使用`Canvas.drawLine()`来绘制直线，该函数的前两个参数就是起始点的横纵坐标，后两个参数是终点的横纵坐标。起点的横纵坐标其实就是仪表盘的中心也就是 View 的中心，而终点的横纵坐标的计算则涉及到三角函数。

参与三角函数运算的参数有三个，分别是一个角度和两条边，其中一条边就是斜边（指针长度），只要把角度算出来，就可以通过这两个已知量算出第三个值，也就是算出对边（纵坐标的值）或者临边（横坐标的值）。

角度的计算也不算难，由于仪表盘始于第二象限，所以第一象限 90° 是固定死的了，然后加上仪表盘开口角度的一半等于仪表盘起始位置的角度。然后再将仪表盘转过的角度除以 20 得到一个刻度的度数，最后再乘以需要指向的刻度，将得到的角度加上仪表盘起始位置的角度等于最终的角度。

有了角度和斜边长度，就可以通过 Kotlin 中的`cos()`来计算临边的长度，通过`sin()`来计算对边的长度。需要注意的是这两个函数接收的参数的单位是“弧度制”，而前面一系列的计算得到的值的单位是角度制，所以还需要通过调用`toRadians()`来将其转换为弧度制。

### 简单总结

绘制少量简单图形时可以直接用 Canvas 的各种 draw 函数，当需要绘制复杂图形的时候可以考虑使用 Path ，因为有 Path 才能使用 PathMeasure 进行测量，并且 Path 在一处被定义了以后，可以通过`Canvas.drawPath()`重复调用来达到代码复用的目的。

三角函数在计算坐标时似乎很常用，在已知起点坐标的情况下，通过角度和三角函数，便可以得到终点的横纵坐标。

## 绘制饼图

绘制一个简单的饼图没有仪表盘那么复杂：

```kotlin
class PieView(context: Context, attrs: AttributeSet?) : View(context, attrs) {
    private val radius = 150f.px
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    private val angles = floatArrayOf(30f, 60f, 120f, 150f)  // 定义一组角度

    // 定义一组颜色
    private val colors = listOf(
        Color.parseColor("#FF3333"),
        Color.parseColor("#E5FFCC"),
        Color.parseColor("#9999FF"),
        Color.parseColor("#99004C")
    )

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        var startAngle = 0f  // 定义起始角度
        for ((index, angle) in angles.withIndex()) {
            paint.color = colors[index]
            canvas.drawArc(
                width / 2f - radius,
                height / 2f - radius,
                width / 2f + radius,
                height / 2f + radius,
                startAngle,
                angle,
                true,
                paint
            )
            startAngle += angle  // 更新起始角度
        }
    }
}
```

解释：

- 这里把`150f.px`单独作为一个变量提取出来，方便复用和修改。同时复用了前面的`Float.px`这个扩展属性，只是上面的代码没写。
- 增加了两个新变量，分别是一组角度和一组颜色，对应饼图中的每块区域。
- 在`onDraw()`使用`Canvas.drawArc()`来绘制饼图，因为饼图本质上是由多个圆弧组成的一个完整的圆，其本身并不是一个圆，所以不应该使用`drawCircle()`。
- 因为需要画 4 块扇形，所以这里使用了循环语句来进行重复绘制操作。关于 Kotlin 中循环语句的语法，这里不多阐述。
- 使用`Canvas.drawArc()`来绘制每一块“饼”，其中第五和第六个参数代表弧的起始角度和结束角度，第七个参数`useCenter`是一个布尔类型的参数，用于指定绘制的弧形是否包含圆心。当这个值为`true`时，绘制的弧形会包含圆心，也就是从圆心开始绘制弧形。这时候，绘制的图形是一个扇形。否则绘制的弧形不会包含圆心，也就是从圆的边缘开始绘制弧形。这时候，绘制的图形是一个弧线。

### 位置偏移

在某些炫酷的图表库当中，当选中饼图当中的某块扇形区域时，该区域会向后偏移并高亮，这里记录以一下如何实现简单的图形偏移：

```kotlin
private val offset = 30f.px  // 图形偏移量
    
override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)
    var startAngle = 0f
    for ((index, angle) in angles.withIndex()) {
        paint.color = colors[index]
        if (index == 1) {
            canvas.save()
            canvas.translate(
                (offset * cos(Math.toRadians((startAngle + angle / 2f).toDouble()))).toFloat(),
                (offset * sin(Math.toRadians((startAngle + angle / 2f).toDouble()))).toFloat()
            )
        }
        canvas.drawArc(
            width / 2f - radius,
            height / 2f - radius,
            width / 2f + radius,
            height / 2f + radius,
            startAngle,
            angle,
            true,
            paint
        )
        startAngle += angle
        if (index == 1) {
            canvas.restore()
        }
    }
}
```

上面的代码仅对第二个扇形（下标为 1 ）进行了偏移，绘制图形前保存了 Canvas 的状态并设置了偏移量，绘制完成后恢复了 Canvas 的状态。计算偏移后的坐标时同样用到了三角函数，偏移的方向为扇形张开角度的一半。
