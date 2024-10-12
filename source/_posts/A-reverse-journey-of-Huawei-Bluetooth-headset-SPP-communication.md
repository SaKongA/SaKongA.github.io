---
title: 一次华为蓝牙耳机SPP通讯的逆向之旅
date: 2024-09-17 20:01:49
updated: 2024-10-09 20:01:49
tags:
---

## Before This:
长久以来，饱受华为蓝牙耳机APP客户端的拉跨使用体验折磨（目前华为耳机的控制APP主要有两个，一个是已经断更许久的智慧音频，另一个是及其臃肿的智慧生活，我就买了一个耳机，你给我上全家生态？）在很久之前就有了重构一个控制APP的想法了，但是奈何实在没有时间，所以耽搁了。大二开学，实习也结束了，今天终于有时间来实现以下这个想法了。不过本篇文章只会讲到比较关键的地方，至于具体的实现方法，后续可能会有帖子讲到。


## 1. 从哪里入手？
我们想要实现一个耳机控制APP，最主要的两个目标如下：
- 向耳机发送数据，接收耳机的返回数据
- 解析耳机返回的数据，得到我们需要的数据形式  
  
在第一个目标中，由于每个厂商对于发送的数据格式的定义不同，所以我们如果想得到具体的控制数据内容，最简单的方法就是直接逆向现有的官方耳机控制APP，如“智慧音频”、“智慧生活”等，然后进行静态代码分析，找到APP中用来发送（或接收）蓝牙数据到设备的总方法，然后使用`Xposed`框架来`Hook`目标方法，从而获得对应数据内容，方便后续的模拟测试。  
  
在第二个目标中，相比发送数据到设备来实现对设备的控制，难度要高出一些，你可能需要逆向静态分析在APP接收到设备的数据后，数据的整个解析过程，例如，设备返回的电量数据会是一组数组，字面上来看是毫无意义的，需要查询APP的解析过程，然后进行模拟实现，来获取真正的数值。

## 2. 逆向哪一个APP ？
本篇文章中，将会对“智慧音频”APP进行逆向分析。
```
APP Name: 智慧音频
Package Nmae: com.huawei.audiogenesis
Version Code: 100011303
Package Size: 17.66M
Reinforce: None
```
- 为什么不选择“智慧生活”APP？  
我们这边把主要逆向“智慧音频”APP而不是“智慧生活”APP，一方面是“智慧生活”APP本身体量很大，涉及到的类和方法过于繁多，会严重干扰我们的方法确定和追溯；另一方面，在“智慧生活”APP中，耳机的控制功能是作为一个单独的耳机控制插件出现的，使用时需要单独下载，插件为APK格式，作为一个module嵌入到“智慧生活”主APP中，导致后面HOOK难度增高，其次，我们在研究时需要单独抓取网络下载GET包获取这个插件本体，而且浅浅的分析后，发现该插件和主APP间存在部分方法互相调用，对后面分析也不利。总的来说，还是为了方便。

## 3. SPP和BLE是什么？
![IMAGE](https://github.com/SaKongA/picx-images-hosting/raw/master/SPP-BLE-(2).2krwighv3i.webp)
在我们开始逆向工作前，我们不得不先来了解一下蓝牙模块的这两种通讯方式，由于通信方式上的差异，APP中向设备发送数据的实现方法也就截然不同，因此我们需要简单的了解下这两种通信方式。  
  
BLE（Bluetooth Low Energy，蓝牙低功耗）和 SPP（Serial Port Profile，串行端口协议）是两种不同的蓝牙通信方式，主要区别在于它们的设计目的、应用场景和工作方式。  

### 1. BLE（Bluetooth Low Energy）
- 简介：BLE 是一种低功耗的蓝牙技术，设计用于需要间歇性、小数据量传输的设备，例如传感器、可穿戴设备、智能家居设备等。
- 特点：
    - 低功耗：BLE 设备在闲置时消耗极低的能量，非常适合需要长时间运行的设备。
    - 低数据传输量：BLE 适合传输较小的数据块，通常用于状态信息的传输，如心率、温度等。
- 适用场景：健康监测设备（如心率监测器）、物联网设备（如各家的手环，手表）、跟踪器（果子的AirTag）、智能锁等。
- 通信方式：基于事件驱动和广播模式的通信，主要通过 GATT（Generic Attribute Profile）协议进行数据交互。  
  
### 2. SPP（Serial Port Profile）
- 简介：SPP 是经典蓝牙的一种协议，用来模拟有线串行通信接口，主要用于在两个设备之间传输大量连续的数据。
- 特点：
    - 高数据传输量：SPP 支持相对较高的数据传输速率，适用于需要连续、较大数据流的场景。
- 适用场景：传统的无线串行连接替代方案，如蓝牙耳机、无线打印机、无线POS机、PDA、蓝牙串口模块等。
- 通信方式：模拟传统串口的数据传输方式，通常是点对点的长连接，稳定且持续的传输数据。

### 3. 总结-两者的区别

| **特性** | **BLE** | **SPP** |
|----------|----------|----------|
| **功耗** | 低 | 较高 |
| **数据量** | 少 | 多 |
| **通信方式** | 事件驱动，广播模式 | 点对点，持续连接 |
| **适用场景** | 可穿戴设备，传感器 | 需要连续数据传输的设备，如蓝牙耳机 |
| **传输协议** | 操作系统提供的GATT接口 | 串口通讯 |

## 4. 逆向源码获取方法
在我们了解到这两种蓝牙通讯方式后，我们可以基本确定，蓝牙耳机是使用SPP协议与设备进行通讯，我们需要在“智慧音频”APP中找到传输数据的函数；  
我们这边先思考一下，发送与接收数据到蓝牙耳机的方法在哪里？  
在一般的蓝牙APP中，通常为了保证代码的简洁性与易读性，会实现一个总的数据传输方法，而我们想要获取设备间的通讯数据，就需要找到相应的数据传输方法；  
但是这里就遇到了一个问题，我们作为逆向者，并不知道这个总的数据发送方法具体的方法名称，有的时候可以通过猜测，来直接定位到，不过大部分情况（或者代码被混淆）下，难以直接确定；  
思考一下，实际上，APP为用户提供了操作页面，用户点击相应的功能按钮、开关，APP会调用这个按钮对应的后端处理的API接口，总的调用流程大概如下：  
![IMAGE](https://github.com/SaKongA/picx-images-hosting/raw/master/SENDDATA.8ojogyu2q9.webp)
所以为了精准的定位这个总的数据传输接口，我们可以从小的功能入手；  
使用Jadx-GUI反编译APK，来获取APP的反编译源码；  
### 1. 数据发送部分  
我们先从简单的例子入手，比如现在APP需要切换耳机的降噪模式（主动降噪一般简称ANC），我们在Jadx中搜索关键词`ANCmode`，位置选择“方法名”，搜索到的结果如下：  
![image](https://github.com/SaKongA/picx-images-hosting/raw/master/search-ANCmode.3godtz2mxa.webp)  
在搜索结果中，我们很明显的就发现了一个名为`MbbCmdApi`的类，而相对应存在的方法包均为`get ··· / set ···`，从中我们可以大胆地猜测，这个类就是我们找的不同功能的后端API接口，打开该类后，证实了这一点；  
我们这边取其中一个方法来进行分析：  
```java
// 定义方法，接收并传入参数
public void getANCModeAndLevel(IRspListener<ANCModeAndLevelInfo> iRspListener) {
    // 调用“getANCModeAndLevel”，传入相关数据
    getANCModeAndLevel(CURRENT_DEVICE, iRspListener);
}
```
此方法调用另一个重载的方法 `getANCModeAndLevel`，并传入 `CURRENT_DEVICE` 和 `iRspListener` 作为参数。可以理解为是为了获取当前设备的 ANC 模式和级别，并通过传入的监听器处理结果；  
继续分析其调用的方法 `getANCModeAndLevel`:
```java
// 定义方法，接收并传入参数
public void getANCModeAndLevel(String str, IRspListener<ANCModeAndLevelInfo> iRspListener) {
    // 输出日志内容
    LogUtils.i(TAG, "Get ANC mode and level");
    // 调用“sendData”，传入相关数据
    sendData(str, MbbAppLayer.getANCModeAndLevel(), FaqConstants.NO_SN, new d4(this, iRspListener));
}
```
此方法通过 sendData 方法向指定的设备发送请求，以获取 ANC 模式和级别，并通过 iRspListener 处理响应；  
我们来具体分析一下传入的数据内容：
- 调用 `sendData` 方法，传入四个参数：
  - `str`：传入的字符串参数。
  - `MbbAppLayer.getANCModeAndLevel()`：调用 `MbbAppLayer` 类中的 `getANCModeAndLevel()` 方法，可能返回与 ANC 模式和级别相关的数据。
  - `FaqConstants.NO_SN`：一个常量，通常用于表示序列号或其他标识符，具体取决于 `FaqConstants` 类的定义。
  - `new d4(this, iRspListener)`：创建一个新的 `d4` 对象，并将当前实例 (`this`) 和 `iRspListener` 作为参数传递。`d4` 可能是一个内部类或回调，用于处理数据发送后的响应。

继续分析其调用的方法 `sendData`：
```java
// 定义私有方法，接收并传入4个参数
private void sendData(String str, byte[] bArr, int i5, IResultListener iResultListener) {
    // 输出日志内容
    LogUtils.i(TAG, "sendData device mac is " + BluetoothUtils.convertMac(str));
    // 检查 BluetoothManager 实例的 AudioDeviceManager 是否为null
    if (BluetoothManager.getInstance().getAudioDeviceManager() == null) {
        // 检查 iResultListener 是否为null
        if (iResultListener != null) {
            // 调用 iResultListener 的 onFailed 方法，报错
            iResultListener.onFailed(ErrorCode.FAILED_MANGER);
            return;
        }
        return;
    }
    // 检查传入的设备标识 str 是否等于 CURRENT_DEVICE
    if (CURRENT_DEVICE.equals(str)) {
        // 获取当前设备的 MAC 地址，设置新的str值
        str = BluetoothManager.getInstance().getAudioDeviceManager().getCurrentMac();
    }
    // 从 AudioDeviceManager 获取与 str 相关联的设备实例，存储在 handlingDevice 变量中
    BaseDevice handlingDevice = BluetoothManager.getInstance().getAudioDeviceManager().getHandlingDevice(str);
    // 检查处理数据设备 handlingDevice 是否为null
    if (handlingDevice != null) {
        // 初始化 sendType 为 REGULAR，表示常规发送类型
        SendType sendType = SendType.REGULAR;
        // 检查 i5 是否等于 0 ?
        if (i5 == 0) {
            // 如果 iResultListener 为 null，则将 sendType 设置为 NO_LISTENER
            if (iResultListener == null) {
                sendType = SendType.NO_LISTENER;
            } else {
                // 如果 iResultListener 不为 null，将 sendType 设置为 SEPARATELY
                sendType = SendType.SEPARATELY;
            }
        }
        // 调用 handlingDevice 的 send 方法，传入相关数据
        handlingDevice.send(bArr, sendType, i5, iResultListener);
    }
}
```
在我们大致分析完该方法后，发现该方法为发送数据的关键方法，我们可以右键这个方法，点击“查找用例”，可以发现，基本上该Api类中的所有功能方法都调用了这个方法，我们可以基本确定，这个方法为关键的数据传输方法；    

### 2. 数据接收部分
上面我们成功获取到了数据发送的方法，现在我们来寻找数据接收的方法；  
前面在我们找到的数据发送部分中，这个方法所在的类似乎并不是该APP底层的数据通讯类，在APP的底层通讯类中，一般是同时包括发送与接收两个方法的，我们继续向下探索；  
```java
// 调用 handlingDevice 的 send 方法，传入相关数据
handlingDevice.send(bArr, sendType, i5, iResultListener);
```
在发送的方法中，调用了`send`方法，我们利用`Jadx`一路追溯下去,最终找到了`SPPManager`类，仔细观察这个类，从命名中也能看得出来是一个关键通讯类，在其中我们发现了我们需要的所有关键方法：`write`与`receive`。  
  
`receive`方法：
```java
// 定义公共方法，传入数组参数
public void receive(byte[] bArr) {
    // 调用方法，传入参数
    this.mDevice.onReceive(bArr);
}
```
`write`方法：  
```java
// 定义公共方法，传入数组参数
public boolean write(byte[] bArr) {
    OutputStream outputStream;
    // 检查状态是否为数据通道准备好，并且输出流不为null
    if (this.progress == SppStateMachine.DATA_CHANNEL_READY && (outputStream = this.mOutStream) != null) {
        try {
            // 将字节数组 bArr 写入到输出流
            outputStream.write(bArr);
            // 输出日志，记录成功发送的数据长度和内容
            LogUtils.d(TAG, "[" + BluetoothUtils.convertMac(this.mDevice.getDevice().getAddress()) + "] (Origin) Sent [Len]: " + bArr.length + " [Data]: " + r0.b(bArr));
            // 写入成功返回 true
            return true;
        // 捕捉IO错误
        } catch (IOException unused) {
            // 输出IO错误日志
            LogUtils.e(TAG, "[" + BluetoothUtils.convertMac(this.mDevice.getDevice().getAddress()) + "] (VirtualDevice Layer) Exception during write");
        }
    } else {
        // 输出数据流错误日志
        LogUtils.d(TAG, "[" + BluetoothUtils.convertMac(this.mDevice.getDevice().getAddress()) + "] (VirtualDevice Layer) Spp state machine not ready or OutStream not ready");
    }
    // 写入失败时返回 false
    return false;
}
```
至此，我们已经找到了我们所需要的所有关键方法；设备间通讯数据类型为数组，接下来，我们通过`Xposed Hook`来获取设备间的通讯数据。  
## 5. 构建Hook程序
在`AndroidStudio`中制作一个基本的`Hook`程序；
### 1. 引入Xposed依赖
在`settings.gradle.kts`中添加`Xposed`的`Maven`仓库
```kts
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // 添加 Xposed 的 Maven 仓库
        maven {
            url = uri("https://api.xposed.info/")
        }
    }
}
```
进入我们app目录下的`build.gradle.kts`引入`xposed`的依赖
```kts
dependencies {

    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.appcompat)
    implementation(libs.material)
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
    // 引入Xposed依赖
    compileOnly("de.robv.android.xposed:api:82")
}
```
###  2. 添加模块作用域（可选）
在`LSPosed`中，启用模块需要勾选相应的作用域；我们在`res/values`目录下创建一个名叫`arrays`的资源文件，添加作用域应用：
```xml
<resources>
    <string-array name="xposedscope" >
        <!-- 这里填写模块的作用域应用的包名 -->
        <item>com.huawei.audiogenesis</item>
    </string-array>
</resources>
```
### 3. 声明模块
我们想让LSPosed框架识别到此APP是一个Xposed模块，就需要进行声明；  
回到`AndroidManifest.xml`文件里，我们将`<application ... />`改成以下形式（注意，是改成！就是把结尾的`/>`换成`> </application>`），修改后的文件如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.SPPHOOKER"
        tools:targetApi="31" >
        <!-- 是否为Xposed模块 -->
        <meta-data
            android:name="xposedmodule"
            android:value="true"/>
        <!-- 模块的简介（在框架中显示） -->
        <meta-data
            android:name="xposeddescription"
            android:value="我是Xposed模块简介" />
        <!-- 模块最低支持的Api版本 一般填54即可 -->
        <meta-data
            android:name="xposedminversion"
            android:value="54"/>
        <!-- 模块作用域 -->
        <meta-data
            android:name="xposedscope"
            android:resource="@array/xposedscope"/>
    </application>

</manifest>
```
然后在`src/main`目录下创建一个文件夹名叫`assets`，并且创建一个文件叫`xposed_init`，注意，它没有后缀名！  
接着我们需要创建一个入口类，名叫`MainHook`（或者随便你想取什么名字都行）。   
创建好后回到我们的`xposed_init`里并用文本文件的方式打开它，输入我们刚刚创建的类的完整路径，如：
```
com.sakongapps.spp_hooker.MainHook
```

### 4. 编写模块
篇幅有限，我们在此不详细解释Xposed模块其中，该Hook类还是很简单的，可以结合注释加以理解。  
其中，示例的MainHook类如下：

```kotlin
class MainHook : IXposedHookLoadPackage {
    private val targetPackageName = "com.huawei.audiogenesis"

    // 当目标包加载时执行
    override fun handleLoadPackage(lpparam: LoadPackageParam) {
        if (lpparam.packageName != targetPackageName) return

        // 查找目标类 SppManager
        val targetClass = XposedHelpers.findClass(
            "com.huawei.audiobluetooth.layer.device.spp.SppManager", 
            lpparam.classLoader
        )

        // 针对 write 和 receive 方法进行 Hook
        listOf("write", "receive").forEach { methodName ->
            targetClass.methods.find { it.name == methodName }?.let { method ->
                XposedBridge.log("发现目标方法：$methodName!")
                XposedBridge.hookMethod(method, object : XC_MethodHook() {
                    override fun beforeHookedMethod(param: MethodHookParam) {
                        logData(methodName, param.args[0] as ByteArray)
                    }
                })
            }
        }
    }

    // 记录数据包信息的通用方法
    private fun logData(methodName: String, byteArray: ByteArray) {
        XposedBridge.log("${if (methodName == "write") "W-发送" else "R-接收"}数据包 [Len]: ${byteArray.size}, [Data]: ${byteArray.contentToString()}")
        XposedBridge.log("-------------------------------------------")
    }
}
```
在编写完成`MainHook`类后，我们前往 `Build - Build App Bundle(s) / APK(s) - Build APK(s)` 中进行构建APK，完成后右下角会弹出提示，点击Locate即可打开APK的所在文件夹，我们在手机上安装模块，并勾选对应的作用域。
![IMAGE](https://github.com/SaKongA/picx-images-hosting/raw/master/BuildAPK.pfbpt9w8e.webp)
## 6. 抓取通讯数据
我们将手机连接到电脑上，进行`Logcat`抓取，同时需要过滤关键词，然后在手机上启动“智慧音频APP”，即可看到相关的通讯数据了；
这里有一点需要注意，如果想使用`Linux`中的`grep`过滤命令，我们需要调用`Android Shell`中的`logcat`，而不是`Windows`中的`adb logcat`，命令如下:
```shell
> adb shell "logcat | grep LSPosed-Bridge"
```
抓取到的通讯信息如下：
```log
10-12 14:52:42.887 27980 27980 I LSPosed-Bridge:   Loading class com.sakongapps.spp_hooker.MainHook
10-12 14:52:42.941 27980 27980 I LSPosed-Bridge: 发现目标方法：receive !
10-12 14:52:42.941 27980 27980 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:42.942 27980 27980 I LSPosed-Bridge: 发现目标方法：write !
10-12 14:52:42.942 27980 27980 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:42.971 27980 27980 I LSPosed-Bridge: 发现目标方法：write!
10-12 14:52:42.971 27980 27980 I LSPosed-Bridge: 发现目标方法：receive!
10-12 14:52:43.652 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 8, [Data]: [90, 0, 3, 0, 1, 6, 62, -67]
10-12 14:52:43.652 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.652 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 8, [Data]: [90, 0, 3, 0, 1, 6, 62, -67]
10-12 14:52:43.652 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.654 27980 28277 I LSPosed-Bridge: W-发送数据包 [Len]: 18, [Data]: [90, 0, 13, 0, 1, 5, 1, 4, 103, 10, 28, -69, 2, 2, 8, 0, -118, 89]
10-12 14:52:43.655 27980 28277 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.655 27980 28277 I LSPosed-Bridge: W-发送数据包 [Len]: 18, [Data]: [90, 0, 13, 0, 1, 5, 1, 4, 103, 10, 28, -69, 2, 2, 8, 0, -118, 89]
10-12 14:52:43.655 27980 28277 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.655 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 11, [Data]: [90, 0, 6, 0, 43, 45, 1, 1, 2, -116, 90]
10-12 14:52:43.655 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.655 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 11, [Data]: [90, 0, 6, 0, 43, 45, 1, 1, 2, -116, 90]
10-12 14:52:43.655 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.681 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 11, [Data]: [90, 0, 6, 0, 43, 47, 1, 1, 1, 81, 81]
10-12 14:52:43.681 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.681 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 11, [Data]: [90, 0, 6, 0, 43, 47, 1, 1, 1, 81, 81]
10-12 14:52:43.681 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.705 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 25, [Data]: [90, 0, 20, 0, 1, 8, 1, 1, 1, 2, 3, 1, 45, 1, 3, 3, 0, 0, 0, 4, 2, 20, 10, 8, 79]
10-12 14:52:43.705 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.705 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 25, [Data]: [90, 0, 20, 0, 1, 8, 1, 1, 1, 2, 3, 1, 45, 1, 3, 3, 0, 0, 0, 4, 2, 20, 10, 8, 79]
10-12 14:52:43.705 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.723 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 146, [Data]: [90, 0, -115, 0, 1, 7, 2, 2, 1, 69, 3, 11, 72, 76, 49, 79, 84, 69, 77, 50, 95, 86, 65, 7, 32, 72, 97, 114, 109, 111, 110, 121, 79, 83, 32, 50, 46, 49, 46, 48, 46, 50, 48, 50, 40, 70, 48, 48, 49, 72, 48, 48, 51, 67, 48, 48, 41, 9, 16, 53, 80, 85, 84, 81, 50, 52, 50, 50, 54, 48, 48, 51, 50, 52, 54, 10, 15, 66, 84, 70, 84, 48, 48, 49, 52, 45, 48, 48, 48, 49, 52, 53, 15, 8, 66, 84, 70, 84, 48, 48, 49, 52, 24, 37, 76, 45, 84, 81, 57, 48, 56, 52, 50, 52, 50, 80, 48, 48, 57, 56, 52, 48, 44, 82, 45, 84, 81, 57, 48, 56, 56, 50, 52, 50, 80, 48, 49, 50, 52, 54, 48, 25, 1, 12, -107, 75]
10-12 14:52:43.723 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.723 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 146, [Data]: [90, 0, -115, 0, 1, 7, 2, 2, 1, 69, 3, 11, 72, 76, 49, 79, 84, 69, 77, 50, 95, 86, 65, 7, 32, 72, 97, 114, 109, 111, 110, 121, 79, 83, 32, 50, 46, 49, 46, 48, 46, 50, 48, 50, 40, 70, 48, 48, 49, 72, 48, 48, 51, 67, 48, 48, 41, 9, 16, 53, 80, 85, 84, 81, 50, 52, 50, 50, 54, 48, 48, 51, 50, 52, 54, 10, 15, 66, 84, 70, 84, 48, 48, 49, 52, 45, 48, 48, 48, 49, 52, 53, 15, 8, 66, 84, 70, 84, 48, 48, 49, 52, 24, 37, 76, 45, 84, 81, 57, 48, 56, 52, 50, 52, 50, 80, 48, 48, 57, 56, 52, 48, 44, 82, 45, 84, 81, 57, 48, 56, 56, 50, 52, 50, 80, 48, 49, 50, 52, 54, 48, 25, 1, 12, -107, 75]
10-12 14:52:43.723 27980 28230 I LSPosed-Bridge: -------------------------------------------
10-12 14:52:43.724 27980 28230 I LSPosed-Bridge: R-接收数据包 [Len]: 49, [Data]: [90, 0, 44, 0, 43, 49, 2, 1, 6, 3, 1, 5, 4, 6, 28, -75, -91, -6, -60, -8, 5, 1, 9, 6, 1, 1, 7, 1, 0, 8, 1, 1, 9, 13, -28, -72, -128, -27, -118, -96, 32, 65, 99, 101, 32, 51, 86, 66, -84]
10-12 14:52:43.724 27980 28230 I LSPosed-Bridge: -------------------------------------------
```
## 7. Final:
在我们可以抓取到通讯数据后，我们就可以通过使用不同功能来捕捉对应通讯数据，这样可以更加方便地定位我们需求功能的通讯命令，但是如果需要更加完整的通讯命令清单，或者后面需要实现对返回数据的解析，就需要继续逆向追溯返回数据的解析部分，后面有时间的话，会继续出相关的文章；  
最后感谢您的耐心阅读！
## 8. Thanks:
- 本文部分引用/借鉴了以下文章，对作者表示感谢：
  - [低功耗蓝牙(BLE) 和 经典蓝牙(SPP) 的区别](https://zhuanlan.zhihu.com/p/680604882)
  - [Xposed模块开发入门保姆级教程](https://blog.ketal.icu/cn/Xposed%E6%A8%A1%E5%9D%97%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8%E4%BF%9D%E5%A7%86%E7%BA%A7%E6%95%99%E7%A8%8B/)