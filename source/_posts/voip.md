---
title: iOS利用voip push实现类似微信(QQ)电话连续响铃效果
date: 2016-06-20 17:04:10
tags:
    - iOS
    - pushkit
    - voip Push
    - 激活应用
---
由于项目中添加了视频语音呼叫功能，某一天老总说要做保活，并拿出手机演示微信的音视频呼叫功能，微信在APP被杀死和黑屏的情况下都能收到呼叫并且能连续响铃和振动，说是要达到这种效果才行。<!-- more -->当时脑子第一个想法是APNs去实现，但是APNs根本实现不了连续通知，就算连续推送也不可能只显示一个通知栏，然后又想到本地通知是可以实现连续响铃效果，但是当APP被杀死或者至于后台根本没办法启动本地通知，所以这个问题的切入点就是APP死了，怎么去激活？后面通过翻墙搜索发现VoIP可以实现激活应用，当时国内几乎搜索不到关于VoIP 推送的介绍，只好对着开发手册去慢慢摸索了。好了，直入正题了。
	
### 1、关于voip
  VoIP 推送基于pushKit框架。以前应用保活是通过与服务器发送周期性的心跳来维持，这样会导致设备被频繁唤醒，消耗电量，类似现在很多android系统的保活。为了代替永久连接机制，开发人员应该使用Pushkit框架。使用PushKit接收VoIP推送有很多好处：
 ``` bash
只有当VoIP发生推送时，设备才会唤醒，从而节省能源。
VoIP推送被认为是高优先级通知，并且毫无延迟地传送。
VoIP推送可以包括比标准推送通知提供的数据更多的数据。
如果收到VoIP推送时，您的应用程序未运行，则会自动重新启动。
即使您的应用在后台运行，您的应用也会在运行时处理推送。
```
最后两点正是我们想要的。另外很多人都认为APNs可以激活应用，经过我多种测试，APNS只能在应用没被杀死的情况下(没有双击划掉)才能激活应用，比如静默推送。但是当app被杀死的情况下，APNs不管怎么更换推送payload的格式，都无法激活应用，要是能也不至于多此一举弄个VoIP推送了。还有很多人担心自己的APP使用VOIP推送会被拒，官方上解释说，必须是VoIP应用才能集成，经过实践证明其实不一定要是VoIP应用，只要你的应用中有类似视频呼叫或者语音呼叫的功能都能通过审核，后面我也会提到怎么样才能通过审核，要是没有相关功能就不用去考虑了。

### 2、VoIP的集成（最后附上详细代码提供下载）
  在Xcode中开启VoIP推送
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_setting.png)
  在Apple Developer创建VoIP证书
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_center.png)
跟APNs证书不同，VoIP证书不区分开发和生产环境，VoIP证书只有一个，生产和开发都可用同一个证书。另外有自己做过推送的应该都知道服务器一般集成的.pem格式的证书，所以还需将证书转成.pem格式，后面会介绍怎么转换.pem证书。
导入framework
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_pushkit.jpg)
Objective-C代码集成，导入头文件
```
#import <PushKit/PushKit.h>
```
设置代理
```
PKPushRegistry *pushRegistry = [[PKPushRegistry alloc] initWithQueue:dispatch_get_main_queue()];
pushRegistry.delegate = self;
pushRegistry.desiredPushTypes = [NSSet setWithObject:PKPushTypeVoIP];
```
代理方法
```
//应用启动此代理方法会返回设备Token 、一般在此将token上传服务器
- (void)pushRegistry:(PKPushRegistry *)registry didUpdatePushCredentials:(PKPushCredentials *)credentials forType:(NSString *)type{
}
```
```
//当VoIP推送过来会调用此方法，一般在这里调起本地通知实现连续响铃、接收视频呼叫请求等操作
- (void)pushRegistry:(PKPushRegistry *)registry didReceiveIncomingPushWithPayload:(PKPushPayload *)payload forType:(NSString *)type {
    
}
```

### 3. 建立本地测试环境
集成VoIP推送看似是简单，但其中会遇到各种坑，如果每次出现问题都要服务器配合测试的话，很难找到问题的根源，而且还浪费时间，所以最好是自己搭建一个简单的测试环境先测试通，然后再与服务器对接。
 
   1、制作.pem格式证书
 ```
    1、将之前生成的voip.cer SSL证书双击导入钥匙串
    2、打开钥匙串访问，在证书中找到对应voip.cer生成的证书，右键导出并选择.p12格式,这里我们命名为voippush.p12,这里导出需要输入密码(随意输入，别忘记了)。
    3、目前我们有两个文件，voip.cer SSL证书和voippush.p12私钥，新建文件夹命名为VoIP、并保存两个文件到VoIP文件夹。
    4、把.cer的SSL证书转换为.pem文件，打开终端命令行cd到VoIP文件夹、执行以下命令
    openssl x509 -in voip.cer  -inform der -out VoiPCert.pem
    5、把.p12私钥转换成.pem文件，执行以下命令（这里需要输入之前导出设置的密码）
    openssl pkcs12 -nocerts -out VoIPKey.pem -in voippush.p12
    6、再把生成的两个.pem整合到一个.pem文件中
    cat VoiPCert.pem VoIPKey.pem > ck.pem
    最终生成的ck.pem文件一般就是服务器用来推送的。
  ```
   
 2、下载服务端推送代码并修改配置信息
    链接: [demo链接，里面包含.php代码](https://github.com/oopsr/VoIPPush.git)
``` bash
    下载后一并放入VoIP文件夹中,并配置好相关信息，注意下面图片中提到的推送环境，请填写对应的环境地址，一般测试VoIP推送的稳定性最好是通过Hoc证书打包在生产环境中测试。
开发环境地址：gateway.sandbox.push.apple.com:2195 
生产环境地址： gateway.push.apple.com:2195
 ```    
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_php.png)
3、推送测试
    用终端命令行cd到我们的VoIP文件夹中，输入： php  php文件名.php 
    就可以收到推送了，haha~。
VoIP文件夹
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_pem.jpg)   
推送终端命令
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_push.jpg)   
最终效果图
![](https://raw.githubusercontent.com/oopsr/oopsr.github.io/master/img/voip_huahua.png)   
###   4、补充

1、当app要上传App Store时，请在iTunes connect上传页面右下角备注中填写你用到VoIP推送的原因，附加上音视频呼叫用到VoIP推送功能的demo演示链接，演示demo必须提供呼出和呼入功能，demo我一般上传到优酷。
2、经过大量测试，VoIP当应用被杀死（双击划掉）并且黑屏大部分情况都能收到推送，很小的情况会收不到推送消息，经测试可能跟手机电量消耗还有信号强弱有关。 再强调一遍，测试稳定性请在生产环境测试。
3、如果不足和错误的地方，欢迎补充和改正，谢谢。

  链接: [推送demo下载](https://github.com/oopsr/VoIPPush.git)

