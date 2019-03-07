---
title: "ASP.NET Core 沉思录 - 环境的思考"
date: 2019-03-06 22:36:57
tags:
- aspnetcore
- environment
---

今天我们来一起思考一下如何在不同的环境应用不同的配置。这里的配置不仅仅指 `IConfiguration` 还包含 `IWebHostBuilder` 的创建过程和 `Startup` 的初始化过程。

<!--more-->

## 0 太长不读

* 环境造成的差异在架构中基本体现在 Infrastructure 中的各个 Adapter 中。而不应当入侵应用程序内部
* 在 ASP.NET Core 中我们需要考虑如何将这些 Adapter（一）放在 service collection 中 （二）（可选）添加到 pipeline 中。
* ASP.NET Core 默认提供了一系列手段来判断当前的环境，只不过这些手段的设计奇怪且不完整。
* `IWebHostBuilder` 的配制方法大多和环境相关，但 `UseSetting` 和环境无关。
* 我们应当应用开闭原则，将相同环境的配置聚合起来，不同环境的配置进行统一抽象。方便维护和扩展。
* 当我们进行设计的时候，需要注意不要将思路局限在 Framework 的设计上，而应当切实考虑我们真正希望解决的问题。

## 1 架构层面的思考

Web Service 的开发和部署过程会涉及若干环境。总的来说可以分为开发环境和部署环境。而部署环境往往又分为 QA、Stage 和 Production 等。对于不同的环境，应用程序可能需要应用不同的配置或实现。还是回到架构的层面上，如下图：

<img src="{{root_url}}/images/blog/core-mvc-architect.png" style="text-align:center" alt="clean architecture"/>

那么这种不同应该体现在架构的哪一个层面上呢？应当让这些不同体现在 *Infrastructure* 的那些 *Adapters* 上。因为 *Adapter* 是其中直接和环境相关的部分。

用一个典型的例子来表示。假定一个注册用户 Account 的业务。在 Application Service 层面，我们提供了如下的接口：

```cs
public class AccountRegistrationService {
    public AccountRegistrationResult Register(AccountRegistrationRequest request) {
        Account account = this.repository.CreateDetached();
        // initialize account from request
        account.Save();
        return AccountRegistrationResult.Create(account);
    }
}
```

在 Domain 层面我们有代表领域对象 Account 的类型 `Account`。`Account` 类型的 Save() 方法可以保存账户信息，其中的实现类似：

```cs
public class Account {
    ...

    public void Save() {
        this.repository.Save(this);
    }
}
```

而其中的 `repository` 则依赖 UnitOfWork 而 UnitOfWork 则可能依赖于具体的持久化实现或者依赖于其他远程服务：

```cs
public class AccountRepository {
    readonly IUnitOfWork session;

    public Account CreateDetached() {
        return new Account(this);
    }

    public void Save(Account account) {
        this.session.RegisterNew(account);
    }
}
```

在这个例子中，`AccountService` 属于 Application Service 层面，`Account` 和 `AccountRepository` 则属于 Domain 层面。这两层的依赖关系是 Application Service 依赖于 Domain。而 Domain 中的 [UnitOfWork](https://martinfowler.com/eaaCatalog/unitOfWork.html) 则是一个接口。假设我们需要将数据写入数据库。则这个接口的实现需要持久化的支持例如它需要使用特定的 `IDbConnection` （Adapter）。即 IUnitOfWork 的实现位于 Infrastructure 层，并在 Infrastructure 层调用 Adapter 向 DB 中写入信息。

而对于不同的环境则可以使用不同的实现，例如，对于运行单元测试的环境，我们不妨叫她 Test 环境。这个 DB 很有可能是一个 in memory 的 SQLite 数据库。而在生产环境则是 MySQL 的集群。

应用程序的内部逻辑最终全部依赖与特定的抽象或接口。它们全部严密的包裹在 Infrastructure 之中，并和外部环境完全隔离。而 Infrastructure 中的 Adapter 则负责联系外部环境。综上所述，环境相关的变化应当全部封闭在 Infrastructure 中。

## 2 ASP.NET Core 中的对应关系

ASP.NET Core 应用程序中的组件的初始化由两个部分构成，第一个部分就是将组件中的类型添加到依赖注入的 `IServiceCollection` 实例中，以便进行创建；第二个部分（可选）即将组件通过 `IApplicationBuilder` 添加到应用程序的处理流水线中。我们一个一个来思考。

### 2.1 依赖注入

ASP.NET Core Web Application 中用依赖注入来决定某种抽象的实现类型。但需要指出的是 ASP.NET 应用程序的依赖注入是分两个阶段进行的。（我们将在另外一篇中介绍），简单来说 ServiceCollection 的构建分为两个部分：

* 为了构建宿主环境而添加的类型；（Infrastructure 层）
* 为了应用程序本身而添加的 Framework（例如 MvC）和各种业务类型。（Infrastructure 层，Application + Domain 层）。

而和环境相关的部分主要位于 “为了构建宿主环境而添加的类型” 中。这一部分的代码属于在 `IStartup` 初始化之前的 WebHostBuilder 构建代码中。一般来说，我们习惯于将 `UseStartup` 调用放在 `IWebHostBuilder` 实例创建的最后，那么也就是 `UseStartup` 之前的代码：

```cs
public static IWebHostBuilder CreateWebHostBuilder(string[] args)
{
    return new WebHostBuilder()
        .UseKestrel()
        .ConfigureLogging(...)
        //
        // The configurations before UseStartup are environment specific
        //
        .UseStartup<Startup>();
}
```

### 2.2 流水线

在流水线配置中主要考虑的是 Web 输入输出上的的变化。例如 Production 环境需要配置 SSL，消除敏感 Header，消除详细的 Error Information 等等。

将组件配置到应用程序的流水线的操作是在 `IStartup` 接口的实现中进行的。定义 IStartup 接口实现的方式大体有两种，第一种是调用 `WebHostBuilderExtensions.Configure` 方法，另一种是使用 `WebHostBuilderExtensions.UseStartup` 方法。不论使用何种方式最终都会归结到对 `IApplicationBuilder` 的操作：

```cs
public void Configure(IApplicationBuilder app) {
    // building pipeline
}
```

在这个时候，宿主初始化相关的类型已经全部可以使用了。因此取用环境相关的信息（环境类型，配置等）就更方便了。

## 3 落地

ASP.NET Core 对这个环节的设计很奇怪。一方面，它提供了非常底层的基于 `IHostingEnvironment.EnvironmentName` 的值来进行环境区分的方法。例如，官方范例中往往会使用如下的代码：

```cs
new WebHostBuilder()
    .UseKestrel()
    .ConfigureLogging((context, logBuilder) => {
        if (context.HostingEnvironment.IsDevelopment()) {
            ...
        }
        else if (context.HostingEnvironment.IsProduction()) {
            ...
        }
        else {
            ...
        }
    })
    ...
```

而另一方面却又在 `Startup` 上设计了命名的 Convension。例如：

```cs
class DevelopmentStartup {}     // for Development
class ProductionStartup {}      // for Production
class Startup {}                // fallback

...

webHostBuilder.UseStartup(assemblyName);
```

又例如：

```cs
class Startup  {
    public void ConfigureServices(IServiceCollection services) { }
    public void ConfigureStagingServices(IServiceCollection services) { }
    public void Configure(IApplicationBuilder app, IHostingEnvironment env) { }
    public void ConfigureStaging(IApplicationBuilder app, IHostingEnvironment env) { }
}
```

这些设计差异很大且每一个都不彻底。而在实际项目中环境属于一个扩展点；而每一套环境的各项配置应当是内聚的。因此上述几种方式或多或少会增加维护上的成本。而较好的设计应当针对如下三个问题：

* 能够立刻说出，我的系统支持几种环境；
* 每一种环境的各种类型的配置（例如，配置源、日志记录、HTTP Client、数据库）是什么样子的，有什么差异；
* 能不能用两步添加一个新的环境：第一，一次性创建一个新环境的所有配置，第二，将这个环境纳入到系统初始化过程中。

为了达到这个要求，需要考虑统一的实现手段。

### 3.1 在 WebHost 开始构建之前我们并不能确定环境信息

一个最简单的想法就是根据不同的环境采取两种完全不同的 `WebHostBuilder` 配置流程。例如：

```cs
WebHostBuilder builder = new WebHostBuilderFactory().Create(env.EnvironmentName);
```

遗憾的是这种设计本身是有问题的。首先，若干环节都可以影响环境的最终确定，包括：

* 当前 Session 的 `ASPNETCORE_ENVIRONMENT` 的值；（请参见 [这里](https://github.com/aspnet/AspNetCore/blob/master/src/Hosting/Hosting/src/WebHostBuilder.cs#L44)）
* Properties/launchSettings.json 中选定 Profile 中 `ASPNETCORE_ENVIRONMENT` 的值（如果用 `dotnet run` 命令执行的话）
* `WebHostBuilder.UseEnvironment(name)` 的参数值；
* `WebHostBuilder.UseSetting(key, value)` 当 `key` 为 `WebHostDefaults.EnvironmentKey` 时的值。
* 若 Host 在 IIS 中，则 web.config 中关于 environmentVariable 的设置。

因此只有在 `WebHostBuilder` 开始 `Build` 时，我们才可以最终确定环境名称。

### 3.2 `UseSetting` 并不是环境相关的

另一种方案是包装 IWebHostBuilder 使其能够依据环境做出相应的 Dispatch。例如：

```cs
abstract class EnvironmentAwareWebHostBuilder : IWebHostBuilder {
    IWebHostBuilder UnderlyingBuilder { get; }
    protected abstract bool IsSupported(IHostingEnvironment hostingEnvironment);

    protected EnvironmentAwareWebHostBuilder(IWebHostBuilder underlyingBuilder)
    {
        // Validation omitted
        UnderlyingBuilder = underlyingBuilder;
    }

    // ...
}
```

从而我们可以分别为不同的环境进行相应的配置。以 `ConfigureService` 方法为例：

```cs
public IWebHostBuilder ConfigureServices(Action<IServiceCollection> configureServices)
{
    UnderlyingBuilder.ConfigureServices(
        (context, services) =>
        {
            if (!IsSupported(context.HostingEnvironment)) { return; }

            configureServices(services);
        });
    return this;
}
```

按照上述方式包装 `ConfigureAppConfiguration`，这样就可以构造以下的扩展方法：

```cs
public static IWebHostBuilder UseEnvironment(
    this IWebHostBuilder builder,
    string environmentName, 
    Action<IWebHostBuilder> configureBuilder)
{
    bool IsEnvironmentSupported(IHostingEnvironment h) => 
        h.IsEnvironment(config.environmentName);

    EnvironmentAwareWebHostBuilder environmentAwareBuilder =
        new DelegatedWebHostBuilder(builder, IsEnvironmentSupported);
    config.configureBuilder(environmentAwareBuilder);

    return builder;
}
```

这种方案下的 `WebHostBuilder` 初始化逻辑就变成了：

```cs
webHostBuilder
    .UseEnvironment("Development", wb => {
        wb
            .ConfigureService((ctx, cb) => { ... })
            .ConfigureLogging((lb) => { ... })
            ...
    })
    .UseEnvironment("Production", wb => {
        // configure for production
    });
```

这样我们至少就可以用若干扩展方法类将不同环境完全分开了。但是这个实现方案是有问题的：`UseSetting` 方法。`IWebHostBuilder` 所公开的方法中除了 `Build`、`ConfigureServices` 和 `ConfigureAppConfiguration` 之外还有第四个方法：`UseSetting`。和上述 `ConfigureXxx` 方法不同，`UseSetting` 方法执行完毕之后其影响马上生效，而且该方法无法根据不同的环境作出变化。即，如果我们使用了：

```cs
webHostBuilder
    .UseEnvironment("Development", wb => wb.UseSetting("Foo", "Bar"))
    .UseEnvironment("Production", wb => wb.UseSetting("Foo", "O_o"));
```

且当前环境为 `Development` 则 `IConfiguration` 实例的 `"Foo"` 对应的值为 `"O_o"`。这就会造成混淆。

### 3.3 还是从扩展点来思考

从第 2 节的论述中我们已经知道和环境相关的配置可能存在于宿主环境初始化过程中，也可能存在 `Startup` 初始化过程中（即 `WebHost.Run` 方法执行过程中）。因此我们必须综合考虑这两个部分，但是这个两个部分天生是不同的。那么强行进行统一也是不合适的。

根据开闭原则，我们还是应该从扩展点上来考虑。首先我们能够确定我们的 Adapter 有哪些。又有哪一些 Adapter 是和环境相关的。例如我们和环境相关的 Adapter 有 DB，配置文件加载，日志记录，`HttpClient`（在非 `Development` 环境中我们可能需要进行客户端证书验证），在流水线创建过程中需要根据环境配置是否需要 HTTPS 强制跳转，需要配置错误信息的详细程度等等。在梳理好这些内容后我们就能有针对性的创建方法对各个部分进行配置了，我们可以使用工厂模式：

```cs
class WebHostConfigureFactory {
    ...

    public IWebHostConfigurator Create(string environmentName) {
        return cachedConfigurators[environmentName];
    }
}
```

而每一个 `IWebHostConfigurator` 中都包含了所有的环境相关配置：

```cs
interface IWebHostConfigurator {
    void AddDatabase(IHostingEnvironment environment, IServiceCollection services);
    void LoadConfiguration(IHostingEnvironment environment, IConfigurationBuilder configBuilder);
    void ConfigureLogging(IHostingEnvironment environment, ILoggingBuilder loggingBuilder);
    void AddHttpClient(IHostingEnvironment environment, IServiceCollection services);
    void ConfigureHttpsRedirection(IHostingEnvironment environment, IConfiguration configuration, IApplicationBuilder builder);
    void ConfigureErrorHandler(IHostingEnvironment environment,  IConfiguration configuration, IApplicationBuilder builder);
}
```

而这样我们为各个环境的扩展点建立了抽象，从而统一配置过程：

```cs
static IWebHostBuilder CreateWebHostBuilder() {
    return new WebHostBuilder()
        .UseKestrel()
        //
        // Common configurations
        //
        .ConfigureServices((context, services) => {
            IWebHostConfigurator configurator = factory.Create(context.HostingEnvironment.EnvironmentName);

            configurator.AddDatabase(context.HostingEnvironment, services);
            configurator.AddHttpClient(context.HostingEnvironment, services);
        })
        .ConfigureLogging((context, logBuilder) => {
            factory
                .Create(context.HostingEnvironment.EnvironmentName)
                .ConfigureLogging(context.HostingEnvironment, logBuilder);
        })
        .UseStartup<Startup>();
}

...

class Startup {
    ...

    public void Configure(IApplicationBuilder app) {
        IWebHostConfigurator configurator = factory.Create(hostingEnvironment.EnvironmentName);
        configurator.ConfigureHttpsRedirection(hostingEnvironment, configuration, app);
        configurator.ConfigureErrorHandler(hostingEnvironment, configuration, app);

        // Common configurations
    }
}
```

## 4 总结

请跳到文章开头 :-D

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>