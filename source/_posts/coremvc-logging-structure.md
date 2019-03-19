---
title: "ASP.NET Core 沉思录 - Logging 的两种介入方法"
date: 2019-03-18 17:22:33
tags:
- aspnetcore
- dependency injection
- logging
---

ASP.NET Core 中依赖注入是一个很重要的环节。因为几乎所有的对象都是由它创建的（相关文章请参见《ASP.NET Core 沉思录 - ServiceProvider 的二度出生》）。因此整个日志记录的相关类型也被直接添加到了 `IServiceCollection` 中。今天我们将介绍各个接口/类型之间的关系，并找到介入日志记录功能的两个主要的入口。

<!--more-->

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

弄清一个结构需要关注两个部分的内容，第一个部分是声明上的依赖关系（静态），另一个部分是调用上的依赖关系（动态）。因此我们也会采用这种方式进行梳理。首先声明上的依赖关系如下：

<img src="{{root_url}}/images/blog/core-mvc-logger-struct.png" style="text-align:center" alt="logger structure"/>

以上结构在运行时会创建三层 `ILogger` 实现

* 最外面的一层就是我们使用的 `ILogger<>` 实现。它是由 `IServiceProvider` 直接创建的。
* 第二层是 `LoggerFactory` 创建的 `Logger`，它是一个组合 Logger。
* 第三层是由 `ILoggerProvider` 实现创建的负责具体记录日志的 `ILogger` 实现。这些 Logger 都会添加到第二层的组合 `Logger` 中。

而后，在日志记录时，正是按照这三层结构进行任务分发的。

## 介入日志功能的几个入口

在了解上述结构之后不难使用我们的具体实现替换默认的 .NET 日志记录结构。首先在使用上 ASP.NET Core 上使用的日志记录接口有两种，第一种即 `ILoggerFactory`，进而创建 `ILogger`。例如 [这里](https://github.com/aspnet/AspNetCore/blob/master/src/Http/Routing/src/RouterMiddleware.cs)。而另外一种则是直接使用 `ILogger<>`。因此我们的介入点有两个：

* 直接替换 `ILoggerFactory` 的实现，这样可以完全定义自己的 Logger -> Sink 结构和过滤逻辑；但工作量较大。
* 实现 `ILoggerProvider`，并将其添加到 `IServiceCollection` 中。这样可以复用现有的日志逻辑。

很多第三方日志库就是有选择的采用了上述一种或者两种方式。例如，以 Serilog 为例，它支持上述两种接口：

首先、我们可以使用 `webHostBuilder.UseSerilog(...)` 的方式将其纳入应用程序中。而这种方式使用第一种介入方法。具体代码请看 [这里](https://github.com/serilog/serilog-aspnetcore/blob/dev/src/Serilog.AspNetCore/SerilogWebHostBuilderExtensions.cs)。

其次、我们可以使用 `loggerBuilder.AddSerilog(...)` 的方式将其纳入应用程序中。而这种方式使用第二种介入方法。具体代码请看 [这里](https://github.com/serilog/serilog-extensions-logging/blob/dev/src/Serilog.Extensions.Logging/SerilogLoggingBuilderExtensions.cs)。

## 总结

日志记录有很多可以思考的地方，今天我们只关注 ASP.NET Core 中日志介入的入口。总结起来有两个主要入口：

* 第一、直接替换 `ILoggerFactory` 实现；
* 第二、实现具体的 `ILoggerProvider` 并将其添加到 `IServiceCollection` 中。

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>