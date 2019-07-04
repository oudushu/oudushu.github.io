---
layout: post
title: 在树莓派上部署ATC网络模拟工具（Augmented Traffic Control）
---

## 一、前言
作为移动开发者的我们，为了良好的用户体验，经常需要模拟手机应用在比较差的网络环境下的表现，模拟网络环境的方式有很多，比如使用Charles，或者在手机的开发者模式下模拟网络环境等等，但是这些都有一定的门槛。

使用Charles，首先你得连接WiFi，然后设置代理，接着开启网络模拟模式，最后测试完了如果忘了关闭代理，可能手机就上不了网了。而且都连上你电脑的代理的话就一次只能模拟一种网络环境。

使用手机的开发者模式，首先你的手机能进入开发者模式。

*如果你需要随便抓一位不懂技术同事帮你测试，这些方式都不太友好，有没有一种方式可以连上WiFi就可以使用的测试方式呢？有！接下来就介绍Facebook出品的一款网络模拟工具ATC。*

## 二、简介
> Augmented Traffic Control (ATC) is a tool to simulate network conditions. It allows controlling the connection that a device has to the internet. Developers can use ATC to test their application across varying network conditions, easily emulating high speed, mobile, and even severely impaired networks. 

ATC全名叫Augmented Traffic Control，是Facebook出品的一款网络模拟工具，移动开发者可以通过这款工具模拟不同条件下的网络环境，可以通过网页自由地模拟网络带宽（bandwidth）、延迟（latency）、丢包率（packet loss）、错包率（corrupted packets）和乱序率（packets ordering）。

***而且！！！更牛逼的是：不同的设备连接到同一WiFi还可以模拟不同的网络环境互不影响。***

![](https://camo.githubusercontent.com/99a779517d18a605046dec36d2beb02166f4e77b/68747470733a2f2f66616365626f6f6b2e6769746875622e696f2f6175676d656e7465642d747261666669632d636f6e74726f6c2f696d616765732f6174635f6f766572766965772e706e67)

***[github地址](https://github.com/facebook/augmented-traffic-control)***

## 三、准备
***1、树莓派3（已内置有无线网卡）  
2、已刷入最新RASPBIAN系统的SD卡***

## 四、安装
***安装主要有两步：  
1、让树莓派有发射AP热点的能力；  
2、安装ATC。***

### 1、让树莓派有发射AP热点的能力
1.安装hostapd虚拟热点程序和dnsmasq配置DHCP、DNS服务程序：

`sudo apt-get install dnsmasq hostapd`

2.编辑`sudo vim /etc/dhcpcd.conf`文件，在该文件的最后加入下面的命令，用来为wlan0固定一个内网IP。

```
interface wlan0
static ip_address=10.0.0.1/24
```

3.编辑`sudo vim /etc/dnsmasq.conf`dnsmasq配置文件，有600+行的内容，我们可以`shift+g`滚动到最下面，然后加入下面的命令，用来控制IP地址的取值范围

```
interface=wlan0
dhcp-range=10.0.0.2,10.0.0.50,255.255.255.0,12h
```

4.添加hostapd.conf配置文件`sudo vim /etc/hostapd/hostapd.conf`并添加下面内容，用于配置热点的账号密码等信息。

```
ssid=PI3                 #账号
wpa_passphrase=12345678  #密码
interface=wlan0
driver=nl80211
hw_mode=g
channel=10
macaddr_acl=0
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
rsn_pairwise=CCMP
```

5.编辑sysctl.conf文件，设置路由转发`sudo vim /etc/sysctl.conf`，打开下面的注释

```
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1
```

6.更新iptables规则，依次执行下面命令

```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
7.编辑`sudo vim /etc/network/interfaces`，在最后加入下面的命令

```
up iptables-restore < /etc/iptables.ipv4.nat
```
8.重启树莓派后执行下面命令，如无意外就可以发射热点了

```
sudo service hostapd start 
sudo service dnsmasq start 
```

### 2、安装ATC
ATC需要依赖Python环境，RASPBIAN系统默认已经支持Python，所以不用再安装，如果你的是Ubuntu或者其他Linux系统，你需要输入下面命令先安装Python的依赖，并安装pip包管理工具。

```
sudo apt-get install python-pip python-dev build-essential）
sudo pip install --upgrade pip 
```

1.安装ATC依赖库

```
sudo pip install atc_thrift atcd django-atc-api django-atc-demo-ui django-atc-profile-storage
```

***注意这里要加上`sudo`，不然会因没有权限而安装失败。***

2.创建新的Django项目

```
django-admin startproject atcui
cd atcui
```

3.编辑settings.py文件`sudo vim atcui/settings.py`，并在INSTALLED_APPS的后面加入下面参数

```
INSTALLED_APPS = (
    ...
    # Django ATC API
    'rest_framework',
    'atc_api',
    # Django ATC Demo UI
    'bootstrap_themes',
    'django_static_jquery',
    'atc_demo_ui',
    # Django ATC Profile Storage
    'atc_profile_storage',
)
```

4.编辑urls.py文件，并加入下面内容

```
cd atcui
sudo vim urls.py
```
```
...
...
from django.views.generic.base import RedirectView
from django.conf.urls import include

urlpatterns = [
    ...
    # Django ATC API
    url(r'^api/v1/', include('atc_api.urls')),
    # Django ATC Demo UI
    url(r'^atc_demo_ui/', include('atc_demo_ui.urls')),
    # Django ATC profile storage
    url(r'^api/v1/profiles/', include('atc_profile_storage.urls')),
    url(r'^$', RedirectView.as_view(url='/atc_demo_ui/', permanent=False)),
]
```

5.更新Django数据库

```
python manage.py migrate
```

到这里，我们已经完成整个环境的部署，步骤有点多，只要一步一步做，应该不会出问题。

## 五、使用

1、启动核心组件atcd

```sudo atcd --atcd-lan wlan0```

出现`Awaiting graceful shutdown.`后按`Ctrl+c`退出。

2、启动Django工程

```
cd ..
sudo python manage.py runserver 0.0.0.0:8000
```

3、移动设备连接WiFi，使用浏览器输入`10.0.0.1:8000/atc_demo_ui/`即可进入控制中心，添加网络配置

![控制中心](http://upload-images.jianshu.io/upload_images/989987-b5a55cc4fb0223a9.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ALLOWED_HOSTS = ['10.0.0.1']

## 六、坑

在部署ATC工具的过程中，我遇到过三个坑，还好解决起来不太难，希望大家再遇到的时候可以快速解决。

### 1、权限问题
安装ATC依赖库时没有权限，在命令签名加上`sudo`即可。

### 2、Invalid HTTP_HOST header :'xxx'. You may need to add u'xxx' to ALLOWED_HOSTS.
![](http://upload-images.jianshu.io/upload_images/989987-00763262ef56bcd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候只需要编辑`atcui`目录下的`settings.py`文件，在ALLOWED_HOSTS后加上本机ip即可：

```
ALLOWED_HOSTS = ['10.0.0.1']
```

### 3、成功进入控制中心，但是中间提示*ATC is not running*

Google了一下，发现还蛮多人遇到同样的问题，下面是作者的回复：
![ATC is not running](http://upload-images.jianshu.io/upload_images/989987-16c0a2c5e0586568.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[Issues地址](https://github.com/facebook/augmented-traffic-control/issues/153)

根据作者的提示，我重新安装了`django-rest-framework`

```
sudo apt-get install django-rest-framework
```
然后重启服务即可。
