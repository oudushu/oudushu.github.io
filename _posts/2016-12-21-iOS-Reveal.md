---
layout: post
title: iOS项目集成Reveal
---

Reveal是一款UI调试工具，在iOS开发过程中可查看UI的层级关系并可动态修改界面，可以有效提高开发效率。这篇文章主要介绍如何把Reveal集成到实际的项目中。

一、Reveal主要有两种集成方式：
1、framework集成（可忽略，直接看第二点）
（1）打开Reveal，选择 Reveal --> Help --> Show Reveal Library in Finder --> iOS Library



（2）把Reveal.framework导入到项目中


（3）配置Target，Target --> Build Setting --> Other Linker Flags 添加以下4项：


（4）运行项目 --> 打开Reveal --> 选择连接设备


项目的UI图层就可以出来了。


这种方法有几个弊端：1、需要配置项目的Build Setting，修改Other Linker Flags，多人协作开发的话如果有人没有配置就会编译出错；2、如果不留意容易打包的时候把Reveal的framework也打包进去，增大包的体积。

以下推荐另一种集成方法。

2、LLDB集成（推荐！！）
（1）运行项目 --> 点击Pause program execution。


（2）在LLDB依次输入以下两条命令

# expr (Class)NSClassFromString(@"IBARevealLoader") == nil ? (void*)dlopen("/Applications/Reveal.app/Contents/SharedSupport/iOS-Libraries/libReveal.dylib", 0x2) : ((void*)0)

# expr (void)[(NSNotificationCenter *)[NSNotificationCenter defaultCenter] postNotificationName:@"IBARevealRequestStart" object:nil];

第一条命令用于加载Reveal的动态链接库；

第二条命令用于启动Reveal调试服务。

输入两条命令后，如果LLDB打印 “ INFO: Reveal Server started ” 即成功启动Reveal服务，点击Continue即可。


虽然说输入两条命令即可启动服务，但是每次重新启动项目后都要重新输入这么长的命令也是挺苦恼的，下面介绍一种方法可以非常方便地使用Reveal的命令。

二、设置LLDB命令别名
打开终端，输入 vim ~/.lldbinit ，然后输入以下内容：

command alias reveal_load_sim expr (Class)NSClassFromString(@"IBARevealLoader") == nil ? (void*)dlopen("/Applications/Reveal.app/Contents/SharedSupport/iOS-Libraries/libReveal.dylib", 0x2) : ((void*)0)

command alias reveal_start expr (void)[(NSNotificationCenter*)[NSNotificationCenter defaultCenter] postNotificationName:@"IBARevealRequestStart" object:nil];

command alias reveal_stop expr (void)[(NSNotificationCenter*)[NSNotificationCenter defaultCenter] postNotificationName:@"IBARevealRequestStop" object:nil];

reveal_load_sim为模拟器加载reveal调试用的动态链接库

reveal_start启动reveal调试功能

reveal_stop结束reveal调试功能


输入 :wq 保存
这时候打开Xcode，在LLDB里面就会有输入提示了。


按顺序输入reveal_load_sim --> reveal_start 即可达到上面的效果。



参考资料
iOS开发中集成Reveal

UI调试神器 for ios：Reveal的使用与破解