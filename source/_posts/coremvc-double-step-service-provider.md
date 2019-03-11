---
title: "ASP.NET Core 沉思录 - ServiceProvider 的二度出生"
date: 2019-03-11 10:19:23
tags:
- aspnetcore
- dependency injection
---

ASP.NET Core 终于将几乎所有的对象创建工作都和依赖注入框架集成了起来。并对大部分的日常工作进行了抽象。使得整个框架扩展更加方便。各个部分的集成也更加容易。今天我们要思考的部分仍然是从一段每一个工程中都大同小异的代码开始的。

<!--more-->

```cs
IWebHostBuilder CreateWebHostBuilder(string[] args)
{
    return new WebHostBuilder()
        .UseKestrel(ko => ko.AddServerHeader = false)
        .ConfigureAppConfiguration(cb => cb.AddCommandLine(args))
        .ConfigureLogging(lb => {...})
        .UseStartup<Startup>();
}
```

## 0 太长不读

<img src="{{root_url}}/images/blog/core-mvc-service-providers.jpg" style="text-align:center" alt="service-providers"/>

* ASP.NET Core 的初始化包含了两个步骤：第一个步骤是 Hosting 相关服务的初始化过程，初始化完毕之后创建了第一个 `IServiceProvider` 对象；第二步是 Application 相关服务的初始化过程。而 Application 的初始化过程可以注入 Hosting 相关的服务。之后，通过 `IStartup.ConfigureServices` 方法创建了第二个 `IServiceProvider` 对象。
* 初始化过程中创建的两个 `IServiceProvider` 均会跟随 `WebHost` 的销毁而销毁。
* 通过 `Startup` 类型的构造函数注入的实例是由 Hosting 初始化阶段创建的 `IServiceProvider` 创建的。只能注入 Hosting 初始化阶段添加的类型。且最好不要使用大量消耗资源的类型。
* 可以在 `Startup.Configure` 方法中添加其他参数，这样会使用 Application 的一个 `Scope` 下的 `IServiceProvider` 进行注入，且在方法调用完毕之后该 `Scope` 即被销毁。因此该方法内可以创建资源占用量较高的需要 `Dispose` 的类型实例而不造成泄露。

## 1 WebHost 的构建主要就是向 `IServiceCollection` 中添加服务

之前提到过，任何 Framework 只有两件事情，第一件事情就是对象怎么创建，第二件事情就是如何将这些创建出来的对象塞到 Framework 处理流水线中。因此 ASP.NET Core 也是这样。在应用程序启动的时候，我们会在 `WebHostBuilder.Build` 方法调用之前进行各种各样的操作，虽然我们调用的大部分操作都是扩展方法（例如上述代码中的 `UseXxx`，和 `ConfigureLogging`），但是归根结底会调用 `IWebHostBuilder` 的以下方法：

```cs
IWebHostBuilder ConfigureAppConfiguration(Action<WebHostBuilderContext, IConfigurationBuilder> configureDelegate);
IWebHostBuilder ConfigureServices(Action<IServiceCollection> configureServices);
IWebHostBuilder ConfigureServices(Action<WebHostBuilderContext, IServiceCollection> configureServices);
```

不论调哪一个方法，它们做的事情其实都是一件。就是告诉应用程序，我到底有哪些对象需要创建，如何创建这些对象，以及其生存期如何管理。从技术角度上来说，就是将需要创建的对象类型添加到 `IServiceCollection` 中。如果感兴趣的同学可以看看 `WebHostBuilder` 的[实现代码](https://github.com/aspnet/AspNetCore/blob/master/src/Hosting/Hosting/src/WebHostBuilder.cs)，就更加清晰了。

例如，以 `ConfigureLogging` 为例，代码请参见[这里](https://github.com/aspnet/Extensions/blob/master/src/Logging/Logging/src/LoggingServiceCollectionExtensions.cs)：

```cs
public static IWebHostBuilder ConfigureLogging(
    this IWebHostBuilder hostBuilder, Action<WebHostBuilderContext, 
    ILoggingBuilder> configureLogging)
{
    return hostBuilder.ConfigureServices((context, collection) => 
        collection.AddLogging(builder => configureLogging(context, builder)));
}

public static IServiceCollection AddLogging(
    this IServiceCollection services, 
    Action<ILoggingBuilder> configure)
{
    if (services == null) { throw new ArgumentNullException(nameof(services)); }

    services.AddOptions();
    services.TryAdd(ServiceDescriptor.Singleton<ILoggerFactory, LoggerFactory>());
    services.TryAdd(ServiceDescriptor.Singleton(typeof(ILogger<>), typeof(Logger<>)));
    services.TryAddEnumerable(ServiceDescriptor.Singleton<IConfigureOptions<LoggerFilterOptions>>(
        new DefaultLoggerLevelConfigureOptions(LogLevel.Information)));
    configure(new LoggingBuilder(services));
    return services;
}
```

可以看到实际上就是将 `IOptions<>`、`IOptionsSnapshot<>`、`IOptionsMonitor<>`、`IOptionsFactory<>`、`IOptionsMonitorCache<>` 以及 `ILoggerFactory`、`ILogger<>`、`IConfigureOptions<LoggerFilterOptions>` 添加到 `IServiceCollection` 中的过程。有关日志的内容我们会在另一篇文章中介绍。

## 2 Startup 初始化时为什么又能注入又有 `IServiceCollection` 呢

在 `WebHost` 的构建过程中，十有八九会出现 `UseStartup` 这句话（如果不出现这句话，那么很大程度上使用了 `Configure` 扩展方法）。`Startup` 是整个 Web 应用程序的起点。应用程序（Web App）托管在宿主（Hosting Environment）中。那么它应当是在初始化的最终阶段执行的。我们来观察一下它的典型结构：

```cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Add application related services to service collection.
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        // Create application pipeline. We will not focus on this method.
    }
}
```

如果单纯观察上述代码那么并没有任何的稀奇之处。`ConfigureServices` 方法将应用需要的类型全部添加到 `IServiceCollection` 实例中，而 `Configure` 来构建 Pipeline（我们此次不讨论该方法）。但是如果我们需要记录日志，读取配置文件，在应用程序生命周期事件中注册新的处理方法时，我们可以将其直接注入 `Startup` 中。例如：

```cs
public class Startup
{
    readonly IConfiguration configuration;
    readonly IApplicationLifetime lifetime;
    readonly ILogger<Startup> logger;

    public Startup(
        IConfiguration configuration, IApplicationLifetime lifetime, ILogger<Startup> logger)
    {
        this.configuration = configuration;
        this.lifetime = lifetime;
        this.logger = logger;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        // Add application related services to service collection.
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        // Create application pipeline.
    }
}
```

那么问题就来了。

* 在 `Startup` 中注入的 `configuration`、`lifetime`、`logger` 这些服务是由哪一个 `IServiceProvider` 创建出来的呢？
* 如果在 `Startup` 创建时 `IServiceProvider` 已然创建，那么 `Startup.ConfigureServices` 在向哪个 `IServiceCollection` 实例添加类型呢？
* 应用程序运行期间的 `IServiceProvider` 是在 `Startup` 创建之前就创建好的那个呢、还是由 `Startup` 配置的 `IServiceCollection` 实例创建的那个呢？

## 3 两阶段 ServiceProvider 创建

既然 `Startup` 中已经有一个 `IServiceProvider` 来给相应的类型进行依赖注入，而平时的应用程序中的依赖注入又能够包含 `Startup.ConfigureServices` 中的类型定义，那么说明在整个初始化过程中先后创建了两个 `IServiceProvider` 对象。

即 ASP.NET Core 的初始化包含了两个步骤：

* 第一个步骤是 Hosting 相关服务的初始化过程，初始化完毕之后创建了第一个 `IServiceProvider` 对象；
* 第二步是 Application 相关服务的初始化过程。而 Application 的初始化过程可以注入 Hosting 相关的服务。之后，通过 `IStartup.ConfigureServices` 方法创建了第二个 `IServiceProvider` 对象。

> **如果你对源代码感兴趣**
>
> 请参考 `WebHostBuilder` 类的 `Build` 方法（[源代码在这里](https://github.com/aspnet/AspNetCore/blob/master/src/Hosting/Hosting/src/WebHostBuilder.cs)）。大致的过程如下：
>
> * `BuildCommonServices` 方法将所有 Hosting 所需的服务（`WebHost` 相关类型以及所有 `IWebHostBuilder` 调用中添加的服务类型）添加到 `IServiceCollection` 对象中。
> * 使用该 `IServiceCollection` 创建 Hosting 相关的 `IServiceProvider`，不妨称之为 `hostingServiceProvider`。
> * 使用该 `hostingServiceProvider` 创建 `IStartup` 对象（这里有和环境相关的 Convension，详情请参见上一篇）。
> * 使用一个复制的 `IServiceCollection` 对象调用 `IStartup.ConfigureServices` 方法创建另外一个 `IServiceProvider` 不妨称之为 `applicationServiceProvider`。

在了解了上述过程之后，那么我们需要注意些什么呢？

首先我们已经了解，`Startup` 可以使用 Hosting 的 `IServiceProvider` 进行注入。但是 `IServiceProvider` 是一个顶级的 Provider，如果我们在 `Startup` 中创建了一个非常消耗资源的对象（实现了 `IDisposable`），则在默认情况下该对象只有在应用程序彻底退出的时候才会销毁。若显式 `Dispose` 该对象的话且该对象不是 `Transient` Scope。则有可能导致 Defect。

## 4 规避初始化过程中的资源泄露

但是如果我真的需要在初始化的时候注入非常消耗资源的对象，而我又希望规避资源的泄露，我该怎么办呢？其实还是有办法的。那就是不使用 `Startup` 的构造函数进行注入而是直接在 `Configure` 方法中通过参数进行注入。

为什么这种方式可以规避资源泄露呢？因为这种注入机智并非典型的依赖注入机制，而是 ASP.NET Core 特意实现的。如果应用程序在初始化时使用的 `UseStartup<TStartup>()` 中的 `TStartup` 并没有实现 `IStartup` 的话，ASP.NET Core 就会使用基于约定的 `IStartup` 实现对 `TStartup` 进行包装。在包装过程中，它会尝试找到 `TStartup` 类型中的 `Configure` 方法，检查参数表中的参数，并使用 `IStartup.ConfigureServices` 创建的 `IServiceProvider` 进行注入。但是这里的 `IServiceProvider` 却并不初始化过程中的顶级 Provider。而是在将整个方法调用包裹在了 `Scope` 里。因此即使在初始化过程中创建非常消耗资源的实例也会随着方法调用结束后 `Scope` 的 `Dispose` 而销毁。具体代码请参见：[ConfigureBuilder 源代码](https://github.com/aspnet/AspNetCore/blob/master/src/Hosting/Hosting/src/Internal/ConfigureBuilder.cs)

## 5 总结

请飞到文章开头的第 0 节 :-D。

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>