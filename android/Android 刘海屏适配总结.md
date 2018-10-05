# Android 刘海屏适配总结

### 一、简介
随着 Apple 发布 iPhone X 之后，各大手机厂商也开始模仿这种刘海屏的设计，而且刘海屏手机的用户也是越来越大，前段时间将项目进行了所有主流厂商的刘海屏手机的适配，以便让刘海屏手机的用户也能有更好的体验。

### 二、刘海屏造成的 UI 显示问题
刘海屏手机因为比平常的手机多了一块顶部的遮挡性刘海，所以会造成顶部 Toolbar 以及搜索框的遮挡，而且有些厂商的手机（vivo、华为），默认是在「无状态栏」的界面将状态栏进行黑化显示，这时候会导致系统下移，从而导致底部的一些 UI 被截断。除此之外，一些控件的显示规则还会受到影响，如 PopupWindow 的显示高度会在「无状态栏」的界面中比普通手机低一个「刘海的高度」，从而遮挡住原先在 PopupWindow 周围的图标。

##### 1、系统下移造成的底部 UI 截断
![小说页码被截断](https://upload-images.jianshu.io/upload_images/4334738-99a72b894593fe55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 2、刘海挡住标题栏和搜索框
![刘海挡住标题栏和搜索框](https://upload-images.jianshu.io/upload_images/4334738-9e79e5ac12a442ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 3、PopupWindow 显示异常
![PopupWindow 显示异常](https://upload-images.jianshu.io/upload_images/4334738-dfdeb3046c595a00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 三、通用的适配方案
理论上来讲，通过 Android P 版本提供的刘海屏相关接口，判断手机是否为刘海屏手机，以及进行一些相应的处理是最合适的方式，但现在在国内使用 Android P 的接口是不现实的，所以只能通过各大厂商提供的技术文档来进行适配，但适配的流程基本是一致的。

##### 刘海屏的适配流程
![刘海屏的适配流程](https://upload-images.jianshu.io/upload_images/4334738-09e2f12aadeb337a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**其中需要着重处理的是：**
- 1、应用是否已经适配刘海屏
- 2、页面是否显示状态栏

##### 3.1 应用是否已经适配刘海屏
现在国内的主流机型（华为、vivo、OPPO、小米）在刘海屏的显示上分为两个阵营：

- 当不显示状态栏时，直接将界面进行显示，「状态栏原先的位置也用于显示界面」，例如：OPPO 
- 当不显示状态栏时，直接「将状态栏原先的位置进行黑化，界面整体下移」，例如：华为、vivo

所以，我们在进行刘海屏适配的时候，首先需要通过一些手段，统一各大厂商的显示方案，让所有的刘海屏手机都利用状态栏的界面，「告知系统」我们已经适配了刘海屏，确保系统不会下移我们的应用，保留原生体验。

这里主要有两种方式：

**1、设置屏幕高宽比例**
因为刘海屏手机的「宽高比」比之前的手机大，如果不适配的话，Android 默认为最大的宽高比为 1.86， 小于刘海屏手机的宽高比，因此我们需要申明更高的宽高比来告诉系统，我们应用已经适配了刘海屏。

只要在 AndroidManifest.xml 中加入如下配置：
```
<meta-data android:name="android.max_aspect"  android:value="2.1"/>
```

也可以在 Application 添加属性：
```
android:maxAspectRatio="ratio_float"
```
ps：这个属性需要 API 26 才支持

**2、设置应用支持 resize**
我们还可以通过设置应用支持 resizeable，来告诉系统我们适配了刘海屏，而且这也是 Google 官方推荐的方式。不过需要注意的是，使用这个属性之后，应用也会跟着支持分屏模式。只需要在 AndroidManifest.xml 中添加：
```
android:resizeableActivity="true"
```
##### 3.2 页面是否显示状态栏

对于刘海屏适配，我们将界面分为两种：
- 对于有状态栏的界面，不会受到刘海屏的影响
- 全屏显示的界面（无状态栏），需要根据界面的显示进行一些控件的下移

因此，我们进行刘海屏适配，其实针对的就是没有状态栏的界面，而有状态栏的界面显示是正常的。对于没有状态栏的界面，主要是将对被刘海遮挡到的控件，设置对应刘海高度的 MarginTop，从而避免控件被遮挡。而对于底部可能被截断的界面，可以考虑将底部做成 ScrollView 的形式。

### 四、各厂商的适配方案
现在 Android P 的接口还没法用，但各手机厂商都制定了自己的 API，对此我们需要对各大机型进行特殊的适配，这里主要介绍 vivo、OPPO、华为 这三种主流手机的适配方案。

#### 华为
华为作为国内的手机厂商大头，自己仿照 Android P 提供的 API，实现了一套几乎差不多的 API，所以我们如果想要告诉系统我们的应用适配了刘海屏，最好直接使用华为的 API，这样才是最保险的。

以下代码来自：[华为刘海屏适配官方技术指导](https://mini.eastday.com/bdmip/180411011257629.html)

##### 1、应用页面设置使用刘海区显示

① 方案一：在 AndroidManifest.xml 中增加 meta-data 属性，此属性不仅可以针对 Application 生效，也可以对 Activity 配置生效：
```
<meta-data android:name="android.notch_support" android:value="true"/>
```
增加这个属性之后，系统就不会对应用进行下移处理，从而保证原生体验。

② 方案二：通过添加窗口 FLAG 的方式设置界面使用刘海区：
```
 public static void setFullScreenWindowLayoutInDisplayCutout(Window window) {
     if (window == null) {
         return;
     }
     WindowManager.LayoutParams layoutParams = window.getAttributes();
     try {
         Class layoutParamsExCls = Class.forName("com.huawei.android.view.LayoutParamsEx");
         Constructor con = layoutParamsExCls.getConstructor(LayoutParams.class);
         Object layoutParamsExObj = con.newInstance(layoutParams);
         Method method = layoutParamsExCls.getMethod("addHwFlags", int.class);
         method.invoke(layoutParamsExObj, FLAG_NOTCH_SUPPORT);
     } catch (ClassNotFoundException | NoSuchMethodException | IllegalAccessException |InstantiationException 
     | InvocationTargetException e) {
         Log.e("test", "hw add notch screen flag api error");
     } catch (Exception e) {
         Log.e("test", "other Exception");
     }
 }
```

##### 2、判断该华为手机是否刘海屏
```
    public static boolean hasNotchInHuawei(Context context) {
        boolean hasNotch = false;
        try {
            ClassLoader cl = context.getClassLoader();
            Class HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
            Method hasNotchInScreen = HwNotchSizeUtil.getMethod("hasNotchInScreen");
            if(hasNotchInScreen != null) {
                hasNotch = (boolean) hasNotchInScreen.invoke(HwNotchSizeUtil);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return hasNotch;
    }
```

##### 3、获取刘海的高度
```
     public static int[] getNotchSize(Context context) {
         int[] ret = new int[]{0, 0};
         try {
             ClassLoader cl = context.getClassLoader();
             Class HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
             Method get = HwNotchSizeUtil.getMethod("getNotchSize");
             ret = (int[]) get.invoke(HwNotchSizeUtil);
         } catch (ClassNotFoundException e) {
             Log.e("test", "getNotchSize ClassNotFoundException");
         } catch (NoSuchMethodException e) {
             Log.e("test", "getNotchSize NoSuchMethodException");
         } catch (Exception e) {
             Log.e("test", "getNotchSize Exception");
         } finally {
             return ret;
         }
```

#### OPPO
OPPO 是主流厂商中的一股清流，学 iPhoneX 是最像的，OPPO 手机对于不显示状态栏的界面，采取的是「状态栏原先的位置也用于显示界面」的方案，所以我们只要进行相关控件的位置移动就可以了。

以下代码来自： [OPPO 凹形屏适配说明](https://open.oppomobile.com/wiki/doc#id=10159)

##### 1、判断该 OPPO 手机是否为刘海屏手机
```
    public static boolean hasNotchInOppo(Context context) {
        return context.getPackageManager().hasSystemFeature("com.oppo.feature.screen.heteromorphism");
    }
```

##### 2、获取刘海屏的高度
对于 OPPO 刘海屏手机的刘海高度，OPPO 官方的文档没有提供相关的 API，但官方文档表示 OPPO 手机的刘海高度和状态栏的高度是一致的，而且我也对此进行了验证，确实如此。所以我们可以直接获取状态栏的高度，作为 OPPO 手机的刘海高度。

```
public static int getStatusBarHeight(Context context) {
    int statusBarHeight = 0;
    int resourceId = context.getResources().getIdentifier("status_bar_height", "dimen", "android");
    if (resourceId > 0) {
        statusBarHeight = context.getResources().getDimensionPixelSize(resourceId);
    }
    return statusBarHeight ;
}
```

#### vivo
vivo 提供的技术文档对于开发者来说是最不友好的，只提供了一个 API 来进行刘海屏的判断，并没有提供刘海高度的获取方式，我们只能通过获取状态栏高度来当做刘海的高度，但在某些机型可能会有些偏差。

官方文档：[vivo 手机适配指南](https://dev.vivo.com.cn/doc/document/info?id=103)


##### 判断该 vivo 手机是否为刘海屏手机
```
    public static boolean hasNotchInVivo(Context context) {
        boolean hasNotch = false;
        try {
            ClassLoader cl = context.getClassLoader();
            Class ftFeature = cl.loadClass("android.util.FtFeature");
            Method[] methods = ftFeature.getDeclaredMethods();
            if (methods != null) {
                for (int i = 0; i < methods.length; i++) {
                    Method method = methods[i];
                    if(method != null) {
                        if (method.getName().equalsIgnoreCase("isFeatureSupport")) {
                            hasNotch = (boolean) method.invoke(ftFeature, 0x00000020);
                            break;
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
            hasNotch = false;
        }
        return hasNotch;
    }
```





















