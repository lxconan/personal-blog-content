---
title: "ASP.NET Core 沉思录 - 结构化日志"
date: 2019-03-24 15:26:59
tags:
- aspnetcore
- logging
---

在 《ASP.NET Core 沉思录 - Logging 的两种介入方法》中我们介绍了 ASP.NET Core 中日志的基本设计结构。这一次我们来观察日志记录的格式，并进一步考虑如何在应用程序中根据不同的需求选择不同的日志记录形式。

<!--more-->

太长不读：直接飞到文章最后 :-D

## Microsoft.Extension.Logging 体系下的日志格式

为了便于阅读，我们仍然将 *Microsoft.Extension.Logging* 的基本设计结构放在这里：

<img src="{{root_url}}/images/blog/core-mvc-logger-struct.png" style="text-align:center" alt="logger structure"/>

### 奇怪的不合理之处

> 注：所谓奇怪就是这种不合理是只是从某一种特定视角看的。

对于记录日志而言，虽然一些具体的日志记录目标和记录的格式会有一些联系，但是日志记录的目标和日志记录的格式应该是两件事情。貌似 *Microsoft.Extension.Logging* 在此处进行了一些抽象。首先，日志具体的记录地点和记录格式全部由具体的 `ILoggerProvider` 创建的 `ILogger` 来完成。而对于日志的格式化方法，则使用 `ILogger.Log` 方法中的委托来完成。该委托中包含了一个 `formatter` 委托参数。该委托接收需要记录的对象，关联的异常实例并返回日志字符串。我们可以在其中定义自己的格式化逻辑。总结起来感觉是：

* 特定的 `ILoggerProvider` 创建将日志记录到特定种类的目的地的日志记录器。例如，`ConsoleLogger`。
* 指定 `ILogger.Log` 方法中的 `formatter` 参数对日志对象进行格式化。

`ILogger.Log` 方法除了 `formatter` 之外还包含如下的参数：

* logLevel：日志的级别。
* eventId：当前事件的标识。
* state：日志对象。
* exception：关联的异常对象。

而 `formatter` 参数将使用其中的 `state` 参数和 `exception` 参数对日志进行格式化。这样通过替换 `formatter` 的逻辑就可以更改日志的形式了。例如，使用如下的逻辑就可以将 `state` 格式化为 JSON 形式：

```cs
// Capture output so that we can assert its content
var writer = new StringWriter();
Console.SetOut(writer);

// Normal initialization logic
var serviceCollection = new ServiceCollection();
serviceCollection.AddLogging(
    config => config.SetMinimumLevel(LogLevel.Debug).AddConsole());
ServiceProvider provider = serviceCollection.BuildServiceProvider();
var loggerFactory = provider.GetService<ILoggerFactory>();
var logger = loggerFactory.CreateLogger("category");

// Write log message
logger.Log(
    LogLevel.Information,
    1,
    new {message = "Hello {name}", name = "World"},
    null,
    (state, exception) => JsonConvert.SerializeObject(state));

// This is very important. The console logger using a async processor to consume
// the queued log message.
Thread.Sleep(1000);

Assert.Equal("...(omitted)...", writer.ToString())
```

如果您尝试了上述范例程序就会感到这个设计好像有问题，而如果联系整个 *Extension.Logging* 体系则感觉问题就更大了：

* `formatter `只是解决了日志 message 部分的格式化问题，而无法影响其他信息的格式化，例如 `eventId`、`logLevel`、`exception` 等。
* 我们根本不会用到具体的 `ILoggerProvider` 而是会使用 `ILoggerFactory` 提供的 `Logger` 门面。这个 `Logger` 是一个组合 `Logger`，也就是它会将 `ILogger<>.Log` 调用分发出来。但是我们很少使用 `ILogger<>.Log` 方法，而会使用扩展方法使用 template message 进行日志记录，这意味着所有的子 `ILogger` 实现都会接到同样的 `formatter`。自此，不同的目标采用不同的消息格式的理想破灭了。

总结一下就是，日志的记录目标和日志的格式混合了起来。职责区分不清。message 的格式化职责交给了门面扩展方法；而另一部分格式化职责交给了具体的 `ILoggerProvider`。

### 从另一种视角看的合理之处

我们换一个视角可能就会得到不一样的体验。首先我们更改分析问题的策略。从端到端的角度来思考问题。当我们记录日志的时候希望有哪几类信息呢？日志作为追踪一个事件的依据，应当能够清晰的说明这个事件。那么小学语文老师就告诉过我们，记录一件事情需要有：

* 时间
* 地点
* 人物
* 起因
* 经过
* 结果

如果我们将这些信息归一归类，我们就可以得到这些信息：

* 物理世界的环境参数：时间
* 判断事件严重程度的依据：结果
* 事件过程的上下文参数：地点（例如 URI 或代表某种操作的入口）、人物（例如谁进行的操作）、起因（例如方法调用参数）、经过（例如调用的那个方法）

而要记录这些信息，则可以对应到程序中的以下几种形式的数据：

* 物理世界的环境参数：例如 `DateTime`、`DateTimeOffset`
* 判断事件严重程度的依据：例如 `LogLevel`
* 事件过程上下文的参数：例如事件的类别 `CategoryName`；一个包含各种各样上下文参数的 `object[]` 对象；以及对人类友好，能够将这些参数串起来的消息模板。

至此，你能够看到这些参数正是 *Microsoft.Extension.Logging* 中门面扩展方法中需要你来提供的参数。它本来也没有希望你调用 `ILogger.Log` 方法。而是希望你调用扩展方法用最舒服的方式达成日志记录的目的。

而作为 `ILoggerProvider` 开发者，你并不一定必须得接受 `formatter` 生成的格式化后的日志消息。你可以选择处理每一个传入参数。具体的请参见 `FormattedLogValues` 类型的源代码。

### 阶段性总结

* 日志记录和记叙文一样，只要满足了六要素就可以说清楚一件事情。而记录这六要素的形式正式 *Extension.Logging* 提供给我们的扩展方法的参数形式。
* 分析问题从端到端分析是一种非常靠谱的分析方法。可以避免走弯路。
* `ILogger` 的扩展方法负责生成日志消息，`ILoggerProvider` 和负责记录工作的 `ILogger` 实现负责格式化日志消息并将日志记录到特定的目标上去。

## 利用 SeriLog 实现灵活的日志记录形式

通过上述分析我们应该能够看到这种设计的合理性。但是不争的事实是 `ILoggerProvider` 一系包揽两种职能，并没有进一步抽象，有没有人来对日志记录的目标和日志记录的整体格式进行抽象呢？有！那就是被千万人喜爱的 `SeriLog`。它在 `ILoggerProvider` 一级商抽象了 `ITextFormatter` 解决了这个问题：

> 我在这里不会介绍 SeriLog 的具体使用方法。网上教程一大堆大家去搜搜好了。我建议直接去官网。

例如，我可以将日志记录到 Console 中，默认情况下，这种日志的格式是给人看的：

```cs
// normal initialization logic
var serviceCollection = new ServiceCollection();
serviceCollection.AddLogging(b =>
{
    Logger seriLogger = new LoggerConfiguration()
        .WriteTo.Console()
        .MinimumLevel.Debug()
        .CreateLogger();

    b.AddSerilog(seriLogger);
});
ServiceProvider provider = serviceCollection.BuildServiceProvider();

// create logger
var loggerFactory = provider.GetService<ILoggerFactory>();
var logger = loggerFactory.CreateLogger("category");

// write log
logger.LogInformation("Hello {name}", "world");
```

此时屏幕上会输出高亮版的，适于阅读的日志，类似这样：

> [18:41:27 INF] Hello **world**

但是如果希望使用其他的格式，则可以通过 `ITextFormatter` 快速的转换格式：

```cs
var serviceCollection = new ServiceCollection();
serviceCollection.AddLogging(b =>
{
    Logger seriLogger = new LoggerConfiguration()
        // PLEASE NOTE that we use JsonFormatter as input paramter
        .WriteTo.Console(new JsonFormatter())
        .MinimumLevel.Debug()
        .CreateLogger();

    b.AddSerilog(seriLogger);
});

ServiceProvider provider = serviceCollection.BuildServiceProvider();

var loggerFactory = provider.GetService<ILoggerFactory>();
var logger = loggerFactory.CreateLogger("category");

logger.LogInformation("Hello {name}", "world");
```

这样就会得到以下的日志：

```json
{
    "Timestamp":"2019-03-24T19:38:54.4833240+08:00",
    "Level":"Information",
    "MessageTemplate":"Hello {name}",
    "Properties":{"name":"world","SourceContext":"category"}
}
```

如果还需要 `formatter` 格式化之后的完整消息，可以在创建 `JsonFormatter` 时指定 `new JsonFormatter(renderMessage: true)` 这样就会得到包含完整可读消息的结果：

```json
{
    "Timestamp":"2019-03-24T19:44:51.0430260+08:00",
    "Level":"Information",
    "MessageTemplate":"Hello {name}",
    "RenderedMessage":"Hello \"world\"",
    "Properties":{"name":"world","SourceContext":"category"}
}
```

这样，在实际的操作中。我们可以直接使用 *Microsoft.Extension.Logging* 默认体系对 `ILoggerProvider` 进行扩展达到对记录目标和记录格式的控制；也可以将其与 SeriLog 集成。通过 Sink 和 `ITextFormatter` 组合的方式分别对记录目标和记录格式进行控制。

## 总结

一图胜千言：

<img src="{{root_url}}/images/blog/core-mvc-logger-default.png" style="text-align:center" alt="logger structure default"/>

图-1 *默认 Microsoft.Extension.Logging 类型及信息传递路径*

<img src="{{root_url}}/images/blog/core-mvc-logger-serilog.png" style="text-align:center" alt="logger structure serilog"/>

图-2 *集成 SeriLog 后类型及信息传递路径*

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>