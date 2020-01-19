---
layout: post
title: iOS接入CallKit框架
---

![Image_1](/media/image/ios_callkit_1.png)

## 一、背景 & CallKit简介
公司针对国外市场，新成立了一个VoIP类型的项目---OFree，能与iOS的CallKit很好的结合，在这里记录一下接入CallKit的过程。

苹果在WWDC 2016发布了iOS 10的新框架CallKit，它允许开发者在VoIP类型APP整合系统原生语音界面，以获得更好的用户体验。接入CallKit后，APP里面的通话会被写入系统通话记录，而且APP通话时的权限比一般VoIP的APP权限要高。
*****
***敏感代码已删除或做混淆处理。***
*****
## 二、原理
### 1、呼入
#### 来电
![Image_2](/media/image/ios_callkit_2.png)

1.OFree接收到服务器的来电信息（后台时VoIP Push）；
2.创建CXCallUpdate对象记录来电信息，并记录该通话唯一标识；
3.通过CXProvider对象把来电信息通知系统；
4.系统接收到来电信息后显示原生来电UI。

#### 响应来电
![Image_3](/media/image/ios_callkit_3.png)

1.用户在原生来电UI上点击接听按钮（或挂断按钮）；
2.系统把动作封装成CXAnswerCallAction，通过CXProvider的Delegate回调到OFree；
3.OFree收到回调后会开始配置mediasdk，调整UI等操作，最后开始通话。

#### 结束来电
![Image_4](/media/image/ios_callkit_4.png)

1.OFree在前台时，用户点击挂断按钮；
2.创建CXTransaction对象，用于包装挂断动作CXEndCallAction（包含该通话唯一标识）；
3.通过CXCallController对象把挂断动作通知系统；
4.系统成功挂断该通话后，通过CXProvider的Delegate回调到OFree；
5.OFree收到回调后会开始释放mediasdk（这里要注意释放顺序，不能在系统回调前把sdk释放掉），调整UI等操作。

### 2、呼出
![Image_5](/media/image/ios_callkit_5.png)

1.用户在OFree呼出电话；
2.创建CXTransaction对象，用于包装呼出动作CXStartCallAction（包含该通话唯一标识、呼出号码等信息）；
3.通过CXCallController对象把呼出动作通知系统；
4.系统成功呼出该通话后，通过CXProvider的Delegate回调到OFree；
5.OFree收到回调后会开始配置mediasdk，调整UI等操作，最后开始通话。

## 三、OFree如何接入
### 1、mediasdk做兼容
1.系统资源申请成功后再配置mediasdk；
2.等系统资源释放后再释放mediasdk。

为满足以上两点，mediasdk开放了下面几个接口，在CallKit模式下，等系统回调后再做mediasdk配置、激活、释放等操作  

```
// In CallKit, it should be called by `perform Answer/start CallAction`
bool configAudio();

// In CallKit, it should be Called by `didActivateAudioSession`
bool startAudio();

// In CallKit, it should be Called by `didDeactivateAudioSession`
void stopAudio();

// In CallKit, it should be Called by `performEndCallAction`
void releaseAudio();

```
### 2、OFree在前台状态

1.在OFree项目中添加`ProviderDelegate`对象，作为与CallKit通信的桥梁；

2.OFree在前台，并且linkd正常连接，`CallController`负责监听来电拓传；

3.接收到来电拓传后，初始化mediasdk，并通过`ProviderDelegate`对象通知系统，触发系统来电页面，并监听系统回调；

4.系统回调`- provider:performAnswerCallAction:`时，调整OFree的来电UI；

5.系统回调`- provider:didActivateAudioSession:`时，配置mediasdk音频参数`configAudio()`，并激活音频通信`startAudio()`；

6.用户点击挂断或者对方点击挂断，通过`ProviderDelegate`对象通知系统通话结束，并监听系统回调；

7.系统回调`- provider:performEndCallAction:`后，释放mediasdk音频资源`releaseAudio()`，释放mediasdk；

调用时序如下：

```
// 来电
2018-05-10 20:05:58.640136+0800 BigoOfree[19394:8610838] [I][CallKit] handleIncomingCall 
2018-05-10 20:05:58.725955+0800 BigoOfree[19394:8610838] [I][CallKit] unprepareSDK
2018-05-10 20:05:58.726996+0800 BigoOfree[19394:8610627] [V][CallKit] prepareSDK
2018-05-10 20:06:02.327572+0800 BigoOfree[19394:8610627] [I][CallKit] performAnswerCallAction
2018-05-10 20:06:02.892451+0800 BigoOfree[19394:8610627] [I][CallKit] didActivateAudioSession
2018-05-10 20:06:03.087589+0800 BigoOfree[19394:8610628] [I][CallKit] configAudio
2018-05-10 20:06:03.240767+0800 BigoOfree[19394:8610627] [I][CallKit] startAudio
// 挂断
2018-05-10 20:06:13.660428+0800 BigoOfree[19394:8610838] [I][CallKit] endCallWithCompletion
2018-05-10 20:06:13.680038+0800 BigoOfree[19394:8610838] [I][CallKit] performEndCallAction
2018-05-10 20:06:13.798381+0800 BigoOfree[19394:8610628] [I][CallKit] releaseAudio
2018-05-10 20:06:13.799067+0800 BigoOfree[19394:8610838] [I][CallKit] unprepareSDK
2018-05-10 20:06:14.392857+0800 BigoOfree[19394:8610850] [I][CallKit] didDeactivateAudioSession
```

### 3、OFree在后台状态（应用被杀掉、手机锁屏）
#### VoIP Push
请看[VoIP Push](http://jpache.com/2016/03/18/iOS-VoIP-VoIP-Push-%E5%BC%80%E5%8F%91%E9%9B%86%E6%88%90/)介绍

OFree接收到VoIP Push后，如果应用未被完全kill掉，应用则被唤醒，
若被完全kill掉，AppDelegate的`- application:didFinishLaunchingWithOptions:`会被调起，
linkd正常登录后接收到来电拓传，这时候会走在前台状态的流程，系统通话页面会被呼起。

OFree在接收到VoIP Push后的调用时序如下：

![OFree在接收到VoIP Push后的调用时序](/media/image/ios_callkit_6.png)

## 四、风险评估
（风险从高到低排序）
#### 1.mediasdk释放问题
CallKit模式下mediasdk依赖系统的回调，如果mediasdk没被正常释放，会影响到下一次通话，所以code review要更加彻底，稳定性测试上要更加严格

#### 2.VoIP Push延迟问题
目前在内测版中，VoIP存在一定的延迟，容易导致超时。

#### 3.优化OFree启动速度
OFree被杀掉后台，收到VoIP Push后会开始启动，如果启动速度太慢会影响用户体验

#### 4.异常处理
要处理好勿扰模式，被其他电话打断等异常处理（与系统电话及其他接入CallKit应用同样优先级，但不会直接被打断，用户可选择接听。）

## 五、注意点
### 1、CallKit的通话中UI
通话接通后，在非锁屏状态下，会跳转到OFree的通话页面
在锁屏状态下，会直接显示系统通话页面

### 2、点击系统通话记录无法跳转
接入CallKit后，通话信息会同步到系统的通话记录里面，点击OFree对应的通话记录后，正常情况下会跳转到OFree并触发以下delegate：

```
- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler {
    return YES;
}
```

但是我遇到了点击通话记录后无法跳转到OFree的情况。原因是`CXHandle`的type设置成了`CXHandleTypeGeneric`，但是`CXProviderConfiguration`的`supportedHandleTypes`并没有包含`CXHandleTypeGeneric`，所以初始化配置时加上该类型即可。

```
+ (CXProviderConfiguration *)providerConfiguration API_AVAILABLE(ios(10.0)) {
    static dispatch_once_t onceToken;
    static CXProviderConfiguration *config = nil;
    dispatch_once(&onceToken, ^{
        config = [[CXProviderConfiguration alloc] initWithLocalizedName:@"OFree"];
        config.supportedHandleTypes = [NSSet setWithObjects:@(CXHandleTypeGeneric), @(CXHandleTypePhoneNumber), nil];
    });
    return config;
}
```


## 六、参考
[官方文档](https://developer.apple.com/documentation/callkit?language=objc)
[官方Demo](https://developer.apple.com/videos/play/wwdc2016/230/)
