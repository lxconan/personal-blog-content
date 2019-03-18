---
title: "ASP.NET Core 沉思录 - Logging 的两种介入方法"
date: 2019-03-18 17:22:33
tags:
- aspnetcore
- dependency injection
- logging
---

ASP.NET Core 中依赖注入是一个很重要的环节。因为几乎所有的对象都是由它创建的（相关文章请参见《ASP.NET Core 沉思录 - ServiceProvider 的二度出生》）。因此整个日志记录的相关类型也被直接添加到了 `IServiceCollection` 中。今天我们将介绍各个接口/类型之间的关系，并找到介入日志记录功能的两个主要的入口。

## ASP.NET Core 的日志功能结构

为什么当我们恰当对日志进行配置之后就可以记录日志呢。那是因为 Framework 一定已经将最关键的类型添加到了 `IServiceCollection` 中。在 ASP.NET Core 中，应用程序的启动一定会创建 `WebHost` 而创建 `WebHost` 一般会用到 `WebHostBuilder`。在 `WebHostBuilder` 创建过程中就将一些核心的日志记录相关类型添加到了 `IServiceCollection` 中。

前面我们介绍过，`IServiceProvider` 有两次创建过程，一次是在调用 `WebHostBuilder.Build` 时创建的 Hosting 相关的 `IServiceProvider`，另一次是在 `WebHost.StartXxx` 方法时创建的应用程序使用的 `IServiceProvider`。而日志相关类型第一次的 `IServiceProvider` 创建之前就已经添加到 `IServiceCollection` 中了。

在本文写作时，第一次 `IServiceCollection` 实例的创建是在 `WebHostBuilder.BuildCommonServices` 方法中，大概的流程是：

* 创建 `ServiceCollection`
* 调用 `AddLogging` 扩展方法
* 将 `IOptions<>` 及其它形式、`ILoggerFactory`、`ILogger<>`、`IConfigurator`、`IConfigureOptions<LoggerFilterOptions>` 添加到 `ServiceCollection` 中。
* 如果在配置 `IWebHostBuilder` 过程中调用过 `ConfigureLogging` 则调用相关委托添加或替换相关日志记录的类型。

代码请参见：[WebHostBuilder.cs](https://github.com/aspnet/AspNetCore/blob/master/src/Hosting/Hosting/src/WebHostBuilder.cs) 以及 [LoggingServiceCollectionExtensions.cs](https://github.com/aspnet/Extensions/blob/master/src/Logging/Logging/src/LoggingServiceCollectionExtensions.cs)。

因此，以上提到的类型就是日志功能的核心类型了。梳理一下这些类型的依赖关系就可以弄清 ASP.NET Core 的日志功能结构了。

> 弄清一个结构需要关注两个部分的内容，第一个部分是声明上的依赖关系（静态），另一个部分是调用上的依赖关系（动态）。