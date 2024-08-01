---
title: 记一次简单的VUE+UniAPP逆向分析过程
date: 2024-07-05 13:29:10
tags:
---

## Before This:  
前些天朋友发给我了一个Android APP，让我看看能不能白嫖一下，不过以前没没有什么实操过应用逆向，所以也没有抱多大希望，抱着试一试的心态，来玩玩，结果到最后竟然成了，这也是我第一次接触VUE+UniApp的Android应用，因此写了这个帖子，来记录一下分析过程，不过难度很低，就当做涨知识了。  

## 1. 熟悉  
其实从报名这里就可以看出来了，“uni.xxxx”-典型的UniApp打包的Android客户端APK包名，不过由于我是第一次接触，一开始并未发觉有什么问题，然后就是APK未加固，可能会简单点；  
![0](images/images/0.jpg "0")
安装后打开，注册登录，发现所有的功能都提示“账户锁定或账户未激活”，查看后发现APP功能需要购买兑换码激活使用；  
![1](images/images/1.jpg "1")

## 2. 分析  
进行操作会检测账号状态，然后根据账号状态来返回相应结果，猜想可能是进行发送POST请求到服务器，来完成这个过程的，遂尝试抓包分析；  
![2](images/images/2.jpg "2")

利用“ProxyPin”进行抓包，这里我已经安装好ProxyPin的HTTPS证书，可以正常抓取HTTPS Packag，选择“更多-过滤-应用白名单”添加目标应用，然后开始抓包；  
![3](images/images/3.jpg "3")

幸运的是，APP没有做反抓包的相关机制，所以抓包很顺利，这里我们随便挑选一个功能，进行抓取请求，可以看到已经抓到验证的POST请求了；  
![4](images/images/4.jpg "4")

这里可以发现，APP竟然使用的是HTTP验证，查看抓包详情中的响应部分，这里是服务器返回的数据，没有加密，很清晰；  
![5](images/images/5.jpg "5")

其中code部分猜测是账号的激活状态，msg即message，返回的toast信息提示，time为时间戳，data为空值，由于我没并不知道code值为多少，账号是激活状态，所以这里猜一下，先试一下“1”；  

点击“重写”按钮，来拦截返回数据，并做修改，将code值改为“1”，保存，返回APP，测试，完美，所有需要验证的功能都可以使用了；  

那到这里，虽然我没没有办法确定code值具体的处理流程，但是至少值“1”可用，那接下来，转到APP代码部分的逆向分析，因为朋友那边不可能直接使用抓包重写。  

## 3. 静态逆向分析  
由于我这边第一次接触UniApp，一开始在这部分分析还是按照普通Java/Kotlin AndroidApp的结构分析的，所以掉坑里了，所以如果你对UniApp很熟悉的话，可以跳过这部分（不过如果你对UniApp很熟悉，应该也不会看我这个文章）；  
![6](images/images/6.jpg "6")

解包APP后，我的思路是从发送POST的URL入手，遂MultiDex之，直接搜索POST目标地址相关代码，当，没有，所以我怀疑是不是用变量代替了字符，于是使用Arsc编辑器搜索POST目标地址相关字符，当，还是没有，到这里我随后又搜索了相关点击事件的入口，结果还是没找到，也包括checkAccount、AccountManager，都没有发现什么有价值的线索，在一筹莫展的时候，发现这个assets资源文件夹貌似很特殊，Google一下，才发现这个APP是VUE+UniAPP的JavaScript套壳APP；  

于是直接找到关键Js:app-service.js，这个文件就是整个“小程序”的逻辑层，幸运的是，没有混淆加密，不过部分关键登录的地方，还是使用了Base64加密，打开自动换行，发现文字被转换为了Unicode字符串，不过这个对我们分析影响不大；  
![7](images/images/7.jpg "7")

在编译后，整个Js逻辑层代码被压缩了，为了更易读，我们使用相关工具(Notepad++、VScode或一些在线网站)格式化一下代码即可；  

现在，搜索“checkAccount”相关代码，55个结果，不过大部分代码逻辑都很相似，我们这边拿第一处为例：  
```javascript
getMessage: function(t) {                              // 定义参数`t`
    var e = this;                                      // 将`this`赋值到变量`e`   
    (0, a.checkAccount)({}).then((function(i) {        // 调用`checkAccount`函数，传入该函数中的对象`{}`,并赋值到`i`
        i.code = 1, 0 == i.code ? uni.showToast({      //  将`i`的`code`属性设置为1，然后判断`i`对象中的`code`值，如果为0，继续执行，如果不为0，跳过
            title: i.msg,                              //  赋值`i`中的`msg`信息到`tittle`
            icon: "none",                              //  icon为空
            duration: 2e3                              //  `duration`，时长，盲猜就是toast消息的通知时长
        }) : e.asheg(t.detail.data[0].action)          //  调用的`e`中的`asheg`方法，并向其中传入`t.detail.data[0].action`
    }))
},
```
到这里，账户状态验证的流程我们也就摸清了，checkAccount方法会将post的回复传到这里，然后判断code值，是否为0，后面我也测试了，只要回复中的code值不为0，都可以通过验证；  

一般这种情况我们直接将“0 == i.code”改为“1 == i.code”，理论上就可以通过验证，但是由于这个验证在逻辑层无处不在，如果55处全部更改，可能会出问题的，所以决定直接对“checkAccount”方法下手，很快就找到了关键方法：  
```javascript
e.checkAccount = function(t) {                         //  函数声明，赋值给`e`，接受`t`
    return (0, n.myRequest)({                          //  返回`(0, n.myRequest)`的结果,调用myRequest函数，用于发起网络请求
        url: "api/common/checkAccount",                //  请求URL
        method: "post",                                //  请求方法
        data: t                                        //  请求的数据，这里指传入的函数`t`
    })
}
```
既然checkAccount会发送POST请求来获取相关回复，那我们可以直接把ProxyPin中抓到的回复体扔给他，把回复喂到他嘴里，就不要在请求了，如下：  
```javascript
e.checkAccount = function(t) {                         
   return new Promise(function(resolve, reject) {
       var response = {
           code: 1,
           msg: "小飞棍来喽，这个msg其实已经没用了哈哈",
           time: "1720159579",                         // 通过对上面验证部分的分析，并不会验证时间戳，所以这里是什么也无所谓了
           data: null
       };
       resolve(response);
   });
};
 ```
## 4. 打包测试  
打包回APK，签名，由于签名冲突，需要卸载现在的APP，安装好后，幸运的是，连签名验证都没有，测试功能，完美。  
![8](images/images/8.jpg "8")

## Final:  
- 需要注意的问题  
在后面的测试中，发现软件的部分功能，需要将数据存储到服务器，这就导致了，APP开发者一觉醒来，发现竟然有没有认证的用户，在他服务器拉屎......（bushi，所以如果开发者真要想搞，也是很简单就能拉闸。  

- **文章仅供学习交流，一切非法行为与本人无关！**  

