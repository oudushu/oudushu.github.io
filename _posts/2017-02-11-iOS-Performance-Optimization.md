---
layout: post
title: iOS 性能优化汇总
---

参考：https://i.cmgine.net/archives/14896.html


入门：
1、用ARC管理内存
2、在正确的地方使用reuseIdentifier
3、尽量吧views设置为完全不透明
4、避免过于庞大的xib
5、不要阻塞主线程
6、不要在ImageView中调整图片大小，尽量使用UIImageView大小相同的图片
7、选择正确的Collection
Arrays: 有序的一组值。使用index来lookup很快，使用value lookup很慢， 插入/删除很慢。

Dictionaries: 存储键值对。 用键来查找比较快。

Sets: 无序的一组值。用值来查找很快，插入/删除很快。

8、打开gzip压缩


中级：
1、重用和懒加载views
2、Cache，缓存不大可能改变但是需要经常读取的东西（NSCache）
3、权衡渲染方法（View？CALayer？CoreGraphics？OpenGL？）
4、处理内存警告
在app delegate中使用applicationDidReceiveMemoryWarning:的方法

在你的自定义UIViewController的子类(subclass)中覆盖didReceiveMemoryWarning

注册并接收 UIApplicationDidReceiveMemoryWarningNotification 的通知

5、重用大开销对象（如NSDateFormatter，NSCalendar）
6、使用Sprite Sheets
7、避免反复处理数据
解析JSON会比XML更快一些，JSON也通常更小更便于传输。从iOS5起有了官方内建的JSON deserialization 就更加方便使用了。

但是XML也有XML的好处，比如使用SAX 来解析XML就像解析本地文件一样，你不需像解析json一样等到整个文档下载完成才开始解析。当你处理很大的数据的时候就会极大地减低内存消耗和增加性能。

8、正确设定背景图片
使用UIColor的 colorWithPatternImage来设置背景色；

在view中添加一个UIImageView作为一个子View。（不使用“imageName:”）

9、减少使用Web特性
10、设定Shadow Path
view.layer.shadowPath = [[UIBezierPath bezierPathWithRect:view.bounds] CGPath];

11、优化UITableView
正确使用reuseIdentifier来重用cells

尽量使所有的view opaque，包括cell自身

避免渐变，图片缩放，后台选人

缓存行高

如果cell内现实的内容来自web，使用异步加载，缓存请求结果

使用shadowPath来画阴影

减少subviews的数量

尽量不适用cellForRowAtIndexPath:，如果你需要用到它，只用一次然后缓存结果

使用正确的数据结构来存储数据

尽量使用rowHeight,sectionFooterHeight和sectionHeaderHeight来设定固定的高，不要请求delegate

12、选择正确的数据存储选项
使用NSUerDefaults

使用XML, JSON, 或者 plist

使用NSCoding存档

使用类似SQLite的本地SQL数据库

使用 Core Data



进阶：
1、加速启动时间
2、使用Autorelease Pool
3、选择是否缓存图片
4、避免日期格式转换
