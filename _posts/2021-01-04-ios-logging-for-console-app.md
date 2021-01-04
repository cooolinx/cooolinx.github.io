---
layout: post
title: "iOS输出日志到Console.app"
subtitle: "有些日志我们希望输出到Mac的Console.app，而非Xcode的控制台"
background: '/assets/bg/puket.jpg'
date:   2021-01-04 17:02:00 +0800
categories: others
---

# 问题

有些时候，我们不希望仅仅将日志输出到Xcode的控制台，而是Mac的`Console.app`（中文系统也叫`控制台`，但是独立的控制台App）。例如在非Xcode环境下检查日志，或是因为App Extension（例如开发科学上网常用的`Network Extension`）。

另外有时我们使用一些Swift以外的语言，例如C/C++等。

我的场景是：**在开发`Network Extension`时，用到了`lwIP`，是Swift -> Go -> C++的结构，需要在lwIP端输出日志到Console.app。**但在网上找了很久都没有很合适的答案。

# 思路

接下来我们编写一段代码，来测试如下几种类型的日志输出情况：

- [NSLog](https://developer.apple.com/documentation/foundation/1395275-nslog)：使用最多的iOS/Mac日志工具，缺点是仅支持Objective-C/Swift
- [OSLog](https://developer.apple.com/documentation/os/logging)：Apple希望在iOS 10+之后一统天下的日志库，Objective-C/Swift/C/C++都可支持
- [syslog](https://github.com/openbsd/src/blob/master/sys/sys/syslog.h)：FreeBSD支持的系统日志库

# 验证

第一步，Xcode创建一个App项目：LogApp，语言选`Swift`其他随意。

第二步，在`AppDelegate.swift`文件中的启动函数中，输入：

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // Override point for customization after application launch.
    hello();
    NSLog("logtest: NSLog")
    os_log("logtest: os_log")
    ...
```

第三步，创建一个`console.h`文件，内容如下：

```c
#ifndef log_h
#define log_h

#include <stdio.h>
#include <syslog.h>
#include <os/log.h>

void hello() {
    printf("logtest: printf\n");
    syslog(LOG_INFO, "logtest: syslog LOG_INFO");
    syslog(LOG_NOTICE, "logtest: syslog LOG_NOTICE");
    os_log_info(OS_LOG_DEFAULT, "logtest: os_log_info in c++");
    os_log_error(OS_LOG_DEFAULT, "logtest: os_log_error in c++");
    os_log_with_type(OS_LOG_DEFAULT, OS_LOG_TYPE_DEFAULT, "logtest: os_log_with_type in c++");
}

#endif /* log_h */
```

第四步，创建桥接文件`LogApp-Bridging-Header.h`，内容如下：

```c
#import "console.h"
```

同时在`Build Settings`中配置`Objective-C Bridging Header`的值为`LogApp/LogApp-Bridging-Header.h`

第五步，运行。

在Xcode控制台中，显示：

```
logtest: printf
2021-01-04 17:24:22.196717+0800 LogApp[6626:6846463] logtest: syslog LOG_INFO
2021-01-04 17:24:22.196758+0800 LogApp[6626:6846463] logtest: syslog LOG_NOTICE
2021-01-04 17:24:22.196814+0800 LogApp[6626:6846463] logtest: os_log_info in c++
2021-01-04 17:24:22.197344+0800 LogApp[6626:6846463] logtest: os_log_error in c++
2021-01-04 17:24:22.197371+0800 LogApp[6626:6846463] logtest: os_log_with_type in c++
2021-01-04 17:24:22.197675+0800 LogApp[6626:6846463] logtest: NSLog
2021-01-04 17:24:22.197735+0800 LogApp[6626:6846463] logtest: os_log
```

在Console.app中显示：

```
默认  17:24:23.023423+0800  LogApp logtest: syslog LOG_NOTICE
错误  17:24:23.023614+0800  LogApp logtest: os_log_error in c++
默认  17:24:23.023643+0800  LogApp logtest: os_log_with_type in c++
默认  17:24:23.023672+0800  LogApp logtest: NSLog
默认  17:24:23.024055+0800  LogApp logtest: os_log
```

结论如上。
