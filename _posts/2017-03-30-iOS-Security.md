---
layout: post
title: 对iOS数据安全的一次小探索
---

## 一、主流保证数据安全的方式
### 1、网络传输安全
***1.采用HTTPS通信协议***  
可防止抓包窃取、篡改传输数据，大大增加了中间人攻击的成本。

***2.对敏感数据进行签名校正***  
采用非对称加密的方式，对敏感数据使用密钥加密，到了客户端用公钥解密，验证数据一致性，防止通信过程中被篡改。

***3.采用密文传输***  
别人即使截取了传输信息，也无法看懂其中的意思。

### 2、本地存储数据安全
***1.数据库加密***  
通过越狱设备可以很容易地把整个应用包（包括里面的数据）copy出来，这样就可以获取里面的数据库，对于没有加密的数据库就可以非常轻易地读取里面的信息，造成信息的泄漏。
***数据库加密可分两个维度：1.整个数据库加密；2.对部分字段先加密再存数据库。***
***对部分字段加密并不适合多字段的加密存储，容易导致加密数据太过分散，影响性能。所以推荐对整个数据库加密。***


***2.KeyChain存储敏感数据***  
KeyChain是iOS系统级的存储方式，安全性无需质疑，且删除应用或者升级系统依然可以保留里面的信息。

### 3、源码安全

使用越狱设备可以很轻易地把应用砸壳，从而把源码dump下来，即使是没有太多经验的开发者也可以得到应用的类信息，包括函数名等。使用IDA等反编译工具可以看到应用的一些类名和方法名，进而可以分析功能实现的逻辑。

***1.字符串混淆***  
对应用程序中使用到的字符串进行加密，保证源码被逆向后也能保护明文字符串。

***2.类名、方法名混淆***  
市面上很多iOS应用都没有混淆类名方法名，以致于很容易使用class-dump下来，从而进行hook操作，[一步一步实现iOS微信自动抢红包(非越狱)](http://www.jianshu.com/p/189afbe3b429)是很有趣的一个应用。[这个库](https://github.com/Polidea/ios-class-guard)可以混淆OC的类名、协议、属性还有方法名。

***3.程序结构混淆加密***  
对应用程序逻辑结构进行打乱混排，保证源码可读性降到最低。[可参考这个库](https://github.com/obfuscator-llvm/obfuscator/)

***4.反调试、反注入等一些主动保护策略***  
加入第三方安全性SDK


## 二、对数据库加密的研究
### 1、主流数据库加密方式

>The SQLite Encryption Extension（收费）

>SQLiteEncrypt（收费）

>SQLiteCrypt（收费）

>SQLCipher（开源免费）


只有SQLCipher是免费的，所以本文主要对SQLCipher进行研究。
### 2、引入SQLCipher第三方库
***1.手动引入***  
请参考[官方教程](https://www.zetetic.net/sqlcipher/ios-tutorial/)。

***2.使用CocoaPods***  
`pod 'FMDB/SQLCipher', '2.5'`

### 3、SQLCipher的可行性研究
#### 1.新建加密数据库
若没有旧数据的情况下使用很简单，只需要在FMDatabase里面的`-open`和`-openWithFlags:`方法里面添加`[self setKey:kDatabaseEncryptKey];`即可。
如下图所示


![添加密钥](http://upload-images.jianshu.io/upload_images/989987-17fbeb85a0beb88f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**但是在团队协作中，如果直接修改pod仓库里面的文件，可能不好同步，下面有一个技巧，就是继承FMDatabase和FMDatabaseQueue并重载其中的方法。**

继承FMDatabase的子类需要重载以下方法

```
- (BOOL)open {
    if (_db) {
        return YES;
    }
    int err = sqlite3_open([self sqlitePath], &_db );
    if(err != SQLITE_OK) {
        NSLog(@"error opening!: %d", err);
        return NO;
    } else {
        // 设置密钥
        [self setKey:kDatabaseEncryptKey];
    }
    if (_maxBusyRetryTimeInterval > 0.0) {
        // set the handler
        [self setMaxBusyRetryTimeInterval:_maxBusyRetryTimeInterval];
    }
    return YES;
}

- (BOOL)openWithFlags:(int)flags {
    if (_db) {
        return YES;
    }
    int err = sqlite3_open_v2([self sqlitePath], &_db, flags, NULL /* Name of VFS module to use */);
    if(err != SQLITE_OK) {
        NSLog(@"error opening!: %d", err);
        return NO;
    } else {
        // 设置密钥
        [self setKey:kDatabaseEncryptKey];
    }
    if (_maxBusyRetryTimeInterval > 0.0) {
        // set the handler
        [self setMaxBusyRetryTimeInterval:_maxBusyRetryTimeInterval];
    }
    return YES;
}

- (const char*)sqlitePath {
    if (!_databasePath) {
        return ":memory:";
    }
    if ([_databasePath length] == 0) {
        return ""; // this creates a temporary database (it's an sqlite thing).
    }
    return [_databasePath fileSystemRepresentation];
}
```
继承FMDatabaseQueue的子类只需要重载一个方法

```
+ (Class)databaseClass {
    return [FMEncryptDatabase class];
}
```

使用的时候只需要更改下面的语句

```
// 原：
self.normalQueue = [FMDatabaseQueue databaseQueueWithPath:self.normalDbPath];
// 改为：
self.encryptQueue = [FMEncryptDatabaseQueue databaseQueueWithPath:self.encryptDbPath];
```

#### 2.加密已有数据库

加密已有数据库我已经封装成下面的方法：

```
/**
 加密数据库（保留原有数据库）
 */
+ (BOOL)encryptDatabase:(NSString *)origPath toPath:(NSString *)toPath {
    sqlite3 *db;
    if (sqlite3_open([origPath UTF8String], &db) == SQLITE_OK) {
        char *err = NULL;
        sqlite3_exec(db, [[NSString stringWithFormat:@"ATTACH DATABASE '%@' AS encrypted KEY '%@';", toPath, kDatabaseEncryptKey] UTF8String], NULL, NULL, &err);
        sqlite3_exec(db, "SELECT sqlcipher_export('encrypted');", NULL, NULL, &err);
        sqlite3_exec(db, "DETACH DATABASE encrypted;", NULL, NULL, &err);
        sqlite3_close(db);
        return err ? NO : YES;
    } else {
        sqlite3_close(db);
        NSLog(@"Open db failed:%s", sqlite3_errmsg(db));
        return NO;
    }
}
```
***这里有两点需要注意的地方：***  
***1、这里必须使用全路径***  
***2、加密后的queue必须同步更换为FMEncryptDatabaseQueue，不然无法读写数据***  

#### 3.解密已加密数据库
解密已加密的数据库已封装为以下方法：

```
/**
 解密数据库（保留原有数据库）
 */
+ (BOOL)unencryptDatabase:(NSString *)origPath toPath:(NSString *)toPath {
    sqlite3 *db;
    if (sqlite3_open([origPath UTF8String], &db) == SQLITE_OK) {
        char *err = NULL;
        sqlite3_exec(db, [[NSString stringWithFormat:@"PRAGMA key = '%@';", kDatabaseEncryptKey] UTF8String], NULL, NULL, &err);
        sqlite3_exec(db, [[NSString stringWithFormat:@"ATTACH DATABASE '%@' AS plaintext KEY '';", toPath] UTF8String], NULL, NULL, NULL);
        sqlite3_exec(db, "SELECT sqlcipher_export('plaintext');", NULL, NULL, &err);
        sqlite3_exec(db, "DETACH DATABASE plaintext;", NULL, NULL, &err);
        sqlite3_close(db);
        
        return err ? NO : YES;
    } else {
        sqlite3_close(db);
        NSLog(@"Open db failed:%s", sqlite3_errmsg(db));
        return NO;
    }
}
```
***这里也有两点需要注意的地方：***  
***1、这里必须使用全路径***  
***2、解密后的queue必须同步更换为FMDatabaseQueue，不然无法读写数据***  

### 4、SQLCipher性能分析
***本次性能分析主要从两个维度下手：
1.对比无加密数据库和加密数据库的插入数据速度；
2.加密已有数据的数据库耗时。***

***本次测试都基于iOS10.3的iPad mini2下进行。***
#### 1.对比插入数据速度
规则：对比无加密数据库和加密数据库使用事务时，在插入1w条、5w条和10w条8个字段的数据的耗时，每组数据测试5次，最后取平均值。最后的测试结果如下图所示：


![插入测试结果](http://upload-images.jianshu.io/upload_images/989987-bb2ea812ee888603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***结论：两者相差在4%左右，在移动设备上存储大数据的情况很少，所以我认为是完全可以接受的。***

#### 2.测试加密数据库耗时
规则：对比存储1w条、5w条和10w条数据的数据库加密的耗时，每组数据测试5次，最后取平均值。最后的测试结果如下图所示：


![加密耗时测试结果](http://upload-images.jianshu.io/upload_images/989987-7bd94c957951fd08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***结论：加密过程需要一定的时间，且跟数据库所存储的数据量有关，数据量越大耗时越长，从本次的测试结果来看，加密存储10w条数据的数据库耗时1.38s是可以接受的。***

### 5、采用SQLCipher存在的风险
 1、SQLCipher依赖本地存储的字符串进行数据库的加密和解密，如果这个字符串被泄漏出去，本地的数据库依然容易被解密，进而被读取里面的信息。可采用KeyChain保存？
 2、加密已存储大数据的数据库会消耗一定的时间，这段时间怎么处理才能让用户无感知？加密失败了怎么处理？
3、欢迎大家补充。。。
### 6、[本文demo](https://github.com/OuDuShu/SQLCipherDemo)