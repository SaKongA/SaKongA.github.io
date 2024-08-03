---
title: 分离机械革命笔记本电脑的EXE格式BIOS固件，并修改Insyde BIOS的UEFI启动logo
date: 2024-08-01 22:49:23
tags:
---
## Before This:  
这段时间接了一个SpringBoot的后端外包订单，攒了点小钱，由于长期以来都是借舍友电脑写代码，出于各方面的原因，决定购入一台笔记本电脑，听取了AngelaCool大哥的建议，锁定了2023款的机械革命Code01 Ver2.0，R7-6800H，16G 4800MHZ DDR5，在海鲜市场2300元蹲到了在保的机器，到手之后也十分满意，后面看开机的“CODE”logo十分不顺眼，想起了以前入坑的台式机主板：华硕A320M，logo被我换成了技嘉的小雕（大概是初中时候的事情了），所以有了想要更换的想法，虽然没有办法做到ROG的动态logo+声音，但是一个logo也挺顺眼，哈哈~
> - 警告：刷写自定义BIOS为高风险操作，台式机中高端主板有BIOS回滚功能（如华硕可以使用`USB BIOS FlashBack™`实现刷BIOS），但大部分笔记本电脑只能通过拆机使用编程器重新刷写BIOS，请严格按照提示内容进行，本教程不保证各品牌机器通用，请三思后使用，出现的任何设备损坏的问题与作者无关！  
> - 备用方案：使用[`HackBGRT`](https://github.com/Metabolix/HackBGRT)也可以实现启动Logo修改，优点：相较于刷BIOS会安全很多，无法启动系统也可以使用PE修复；缺点：是Windows层面的修改，部分机器会出现开机先显示OEM Logo，在显示定制Logo的现象，无法根治。
## 1. 什么是 Insyde UEFI Bios？
在聊Insyde UEFI Bios之前不得不提一下AMI UEFI Bios，Ami UEFI BIOS 和 Insyde UEFI BIOS 是两种常见的UEFI（统一可扩展固件接口）固件实现，它们在现代计算机的启动过程和硬件初始化中起着关键作用。
### [AMI UEFI BIOS](https://www.ami.com/)
- 公司背景：AMI（American Megatrends International）是一家知名的计算机硬件固件开发公司。其 BIOS 和 UEFI 固件在业界广泛使用。  
- 特性：
  - 用户界面：通常提供直观的图形用户界面（GUI），使用户可以轻松调整系统设置。
  - 兼容性：支持广泛的硬件平台，兼容性良好。
  - 功能：包括自定义启动选项、系统监控、超频设置等。
  - 更新：AMI 定期发布固件更新，以支持新的硬件和修复已知问题。  
- 应用：常见于各种主板和计算机品牌中，尤其是在台式机和工作站中较为普遍。

### [Insyde UEFI BIOS](https://www.insyde.com/zh-hans)
- 公司背景：Insyde Software 是另一家主要的计算机固件提供商，专注于 BIOS 和 UEFI 固件的开发。
- 特性：
  - 用户界面：通常提供用户友好的图形界面，适合各种用户需求。
  - 功能：包括高级电源管理、安全启动、虚拟化支持等功能。
  - 可定制性：提供较强的定制能力，制造商可以根据需要对固件进行调整。
  - 更新：Insyde 也提供固件更新，以保证兼容性和功能的最新状态。
- 应用：常见于笔记本电脑和一些台式机中，尤其是在笔记本电脑市场中占有一定份额。  

## 2. 下载 BIOS 固件
- [机械革命 | 服务与支持](https://www.mechrevo.com/service/)  
Code01 Ver2.0 下载后是 BIOS+EC 合在一起的exe可执行文件，文件名称为:
```
BIOS_EC_UpdateTool(BIOS 0016.007.08.05 + EC 072).exe  
```
## 3. 分离 BIOS 固件
- 常见的BIOS固件格式有`.cap`、`.rom`、`.fd`、`.bin`，所以我们的目的就是从exe可执行文件中来获取这些格式的固件，解包exe可执行文件，最简单的方法就是使用7Zip打开，打开后为以下结构:
```
- BIOS_EC_UPDATETOOL(BIOS 0016.007.08.05 + EC 072)
│  .reloc
│  .text
│
└─.rsrc
    │  version.txt
    │
    ├─GROUP_ICON
    │      32512
    │
    ├─ICON
    │      1.ico
    │
    └─MANIFEST
            1
```
- 解压缩后没有发现明显的 BIOS 固件文件，查看文件目录后，`.rsrc`为可执行文件的运行资源，常理来说BIOS固件不会因为一个可执行文件的更新方式而改变封装格式，所以推断存在其他的资源文件中，决定通过文件大小判断 BIOS 固件所在位置，常见的BIOS芯片容量大小为 16MB / 32MB，压缩后大概在10MB-20MB，继续检查文件目录，发现`.text`文件十分可疑
```
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2046/9/25     15:18       14362112 .text
```
- 后面对该文件进行过更改后缀，但AMI/Insyde UEFI BIOS工具均无法识别，所以推断思路有误，再次通过7zip尝试打开文件，果然，解压缩得到了以下目录:
```
- .text
    BiosImageProcx64.dll
    Ding.wav
    FlsHook.exe
    FWUpdLcl.exe
    H2OFFT-Wx64.exe   <------
    H2OFFT.cat
    H2OFFT.inf
    H2OFFT64.sys
    InterToolx64.efi
    Lilac.FD   <------
    mfc90u.dll
    Microsoft.VC90.CRT.manifest
    Microsoft.VC90.MFC.manifest
    msvcp90.dll
    msvcr90.dll
    platform.ini

没有子文件夹
```
- 这边看到`H2OFFT`字样的文件，就可以直接判断该固件为Insyde的 BIOS 固件了，文件夹中的`H2OFFT-Wx64.exe`为Insyde UEFI BIOS升级程序，继续查看，发现`Lilac.FD`文件，为 BIOS 固件后缀，检查后发现大小也合适:
```
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2023/4/17      9:46       33554432 Lilac.FD
```
- 至此，我们获取到了FD格式的BIOS固件
## 4. 解包、替换、打包
- 导入固件：这边使用`InsydeH2O`来编辑Insyde UEFI BIOS固件，启动`H2OEZE-Win32.exe`，然后从左上角导入：
```
File -> Load BIOS image from... -> Lilac.FD
```
- 查看资源文件：在左侧资源栏中找到Logo文件：
```
Function -> Components -> Logo
```
- 在右侧的`Current internal logo`中找到目前的启动logo，并记下相应的文件格式与分辨率：
```
931F77D1-10FE-48BF-AB72-773D389E3FAA (JPG - 800 * 600)
```
- 那我需要准备的Logo文件格式为`JPG`,分辨率为`800*600`(这边建议自己转换格式和裁剪，虽然H2OEZE也会帮你转换裁剪，但是效果并不好)，准备好图像文件后，在工具右侧`Load image from...`中选择，点击下方`Apply`后，即可替换成功
- Tips：可以通过`AMI BIOS Logo Changer`工具来提取其他厂商AMI BIOS的启动Logo，工具见文末~
- 最后从左上角导出保存：
```
File -> Save as... -> Lilac-new.FD
```
- 在窗口左下角出现`Successful`后就可以关闭工具了
## 5. 刷入 UEFI BIOS 固件
- 这边我使用自带的`H2OFFT-Wx64.exe`来安装固件，该程序会自动读取同目录下的FD格式固件，所以在启动前确保程序目录下只存在一个即将刷入的FD格式固件，不要继续保留修改前的固件，修改后的文件目录如下：
```
- .text
    BiosImageProcx64.dll
    Ding.wav
    FlsHook.exe
    FWUpdLcl.exe
    H2OFFT-Wx64.exe
    H2OFFT.cat
    H2OFFT.inf
    H2OFFT64.sys
    InterToolx64.efi
    Lilac-new.FD   <------
    mfc90u.dll
    Microsoft.VC90.CRT.manifest
    Microsoft.VC90.MFC.manifest
    msvcp90.dll
    msvcr90.dll
    platform.ini

没有子文件夹
```
- 启动`H2OFFT-Wx64.exe`后，会弹出`Caution!`窗口，先不要着急按确定，看清楚提示内容后，确定好左侧的型号后，无误点击升级就可以了，后面升级完成后会自动重启电脑，如果电脑重启黑屏后长时间（约30秒）没有关机，可以长按关机键强制关机，然后开机
- 至此，刷写成功。

## Download Tools：
- [InsydeH2OEZE_03.04.zip](https://drive.google.com/file/d/1dsZIFeD5XgsKU3IWGQQ88_iETMrsU6UH/view?usp=sharing)
- [AMI_BIOS_Logo_Changer_v5.exe](https://drive.google.com/file/d/1i9SkB8tXG_SROGEbQ6K-KntlFUhLWlFe/view?usp=sharing)