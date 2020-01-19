---
layout: post
title: 直播间横屏技术方案
---


# 直播间横屏技术方案

### 一、背景
目前Likee直播间已支持PC推流直播，但只支持竖屏播放，导致观看体验不佳，故需要把横屏模式加上。
### 二、面临的问题
1、只有两天开发时间，时间紧迫；
2、屏幕切换时存在UI异常风险；
3、横屏模式需要应用全局支持横屏方向，存在影响其他业务风险；
### 三、方案
#### 1、切换横屏入口的显示（加开关）
joinChannel/joinGroup成功 -&gt; roomType ==&nbsp;BSRoomType_PCRoom -&gt; 判断开关 -&gt; 显示入口

#### 2、横竖屏切换初步方案
##### 1.项目配置：


在AppDelegate里重写方法：
```
- (UIInterfaceOrientationMask)application:(UIApplication *)application supportedInterfaceOrientationsForWindow:(UIWindow *)window {
    UIViewController *topViewController = [BaseViewController bl_findTopMostViewController];
    if (topViewController && [topViewController isKindOfClass:NSClassFromString(@"BVLiveLandscapeViewController")]) {
        return topViewController.supportedInterfaceOrientations;
    }
    return UIInterfaceOrientationMaskPortrait;
}
```

##### 2.竖屏转横屏：
新建一个支持横屏的BVLiveLandscapeViewController并重写UIViewController的方法：
```
- (UIInterfaceOrientationMask)supportedInterfaceOrientations {
    return UIInterfaceOrientationMaskLandscapeRight | UIInterfaceOrientationMaskPortrait; // 这里加上竖屏，是因为横屏转竖屏时，需要把当前vc先转过来，后面会讲到
}

- (BOOL)shouldAutorotate {
    return YES;
}
```
从竖屏vc（`BSWatchLiveShowViewController`）旋转时，初始化横屏vc（`BVLiveLandscapeViewController`），并把竖屏vc当前持有的`BLLiveRoomAudienceSession`传递给横屏vc。然后在当前navigationController的子vc中，把竖屏vc替换成横屏vc（原竖屏vc会销毁）。

```
- (void)switchToLandscapeView {
    BVLiveLandscapeViewController *viewController = [[BVLiveLandscapeViewController alloc] initWithAudienceSession:self.audienceSession];
    
    NSMutableArray *viewControllers = [[NSMutableArray alloc] initWithArray:self.navigationController.viewControllers];
    [viewControllers removeObject:self];
    [viewControllers addObject:viewController];
    [self.navigationController setViewControllers:viewControllers animated:YES];
    
    self.audienceSession = nil;
}
```

横屏vc初始化videoView，并调用session的方法传递给mediasdk（`[self.audienceSession changeRemoteViewForGroup:self.videoView];`）

##### 3.横屏转竖屏：
正常情况下，从横屏转竖屏时，新的竖屏vc会被初始化。而竖屏vc的部分子view初始化时会通过`[[UIScreen mainScreen] bounds].size`取当前屏幕的size布局frame，但是此时的屏幕状态还是横屏状态，所以取出来的屏幕size是不对的，导致旋转后的vc的UI元素会出现错乱。

***
为了避免UI元素出现错乱的问题，目前有三种方案：

(1)子view全改为AutoLayout布局（工时不允许，放弃）
(2)子view补充autoresizingMask属性（工时不允许，放弃）
(3)横屏vc先转成竖屏，再跳转真正的竖屏vc（鉴于当前横屏vc的UI元素比较简单，且工时紧迫，现采用这种临时方案，若后面要丰富横屏模式的内容，估计还是需要采用上面两种方案。）
***

在用户点击恢复竖屏的按钮时，通过`[self setOrientation:UIInterfaceOrientationPortrait];`设置当前vc为竖屏状态，然后重写UIViewController的方法监听旋转进度：
```
- (void)viewWillTransitionToSize:(CGSize)size withTransitionCoordinator:(id<UIViewControllerTransitionCoordinator>)coordinator {
    [super viewWillTransitionToSize:size withTransitionCoordinator:coordinator];
    
    self.rt_navigationController.view.backgroundColor = [UIColor blackColor]; // 解决旋转时view的空隙白色的问题
    
    [coordinator animateAlongsideTransition:^(id<UIViewControllerTransitionCoordinatorContext> context) {
        // Place code here to perform animations during the rotation.
        // You can pass nil or leave this block empty if not necessary.
    } completion:^(id<UIViewControllerTransitionCoordinatorContext> context) {
        // Code here will execute after the rotation has finished.
        // Equivalent to placing it in the deprecated method -[didRotateFromInterfaceOrientation:]
        
        self.rt_navigationController.view.backgroundColor = [UIColor whiteColor]; // 恢复原背景色
        
        if (self.isUserClickedToPortraite) {
            [self jumpToPortraitViewController]; // 旋转完成，判断是用户触发的，则跳转到竖屏vc
        } 
    }];
}
```
#### 四、异常case
##### 1、旋转时白底
重写UIViewController的方法监听旋转进度，在执行旋转前把`BVRootNavigationController`的背景色设为黑色，旋转完成后恢复为原来的白色，`self.rt_navigationController.view.backgroundColor = [UIColor blackColor];`
##### 2、横屏转竖屏视频闪烁一下
原因是把新的videoView设置到mediasdk时，mediasdk把画面渲染在新的videoView上会卡顿一下。目前的做法是，在新建竖屏vc时，会把当前的videoView传递到竖屏vc，在竖屏vc重新布局约束。
##### 3、横屏转竖屏时旋转时画面变形
原因是mediasdk识别到方向改变时，会修改渲染方向，此时画面比例不对画面会有拉伸。目前的做法是横屏状态下videoView写死16:9比例。
##### 4、旋屏时视频过渡不自然
转场之前先把videoView改为新的vc大小
```
//模拟横屏video展示状态，圆滑的过渡到横屏
    [self.videoView mas_remakeConstraints:^(MASConstraintMaker *make) {
        make.edges.mas_equalTo(self.watchLiveView);
    }];
```
##### 5、屏蔽手机自动旋转
前面为了能手动把横屏转成竖屏，在supportedInterfaceOrientations方法里添加了竖屏的支持，这会导致应用会随系统自动切换方向，而我们是希望能屏蔽自动切换方向，目前的做法是维护一个标志位，在需要手动把横屏转成竖屏时再把标志位打开：
```
- (BOOL)shouldAutorotate {
    return self.shouldRotate;
}

- (void)setOrientation:(UIInterfaceOrientation)orientation {   
    self.shouldRotate = YES;
    [super setOrientation:orientation];
    self.shouldRotate = NO;
}
```
##### 6、启动页横屏了
![c5e1eb0cc92e07901a9b34c2354a9634.png](evernotecid://B3EB39CD-D5CA-4304-B017-DED874D7F245/appyinxiangcom/10739293/ENResource/p1855)
原来的做法在项目的General里选中 “Landscape Left” 和 “Landscape Right"，这会导致启动页横屏，目前是不勾选General里的横屏选项，而在AppDelegate里重写，也能达到一样的效果：

```
- (UIInterfaceOrientationMask)application:(UIApplication *)application supportedInterfaceOrientationsForWindow:(UIWindow *)window {

}
```

>Discussion

>This method returns the total set of interface orientations supported by the app. When determining whether to rotate a particular view controller, the orientations returned by this method are intersected with the orientations supported by the root view controller or topmost presented view controller. The app and view controller must agree before the rotation is allowed.

>If you do not implement this method, the app uses the values in the UIInterfaceOrientation key of the app’s Info.plist as the default interface orientations.


##### 7、`[UIDevice currentDevice].orientation`获得的方向不对
在一些极端情况，如从竖屏转横屏时，把设备迅速摆正，此时偶现`[UIDevice currentDevice].orientation`获得的方向不对。目前的做法是不依赖`[UIDevice currentDevice].orientation`，而是自己维护一个状态。

##### 8、崩溃
UIAlertController加了一个分类，重写了shouldAutorotate
```
2019-12-25 20:15:51.180535+0800 LIKE[894:136992] *** Terminating app due to uncaught exception 'UIApplicationInvalidInterfaceOrientation', reason: 'Supported orientations has no common orientation with the application, and [UIAlertController shouldAutorotate] is returning YES'
*** First throw call stack:
(0x1a0fb098c 0x1a0cd90a4 0x1a0ea6054 0x1a49ca8cc 0x1a49cae14 0x1a49cb450 0x1a49b7cb8 0x1a543f048 0x1a54331d4 0x1a128830c 0x1a54330d8 0x1a54419c4 0x1a4615ec8 0x1a48d69b4 0x1a48d44f8 0x1a4fc6b98 0x1a4fb67c0 0x1a4fe6594 0x1a0f2dc48 0x1a0f28b34 0x1a0f29100 0x1a0f288bc 0x1aad93328 0x1a4fbd6d4 0x104f97f30 0x1a0db3460)
libc++abi.dylib: terminating with uncaught exception of type NSException
```