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
  
在第一个目标中，由于每个厂商对于发送的数据格式的定义不同，所以我们如果想得到具体的控制数据内容，最简单的方法就是直接逆向现有的官方耳机控制APP，如“智慧音频”、“智慧生活”等，然后进行静态代码分析，找到APP中用来发送（或接收）蓝牙数据到设备的总方法，然后使用Xposed框架来Hook目标方法，从而获得对应数据内容，方便后续的模拟测试。  
  
在第二个目标中，相比发送数据到设备来实现对设备的控制，难度要高出一些，你可能需要逆向静态分析在APP接收到设备的数据后，数据的整个解析过程，例如，设备返回的电量数据会是一组数组，字面上来看是毫无意义的，需要查询APP的解析过程，然后进行模拟实现，来获取真正的数值。

## 2. 智慧音频APP ？
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
![IMAGE](https://github.com/SaKongA/picx-images-hosting/raw/master/sppble.8s3aeohkw4.webp)
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
我们这边先思考一下，发送数据到蓝牙耳机的方法在哪里？  
在一般的蓝牙APP中，通常为了保证代码的简洁性与易读性，会实现一个总的数据传输方法，而我们想要获取设备间的通讯数据，就需要找到相应的数据传输方法；  
但是这里就遇到了一个问题，我们作为逆向者，并不知道这个总的数据发送方法具体的方法名称，有的时候可以通过猜测，来直接定位到，不过大部分情况（或者代码被混淆）下，难以直接确定；  
思考一下，实际上，APP为用户提供了操作页面，用户点击相应的功能按钮、开关，APP会调用这个按钮对应的后端处理的API接口，总的调用流程大概如下：  
![IMAGE](https://github.com/SaKongA/picx-images-hosting/raw/master/SENDDATA.8ojogyu2q9.webp)
所以为了精准的定位这个总的数据传输接口，我们可以从小的功能入手；  
使用Jadx-GUI反编译APK，来获取APP的反编译源码；  
我们先从简单的例子入手，比如现在APP需要切换耳机的降噪模式（主动降噪一般简称ANC），我们在Jadx中搜索关键词“ANCmode”，位置选择“方法名”，搜索到的结果如下：  
![image](https://github.com/SaKongA/picx-images-hosting/raw/master/search-ANCmode.3godtz2mxa.webp)  
在搜索结果中，我们很明显的就发现了一个名为“MbbCmdApi”的类，而相对应存在的方法包均为“get ···”/“set ···”，从中我们可以大胆地猜测，这个类就是我们找的不同功能的后端API接口，打开该类后，证实了这一点；  
我们这边取其中一个方法来进行分析：  
```java
// 定义方法，接收并传入参数
public void getANCModeAndLevel(IRspListener<ANCModeAndLevelInfo> iRspListener) {
    // 调用“getANCModeAndLevel”，传入相关数据
    getANCModeAndLevel(CURRENT_DEVICE, iRspListener);
}
```
此方法调用另一个重载的方法 getANCModeAndLevel，并传入 CURRENT_DEVICE 和 iRspListener 作为参数。可以理解为是为了获取当前设备的 ANC 模式和级别，并通过传入的监听器处理结果；  
继续分析其调用的方法 getANCModeAndLevel:
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
- 调用 sendData 方法，传入四个参数：
  - str：传入的字符串参数。
  - MbbAppLayer.getANCModeAndLevel()：调用 MbbAppLayer 类中的 getANCModeAndLevel() 方法，可能返回与 ANC 模式和级别相关的数据。
  - FaqConstants.NO_SN：一个常量，通常用于表示序列号或其他标识符，具体取决于 FaqConstants 类的定义。
  - new d4(this, iRspListener)：创建一个新的 d4 对象，并将当前实例 (this) 和 iRspListener 作为参数传递。d4 可能是一个内部类或回调，用于处理数据发送后的响应。

继续分析其调用的方法 sendData：
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
其实分析到这里的时候，我们就可以直接进行HOOK，来获取传输的具体数据了，如果感兴趣的话，可以继续向下分析数据发送的具体过程；
## 5. 构建Hook程序
在AndroidStudio中制作一个基本的Hook程序，MainHook类如下

```kotlin
class MainHook : IXposedHookLoadPackage {
    var TAG: String = "SPP-HOOKER"
    private val mStrPackageName = "com.huawei.audiogenesis" //HOOK APP目标的包名

    override fun handleLoadPackage(loadPackageParam: LoadPackageParam) {
        //判断包名是否一致
        if (loadPackageParam.packageName == mStrPackageName) {
            val targetClass = XposedHelpers.findClass(
                "com.huawei.audiobluetooth.layer.device.spp.SppManager",
                loadPackageParam.classLoader
            )
            targetClass.methods.forEach {
                if (it.name == "write") {
                    XposedBridge.log("发现目标方法：" + "write" + " !")
                    XposedBridge.log("-------------------------------------------")
                    XposedBridge.hookMethod(it, object : XC_MethodHook() {
                        override fun beforeHookedMethod(param: MethodHookParam) {
                            onPreInvokeW(param)
                        }
                    })
                    return@forEach
                }
                if (it.name == "receive") {
                    XposedBridge.log("发现目标方法：" + "receive" + " !")
                    XposedBridge.log("-------------------------------------------")
                    XposedBridge.hookMethod(it, object : XC_MethodHook() {
                        override fun beforeHookedMethod(param: MethodHookParam) {
                            onPreInvokeR(param)
                        }
                    })
                    return@forEach
                }
            }
        }
    }
}

fun onPreInvokeW(param: MethodHookParam) {
    val byteArray1 = param.args[0] as ByteArray
    XposedBridge.log("W-发送数据包 [Len]: ${byteArray1.size}, [Data]: ${byteArray1.contentToString()}")
    XposedBridge.log("-------------------------------------------")
}

fun onPreInvokeR(param: MethodHookParam) {
    val byteArray2 = param.args[0] as ByteArray
    XposedBridge.log("R-接收数据包 [Len]: ${byteArray2.size}, [Data]: ${byteArray2.contentToString()}")
    XposedBridge.log("-------------------------------------------")
}
```