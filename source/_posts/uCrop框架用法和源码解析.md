---
title: uCrop框架用法和源码解析
categories:
  - Android
tags:
  - Android
  - 源码
  - uCrop
  - 图片处理
comments: true
copyright: true
abbrlink: 4e4b8b52
date: 2019-08-04 17:37:57
updated: 2019-08-04 17:37:57
---
>本人能力不足，在看到源码最后一部分的时候大量抄袭[可能是最详细的UCrop源码解析](https://www.jianshu.com/p/1d3fb16fb412)

# 1. uCrop简介
uCrop是目前较火的图片裁剪框架，开发者宣称他会比目前市面上所有的图片裁剪方案都要更流畅。外加他封装程度较高，可自定义，而且颜值很高（似乎这个才是重点），现在越来越多APP选择使用它。
[github](https://github.com/Yalantis/uCrop)
<!-- More -->

# 2. 使用方法
得益于uCrop优秀的封装，uCrop的使用方法特简单。
## 2.1 导入依赖
1. 先在项目的`build.gradle`中添加

    ```java
    allprojects {
      repositories {
        jcenter()
        maven { url "https://jitpack.io" }
      }
    }
    ```

    并在`module`的`build.gradle`中添加
    
    `implementation 'com.github.yalantis:ucrop:2.2.3'` - 轻量级框架
    
    `implementation 'com.github.yalantis:ucrop:2.2.3-native'` - 获得框架全部强大的功能以及图片的高质量(最终可能会导致apk的大小增加1.5MB以上)
2. 由于框架的本质是调用到另一个Activity去处理图片，所以需要在AndroidManifest.xml中将UCropActivity添加进去
    ```xml
    <activity
        android:name="com.yalantis.ucrop.UCropActivity"
        android:screenOrientation="portrait"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar"/>
    ```

到这你就能把cUrop全部导入到你的项目里面了，接下来咱们就拉将如何调用

## 2.2 开始基本的调用
调用起来很简单：
```java
UCrop.of(sourceUri, destinationUri)
    .start(context);
```
其中`sourceUri`是输入图片的`Uri`，`destinationUri`是输出图片的`Uri`。然后他就会由`Intent`的调动跳到`UCropActivity`，用户就在`UCropActivity`里面进行图片裁剪操作，然后最后由`UCropActivity`发起一个`Intent`回到你的`Activity`。

## 2.3 处理回来的数据
由于是从`UCropAcitivity`传回数据，所以你需要在你的`Activity`里面的`onActivityResult`方法处理`uCrop`返回的信息：
```java
@Override
public void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (resultCode == RESULT_OK && requestCode == UCrop.REQUEST_CROP) {
        final Uri resultUri = UCrop.getOutput(data);
    } else if (resultCode == UCrop.RESULT_ERROR) {
        final Throwable cropError = UCrop.getError(data);
    }
}
```

到这，基本用法就完了，你就可以尽情的使用uCrop。但是我前面说过，uCrop封装程度好，这点很多图片处理框架都可以做到，基本上都是把需要的数据传到自己的Activity之后由自己的Activity处理，所以很多框架看起来都有优秀的封装，那uCrop相比其他又有啥好呢，答案就是自定义灵活：

## 2.4 uCrop高阶用法

### 2.4.1 配置uCrop
```java
/**
  * 启动裁剪
  * @param activity 上下文
  * @param sourceFilePath 需要裁剪图片的绝对路径
  * @param requestCode 比如：UCrop.REQUEST_CROP
  * @param aspectRatioX 裁剪图片宽高比
  * @param aspectRatioY 裁剪图片宽高比
  * @return
  */
public static String startUCrop(Activity activity, String sourceFilePath, 
	int requestCode, float aspectRatioX, float aspectRatioY) {
    Uri sourceUri = Uri.fromFile(new File(sourceFilePath));
    File outDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES);
    if (!outDir.exists()) {
        outDir.mkdirs();
    }
    File outFile = new File(outDir, System.currentTimeMillis() + ".jpg");
    //裁剪后图片的绝对路径
    String cameraScalePath = outFile.getAbsolutePath();
    Uri destinationUri = Uri.fromFile(outFile);
    //初始化，第一个参数：需要裁剪的图片；第二个参数：裁剪后图片
    UCrop uCrop = UCrop.of(sourceUri, destinationUri);
    //初始化UCrop配置
    UCrop.Options options = new UCrop.Options();
    //设置裁剪图片可操作的手势
    options.setAllowedGestures(UCropActivity.SCALE, UCropActivity.ROTATE, UCropActivity.ALL);
    //是否隐藏底部容器，默认显示
    options.setHideBottomControls(true);
    //设置toolbar颜色
    options.setToolbarColor(ActivityCompat.getColor(activity, R.color.colorPrimary));
    //设置状态栏颜色
    options.setStatusBarColor(ActivityCompat.getColor(activity, R.color.colorPrimary));
    //是否能调整裁剪框
    options.setFreeStyleCropEnabled(true);
    //UCrop配置
    uCrop.withOptions(options);
    //设置裁剪图片的宽高比，比如16：9
    uCrop.withAspectRatio(aspectRatioX, aspectRatioY);
    //uCrop.useSourceImageAspectRatio();
    //跳转裁剪页面
    uCrop.start(activity, requestCode);
    return cameraScalePath;
}
```

### 2.4.2 其他配置
```java
//设置Toolbar标题
void setToolbarTitle(@Nullable String text)
//设置裁剪的图片格式
void setCompressionFormat(@NonNull Bitmap.CompressFormat format)
//设置裁剪的图片质量，取值0-100
void setCompressionQuality(@IntRange(from = 0) int compressQuality)
//设置最多缩放的比例尺
void setMaxScaleMultiplier(@FloatRange(from = 1.0, fromInclusive = false) float maxScaleMultiplier)
//动画时间
void setImageToCropBoundsAnimDuration(@IntRange(from = 100) int durationMillis)
//设置图片压缩最大值
void setMaxBitmapSize(@IntRange(from = 100) int maxBitmapSize)
//是否显示椭圆裁剪框阴影
void setOvalDimmedLayer(boolean isOval) 
//设置椭圆裁剪框阴影颜色
void setDimmedLayerColor(@ColorInt int color)
//是否显示裁剪框
void setShowCropFrame(boolean show)
//设置裁剪框边的宽度
void setCropFrameStrokeWidth(@IntRange(from = 0) int width)
//是否显示裁剪框网格
void setShowCropGrid(boolean show) 
//设置裁剪框网格颜色
void setCropGridColor(@ColorInt int color)
//设置裁剪框网格宽
void setCropGridStrokeWidth(@IntRange(from = 0) int width)
```

# 3. 源码解析
>在我开始说源码之前，我建议大家可以先看下我下面的连接，因为本框架的作者真的是个好人，他不仅为我们贡献了这么好的一个框架，还把自己写这个框架的思路都写了出来，大家可以看看
[英文原版](https://yalantis.com/blog/how-we-created-ucrop-our-own-image-cropping-library-for-android/)
[国内网友翻译版](https://blog.csdn.net/wood_water_peng/article/details/51306274)
[百度网页翻译机翻版](https://fanyi.baidu.com/transpage?query=https%3A%2F%2Fyalantis.com%2Fblog%2Fhow-we-created-ucrop-our-own-image-cropping-library-for-android%2F%23&source=url&ie=utf8&from=auto&to=zh&render=1)
其实我个人感觉百度机翻没有谷歌翻译的好，大家有条件的可以使用谷歌翻译浏览器插件翻译整个网页（谷歌翻译好像国内可以直接访问）

代码结构大致分为三个部分:
## 3.1 第一部分：UCropActivity（整个框架的外在，用户操作图片的地方）
他的功能就是项目主要的界面，以及实现一些基本的初始化。你跳转到uCrop看到的那个操作图片的界面就是它。

这块看源码的时候代码居多，但是，说实话，就像刚刚说的一样，他除了初始化还是初始化。初始化完`Toolbar`接着初始化`ViewGroup`，初始化完`ViewGroup`接着初始化`Image`数据等等。所以这块我就没咋细看~~（其实是因为代码太长了，逃）~~
## 3.2 第二部分：OverlayView（绘制裁剪框）
这一块主要就是来画你所看到的图片中的裁剪的辅助线。

在构造方法里面就调用了一个方法，就是`init()`，而`init()`方法也就干了一件事——判断。当系统小于`JELLY_BEA_MR2`也就是`Android4.3`时，启动了硬件加速，至于为什么`setLayerType(LAYER_TYPE_SOFTWARE, null);`这个看着就像启动硬件加速的方法，甚至参数里面还有软件这个单词的方法能启动硬件加速，请大家移步[HenCoder Android 自定义 View 1-8 硬件加速](https://hencoder.com/ui-1-8/)（进去直接搜索这个方法即可，就能找到解释的地方），我再次不做解释。

这个类主要有两个方法
1. `drawDimmedLayer()`绘制裁剪框之外的灰色部分
2. `drawCropGrid()`绘制裁剪框

那我们分别来看下这两个方法：
### 3.2.1 drawDimmedLayer()
```java
protected void drawDimmedLayer(@NonNull Canvas canvas) {
    //先保存当前当前画布
    canvas.save();
    //判断是否显示圆框
    if (mCircleDimmedLayer) {
        //按Path路径裁剪
        canvas.clipPath(mCircularPath, Region.Op.DIFFERENCE);
    } else {
        //裁剪矩形
        canvas.clipRect(mCropViewRect, Region.Op.DIFFERENCE);
    }
    //着色
    canvas.drawColor(mDimmedColor);
    //恢复之前保存的Canvas的状态
    canvas.restore();

    if (mCircleDimmedLayer) { // 绘制1px笔划以修复反锯齿
        canvas.drawCircle(mCropViewRect.centerX(), mCropViewRect.centerY(),
                Math.min(mCropViewRect.width(), mCropViewRect.height()) / 2.f, mDimmedStrokePaint);
    }
}
```
首先就是一个`mCircleDimmedLayer`，这个我真的很迷，因为我不知道她是咋来的，于是我就看`OverlayView`有没有对这个变量的赋值，于是整个类我就找到了一个`setCircleDimmedLayer()`方法，于是我看这个方法是在哪被调用了的，然后我就找到他分别被UCropActivity和UCropFragment两个类调用到，而且一个是`intent.getBooleanExtra()`方法一个是`bundle.getBoolean()`方法，看到这个我相信大家都有点数了，这明显就是其他类传过来的啊，我发现他两的key的值都是`UCrop.Options.EXTRA_CIRCLE_DIMMED_LAYER`，那我就懂了，找整个框架里面哪儿提到过这个值不就得了，于是我就发现除了上面两个方法以及他的初始化以外，我发现了第4个调用的地方，也是唯一一个调用的地方——Ucrop.setCircleDimmedLayer()：
```java
/**
 * @param isCircle - set it to true if you want dimmed layer to have an circle inside
 * iscircle-如果希望暗显层中有一个圆，请将其设置为true。
 */
public void setCircleDimmedLayer(boolean isCircle) {
    mOptionBundle.putBoolean(EXTRA_CIRCLE_DIMMED_LAYER, isCircle);
}
```
注释上面是原话，下面是我百度机翻的翻译。看了就懂了吧，反正我没懂，我也完全没有见到哪调用过这个方法，我更不懂啥叫希望暗显层有个圆，啥玩意？充满线条的黑？？？
直到我将UCrop的调用方法修改了并运行之后我才懂了：
```java
val options = UCrop.Options()
options.setCircleDimmedLayer(true)
UCrop.of(uri, destinationUri)
        .withOptions(options)
        .start(this)
```
结果是：
![Screenshot_20190802-211515_PhotoXiu_puzzle](https://cdn.littlecorgi.top/mweb/Screenshot_20190802-211515_PhotoXiu_puzzle.jpg)

然后就懂了，应该是能截一个圆形的图案吧，然后我点下了✔️，然后……

![Screenshot_20190802-211547_PhotoXiu_puzzle](https://cdn.littlecorgi.top/mweb/Screenshot_20190802-211547_PhotoXiu_puzzle.jpg)
无话可说，作者牛逼！！！


回去回去，刚刚说到`drawDimmedLayer()`，可以看到，如果`mCircleDimmedLayer`为`true`就调用`clipPath()`跟着路径裁切一个矩形加原，不然的话就调用`clipRect()`裁切一个矩形。然后加入颜色，然后完了

### 3.2.2 drawCropGrid()
```java
protected void drawCropGrid(@NonNull Canvas canvas) {
    // 判断是否显示剪裁框
    if (mShowCropGrid) {
        // 判断矩形数据是否为空，mGridPoints 如果等于空的话进入填充数据
        if (mGridPoints == null && !mCropViewRect.isEmpty()) {
            // 该数组为 canvas.drawLines 的第一个参数，该参数要求其元素个数为 4 的倍数
            mGridPoints = new float[(mCropGridRowCount) * 4 + (mCropGridColumnCount) * 4];

            int index = 0;
            // 组装数据，数据为每一组线段的坐标点
            for (int i = 0; i < mCropGridRowCount; i++) {
                mGridPoints[index++] = mCropViewRect.left;
                mGridPoints[index++] = (mCropViewRect.height() * (((float) i + 1.0f) / (float) (mCropGridRowCount + 1))) + mCropViewRect.top;
                mGridPoints[index++] = mCropViewRect.right;
                mGridPoints[index++] = (mCropViewRect.height() * (((float) i + 1.0f) / (float) (mCropGridRowCount + 1))) + mCropViewRect.top;
            }

            for (int i = 0; i < mCropGridColumnCount; i++) {
                mGridPoints[index++] = (mCropViewRect.width() * (((float) i + 1.0f) / (float) (mCropGridColumnCount + 1))) + mCropViewRect.left;
                mGridPoints[index++] = mCropViewRect.top;
                mGridPoints[index++] = (mCropViewRect.width() * (((float) i + 1.0f) / (float) (mCropGridColumnCount + 1))) + mCropViewRect.left;
                mGridPoints[index++] = mCropViewRect.bottom;
            }
        }
        //绘制线段
        if (mGridPoints != null) {
            canvas.drawLines(mGridPoints, mCropGridPaint);
        }
    }
    //绘制矩形包裹线段
    if (mShowCropFrame) {
        canvas.drawRect(mCropViewRect, mCropFramePaint);
    }
    //绘制边角包裹,mFreestyleCropMode此参数如果等于1的话 剪裁框为可移动状态，一般不用
    if (mFreestyleCropMode != FREESTYLE_CROP_MODE_DISABLE) {
        canvas.save();

        mTempRect.set(mCropViewRect);
        mTempRect.inset(mCropRectCornerTouchAreaLineLength, -mCropRectCornerTouchAreaLineLength);
        canvas.clipRect(mTempRect, Region.Op.DIFFERENCE);

        mTempRect.set(mCropViewRect);
        mTempRect.inset(-mCropRectCornerTouchAreaLineLength, mCropRectCornerTouchAreaLineLength);
        canvas.clipRect(mTempRect, Region.Op.DIFFERENCE);

        canvas.drawRect(mCropViewRect, mCropFrameCornersPaint);

        canvas.restore();
    }
}
```
一开头又是一个和上面类似的变量`mShowCropGrid`，这下我就不说我找的具体步骤，他的功能就是如果他是`true`就会在裁剪框中显示9宫格线，为`false`就没有。接着就是画线部分，我觉得这个我不用讲啥，也没啥讲的，唯一就是为什么mGridPoints这个数组的大小是4的倍数，大家可以看下这个博客[Android Canvas DrawLines中第一个参数的解释](https://blog.csdn.net/pimkle/article/details/16946423)


## 3.3 第三部分：GestureCropImageView（正在框架的内在，代码操作操作图片的地方）
这个是整个项目最核心的地方。前面的两部分都是UI的，而这个才是真正的对图片进行处理的部分，也是我最想知道了解的部分。
这部分作者也在他的博客里面说的最多最清楚。
作者把这部分的逻辑分为了三个部分
1. `TransformImageView extends ImageView`
    他处理了
    1. 从源拿到图片
    2. 将图片进变换（平移、缩放、旋转），并应用到当前图片上

2. `CropImageView extends TransformImageView`
    他处理了
    1. 绘制裁剪边框和网格
    2. 为裁剪区域设置一张图片（如果用户对图片操作导致裁剪区域出现了空白，那么图片应自动移动到边界填充空白区域）
    3. 继承父类方法，使用更精准的规则来操作矩阵（限制最大和最小缩放比）
    4. 添加方法和缩小的方法
    5. 裁剪图片
3. `GestureCropImageView extends CropImageView`
    他处理了
    1. 监听用户手势，并调用对应的正确的方法

### 3.3.1 TransformImageView
作者说这是最容易的部分。
在看这个类之前我们先来看看`BitmapLoadTask`类，这个类是一切图像处理的基础，这个类负责了`Uri`解码`bitmap`，并处理分辨率：
首先根据拿到的`Uri`解析位图：
```java
final ParcelFileDescriptor parcelFileDescriptor;
try {
    parcelFileDescriptor = mContext.getContentResolver().openFileDescriptor(mInputUri, "r");
} catch (FileNotFoundException e) {
    return new BitmapWorkerResult(e);
}

final FileDescriptor fileDescriptor;
if (parcelFileDescriptor != null) {
    fileDescriptor = parcelFileDescriptor.getFileDescriptor();
} else {
    return new BitmapWorkerResult(new NullPointerException("ParcelFileDescriptor was null for given Uri: [" + mInputUri + "]"));
}
```
现在，可以使用`BitmapFactory`方法解码`FileDescriptor`。

但在解码位图之前，有必要知道它的大小，因为如果分辨率太高，位图将被二次采样。
```java
final BitmapFactory.Options options = new BitmapFactory.Options();



options.inJustDecodeBounds = true;

BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);

options.inSampleSize = calculateInSampleSize(options, requiredWidth, requiredHeight);

options.inJustDecodeBounds = false;



Bitmap decodeSampledBitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);

close(parcelFileDescriptor);



ExifInterface exif = getExif(uri);

if (exif != null) {

  int exifOrientation = exif.getAttributeInt(ExifInterface.TAG_ORIENTATION, ExifInterface.ORIENTATION_NORMAL);

  return rotateBitmap(decodeSampledBitmap, exifToDegrees(exifOrientation));

} else {

  return decodeSampledBitmap;

}
```
这样就拿到了`bitmap`实例了，就可以去`TansformImageView`去对图片进行调整了。
其实这个类我也不知道说啥😂，我觉得这个类也就是把`Matrix`的`postTranslate()`、`postRotate()`和`postScale()`方法给封装了下。
关于Matrix的知识大家可以参考这篇博客：[安卓自定义View进阶-Matrix原理](https://www.gcssloop.com/customview/Matrix_Basic)

### 3.3.2 CropImageView
这一层是最复杂的一层，作者的操作大致可以分为3步：图片裁剪框偏移计算、图片归为动画处理、裁剪图片

- 第一步：图片裁剪框偏移计算
当用户手指移开时，要确保图片处于裁剪区域中，如果不处于，需要通过平移把它移过来：

```java
public void setImageToWrapCropBounds(boolean animate) {
    //如果图片加载完毕并且图片不处于剪裁区域
    if (mBitmapLaidOut && !isImageWrapCropBounds()) {

        //获取中心点X,Y坐标
        float currentX = mCurrentImageCenter[0];
        float currentY = mCurrentImageCenter[1];
        //获取缩放比例
        float currentScale = getCurrentScale();

        //获取偏移距离
        float deltaX = mCropRect.centerX() - currentX;
        float deltaY = mCropRect.centerY() - currentY;
        float deltaScale = 0;

        mTempMatrix.reset();
        mTempMatrix.setTranslate(deltaX, deltaY);

        final float[] tempCurrentImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
        mTempMatrix.mapPoints(tempCurrentImageCorners);

        //判断图片是否包含在剪裁区域
        boolean willImageWrapCropBoundsAfterTranslate = isImageWrapCropBounds(tempCurrentImageCorners);

        //如果包含在剪裁区域
        if (willImageWrapCropBoundsAfterTranslate) {
            //获取偏移的距离
            final float[] imageIndents = calculateImageIndents();
            //偏移的距离，横坐标加横坐标 纵坐标加纵坐标
            deltaX = -(imageIndents[0] + imageIndents[2]);
            deltaY = -(imageIndents[1] + imageIndents[3]);
        } else {
            //如果不包含在剪裁区域，创建临时矩形
            RectF tempCropRect = new RectF(mCropRect);
            mTempMatrix.reset();
            //设置偏移角度
            mTempMatrix.setRotate(getCurrentAngle());
            mTempMatrix.mapRect(tempCropRect);

            //获得矩形的边长坐标
            final float[] currentImageSides = RectUtils.getRectSidesFromCorners(mCurrentImageCorners);
            //获取放大比例
            deltaScale = Math.max(tempCropRect.width() / currentImageSides[0],
                    tempCropRect.height() / currentImageSides[1]);
            deltaScale = deltaScale * currentScale - currentScale;
        }

        //如果需要动画
        if (animate) {
            post(mWrapCropBoundsRunnable = new WrapCropBoundsRunnable(
                    CropImageView.this, mImageToWrapCropBoundsAnimDuration, currentX, currentY, deltaX, deltaY,
                    currentScale, deltaScale, willImageWrapCropBoundsAfterTranslate));
        } else {
            //不需要动画，直接移动到目标位置
            postTranslate(deltaX, deltaY);
            if (!willImageWrapCropBoundsAfterTranslate) {
                zoomInImage(currentScale + deltaScale, mCropRect.centerX(), mCropRect.centerY());
            }
        }
    }
}
```

- 第二步：处理平移
通过一个Runnable线程来处理平移，并且通过时间差值的计算来移动动画，使动画看起来更真实：
```java
/**
 * 此可运行文件用于动画图像，使其完全填充裁剪边界。
 * 给定值在动画期间内插。
 * runnable可以终止于vie{@link #cancelAllAnimations()}方法，
 * 也可以在触发{@link WrapCropBoundsRunnable#run()}方法内的某些条件时终止。
 */
private static class WrapCropBoundsRunnable implements Runnable {

    private final WeakReference<CropImageView> mCropImageView;

    private final long mDurationMs, mStartTime;
    private final float mOldX, mOldY;
    private final float mCenterDiffX, mCenterDiffY;
    private final float mOldScale;
    private final float mDeltaScale;
    private final boolean mWillBeImageInBoundsAfterTranslate;

    public WrapCropBoundsRunnable(CropImageView cropImageView,
                                long durationMs,
                                float oldX, float oldY,
                                float centerDiffX, float centerDiffY,
                                float oldScale, float deltaScale,
                                boolean willBeImageInBoundsAfterTranslate) {

        mCropImageView = new WeakReference<>(cropImageView);

        mDurationMs = durationMs;
        mStartTime = System.currentTimeMillis();
        mOldX = oldX;
        mOldY = oldY;
        mCenterDiffX = centerDiffX;
        mCenterDiffY = centerDiffY;
        mOldScale = oldScale;
        mDeltaScale = deltaScale;
        mWillBeImageInBoundsAfterTranslate = willBeImageInBoundsAfterTranslate;
    }

    @Override
    public void run() {
        CropImageView cropImageView = mCropImageView.get();
        if (cropImageView == null) {
            return;
        }

        long now = System.currentTimeMillis();
        float currentMs = Math.min(mDurationMs, now - mStartTime);

        float newX = CubicEasing.easeOut(currentMs, 0, mCenterDiffX, mDurationMs);
        float newY = CubicEasing.easeOut(currentMs, 0, mCenterDiffY, mDurationMs);
        float newScale = CubicEasing.easeInOut(currentMs, 0, mDeltaScale, mDurationMs);

        if (currentMs < mDurationMs) {
            cropImageView.postTranslate(newX - (cropImageView.mCurrentImageCenter[0] - mOldX), newY - (cropImageView.mCurrentImageCenter[1] - mOldY));
            if (!mWillBeImageInBoundsAfterTranslate) {
                cropImageView.zoomInImage(mOldScale + newScale, cropImageView.mCropRect.centerX(), cropImageView.mCropRect.centerY());
            }
            if (!cropImageView.isImageWrapCropBounds()) {
                cropImageView.post(this);
            }
        }
    }
}
```

下面还有另一个线程，用于双击放大:

```java
/**
 * 此可运行项用于设置图像缩放的动画。
 * 给定值在动画期间内插。
 * runnable可以终止vie {@link #cancelAllAnimations()}方法，
 * 也可以在触发{@link ZoomImageToPosition#run()}方法内的某些条件时终止。
 */
private static class ZoomImageToPosition implements Runnable {

    private final WeakReference<CropImageView> mCropImageView;

    private final long mDurationMs, mStartTime;
    private final float mOldScale;
    private final float mDeltaScale;
    private final float mDestX;
    private final float mDestY;

    public ZoomImageToPosition(CropImageView cropImageView,
                                long durationMs,
                                float oldScale, float deltaScale,
                                float destX, float destY) {

        mCropImageView = new WeakReference<>(cropImageView);

        mStartTime = System.currentTimeMillis();
        mDurationMs = durationMs;
        mOldScale = oldScale;
        mDeltaScale = deltaScale;
        mDestX = destX;
        mDestY = destY;
    }

    @Override
    public void run() {
        CropImageView cropImageView = mCropImageView.get();
        if (cropImageView == null) {
            return;
        }

        long now = System.currentTimeMillis();
        float currentMs = Math.min(mDurationMs, now - mStartTime);
        float newScale = CubicEasing.easeInOut(currentMs, 0, mDeltaScale, mDurationMs);

        if (currentMs < mDurationMs) {
            cropImageView.zoomInImage(mOldScale + newScale, mDestX, mDestY);
            cropImageView.post(this);
        } else {
            cropImageView.setImageToWrapCropBounds();
        }
    }
}
```

- 第三步：裁剪图片

```java
/**
 * 取消所有当前动画并设置图像以填充裁剪区域（不带动画）。
 * 然后用适当的参数创建并执行{@link BitmapCropTask}。
 */
public void cropAndSaveImage(@NonNull Bitmap.CompressFormat compressFormat, int compressQuality,
                            @Nullable BitmapCropCallback cropCallback) {
    //结束子线程
    cancelAllAnimations();
    //设置要剪裁的图片，不需要位移动画
    setImageToWrapCropBounds(false);

    //存储图片信息，四个参数分别为：mCropRect要剪裁的图片矩阵，当前图片要剪裁的矩阵，当前放大的值，当前旋转的角度
    final ImageState imageState = new ImageState(
            mCropRect, RectUtils.trapToRect(mCurrentImageCorners),
            getCurrentScale(), getCurrentAngle());

    //剪裁参数，mMaxResultImageSizeX，mMaxResultImageSizeY：剪裁图片的最大宽度、高度。
    final CropParameters cropParameters = new CropParameters(
            mMaxResultImageSizeX, mMaxResultImageSizeY,
            compressFormat, compressQuality,
            getImageInputPath(), getImageOutputPath(), getExifInfo());

    //剪裁操作放到AsyncTask中执行
    new BitmapCropTask(getViewBitmap(), imageState, cropParameters, cropCallback).execute();
}
```
这块核心方法还是在`BitmapCropTask`中：
```java
//调整剪裁大小，如果有设置最大剪裁大小也会在这里做调整到设置范围
private float resize() {
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(mImageInputPath, options);

    boolean swapSides = mExifInfo.getExifDegrees() == 90 || mExifInfo.getExifDegrees() == 270;
    float scaleX = (swapSides ? options.outHeight : options.outWidth) / (float) mViewBitmap.getWidth();
    float scaleY = (swapSides ? options.outWidth : options.outHeight) / (float) mViewBitmap.getHeight();

    float resizeScale = Math.min(scaleX, scaleY);

    mCurrentScale /= resizeScale;

    resizeScale = 1;
    if (mMaxResultImageSizeX > 0 && mMaxResultImageSizeY > 0) {
        float cropWidth = mCropRect.width() / mCurrentScale;
        float cropHeight = mCropRect.height() / mCurrentScale;

        if (cropWidth > mMaxResultImageSizeX || cropHeight > mMaxResultImageSizeY) {

            scaleX = mMaxResultImageSizeX / cropWidth;
            scaleY = mMaxResultImageSizeY / cropHeight;
            resizeScale = Math.min(scaleX, scaleY);

            mCurrentScale /= resizeScale;
        }
    }
    return resizeScale;
}

// 剪裁图片
private boolean crop(float resizeScale) throws IOException {
    ExifInterface originalExif = new ExifInterface(mImageInputPath);

    //四舍五入取整
    cropOffsetX = Math.round((mCropRect.left - mCurrentImageRect.left) / mCurrentScale);
    cropOffsetY = Math.round((mCropRect.top - mCurrentImageRect.top) / mCurrentScale);
    mCroppedImageWidth = Math.round(mCropRect.width() / mCurrentScale);
    mCroppedImageHeight = Math.round(mCropRect.height() / mCurrentScale);

    //计算出图片是否需要被剪裁
    boolean shouldCrop = shouldCrop(mCroppedImageWidth, mCroppedImageHeight);
    Log.i(TAG, "Should crop: " + shouldCrop);

    if (shouldCrop) {
        //调用C++方法剪裁
        boolean cropped = cropCImg(mImageInputPath, mImageOutputPath,
                cropOffsetX, cropOffsetY, mCroppedImageWidth, mCroppedImageHeight,
                mCurrentAngle, resizeScale, mCompressFormat.ordinal(), mCompressQuality,
                mExifInfo.getExifDegrees(), mExifInfo.getExifTranslation());
        //剪裁成功复制图片EXIF信息
        if (cropped && mCompressFormat.equals(Bitmap.CompressFormat.JPEG)) {
            ImageHeaderParser.copyExif(originalExif, mCroppedImageWidth, mCroppedImageHeight, mImageOutputPath);
        }
        return cropped;
    } else {
        //直接复制图片到目标文件夹
        FileUtils.copyFile(mImageInputPath, mImageOutputPath);
        return false;
    }
}
```

### 3.3.3 GestureCropImageView
这个类主要就是对手势的监听，所以我们简单粗暴，直接找他的onTouchEvent方法：
```java
/**
 * 如果是ACTION_DOWN event，用户触摸屏幕，必须取消所有当前动画。
 * 如果是ACTION_UP event，用户从屏幕上取下所有手指，必须纠正当前图像位置。
 * 如果有两个以上的手指-更新焦点坐标。
 * 如果已启用，则将事件传递给手势检测器。
 */
@Override
public boolean onTouchEvent(MotionEvent event) {
    if ((event.getAction() & MotionEvent.ACTION_MASK) == MotionEvent.ACTION_DOWN) {
        cancelAllAnimations();
    }

    if (event.getPointerCount() > 1) {
        mMidPntX = (event.getX(0) + event.getX(1)) / 2;
        mMidPntY = (event.getY(0) + event.getY(1)) / 2;
    }

    //双击监听和拖动监听
    mGestureDetector.onTouchEvent(event);

    //两指缩放监听
    if (mIsScaleEnabled) {
        mScaleDetector.onTouchEvent(event);
    }

    //旋转监听
    if (mIsRotateEnabled) {
        mRotateDetector.onTouchEvent(event);
    }

    if ((event.getAction() & MotionEvent.ACTION_MASK) == MotionEvent.ACTION_UP) {
        //最后一指抬起时判断图片是否填充剪裁框
        setImageToWrapCropBounds();
    }
    return true;
}
```