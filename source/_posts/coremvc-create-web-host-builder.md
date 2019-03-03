---
title: "ASP.NET Core 沉思录 - CreateWebHostBuilder 是一个 Convension"
date: 2019-03-02 10:01:13
tags:
- aspnetcore
- testing
---

失踪人口回归。去年六月份开始，我开始翻译一千多页的《CSharp 7 in a Nutshell》到现在为止终于告一段落。我又回归了表世界。从这次开始我希望展开一个全新的主题。我叫它 ASP.NET Core 沉思录（多么高大上的名字，自我陶醉~）。今天是第一个主题。`CreateWebHostBuilder` 是一个 Convension。

<!--more-->

## 太长不读

对于 `WebApplicationFactory<T>` 而言，默认情况下会采取如下假定：

* `Startup` 所在的程序集应当就是应用程序入口（`Main`）所在的程序集；（官方工程模板的坑）
* 应用程序入口所在的类（`Program`），里面会包含整个创建和配置 `IWebHostBuilder` 的过程；
* 创建和配置 `IWebHostBuilder` 的过程是由应用程序入口所在类的 `CreateWebHostBuilder` 方法完成的。

在满足上述假定的情况下，无需额外代码，Web 应用的执行和测试将共享相同的逻辑。如若不然，则测试失败。如果无法满足上述三种条件还可以通过集成 `WebApplicationFactory<T>` 并重写 `CreateWebHostBuilder` 方法来解决。

以上约束仅仅限定于 `WebApplicationFactory<T>`，若直接在测试中使用 `TestServer` 则没有这种限制。

`WebApplicationFactory<T>` 的 `T` 并不是 `TStartup`，而是应用程序入口所在的程序集中的任意类型。

## 娓娓道来

如果我们使用 dotnet 命令行创建一个 ASP.NET Core MVC/WebAPI 的工程。那么它的启动代码大概是这样的：

```cs
public static class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args)
    {
        // Modified a little bit for the sake of illustration
        return new WebHostBuilder()
            .UseKestrel()
            .ConfigureLogging(lb =>
            {
                lb.SetMinimumLevel(LogLevel.Debug).AddConsole();
            })
            .UseStartup<Startup>();
    }
}
```

有没有小伙伴好奇，为什么需要一个 `CreateWebHostBuilder` 方法？从直观上看，它是创建并完成基本的 `IWebHostBuilder` 配置的方法。这个方法应在测试中进行复用以确保测试和应用程序中的 `IWebHostBuilder` 配置几乎相同，例如：

```cs
[Fact]
public async Task should_get_response_text()
{
    IWebHostBuilder webHostBuilder = Program.CreateWebHostBuilder(Array.Empty<string>());

    using (var testServer = new TestServer(webHostBuilder))
    using (HttpClient client = testServer.CreateClient())
    {
        HttpResponseMessage response = await client.GetAsync("/message");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        Assert.Equal("Hello", await response.Content.ReadAsStringAsync());
    }
}
```

这个测试是可以顺利通过的。但是我们认为将 `Program.CreateWebHostBuilder` 暴露并不是一个好的感觉。我们更希望把这个配置过程分离。例如分离到一个类中：

```cs
public class WebHostBuilderConfigurator
{
    public IWebHostBuilder Configure(IWebHostBuilder webHostBuilder)
    {
        return webHostBuilder
            .UseKestrel()
            .ConfigureLogging(lb =>
            {
                lb.SetMinimumLevel(LogLevel.Debug).AddConsole();
            })
            .UseStartup<Startup>();
    }
}
```

这样，`Program` 仅仅包含整个应用程序的入口，`CreateWebHostBuilder` 方法就被删掉了：

```cs
public static void Main(string[] args)
{
    var webHostBuilder = new WebHostBuilder();
    new WebHostBuilderConfigurator().Configure(webHostBuilder).Build().Run();
}
```

测试也就变成了：

```cs
[Fact]
public async Task should_get_response_text()
{
    IWebHostBuilder webHostBuilder = new WebHostBuilderConfigurator().Configure(new WebHostBuilder());

    using (var testServer = new TestServer(webHostBuilder))
    using (HttpClient client = testServer.CreateClient())
    {
        HttpResponseMessage response = await client.GetAsync("/message");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        Assert.Equal("Hello", await response.Content.ReadAsStringAsync());
    }
}
```

看起来不错，测试也通过了真是可喜可贺。现在我们准备使用更加完善的 `WebApplicationFactory<T>` 代替 `TestServer` 进行测试：

```cs
[Fact]
public async Task should_get_response_text_using_web_app_factory()
{
    using (var factory = new WebApplicationFactory<Startup>().WithWebHostBuilder(
        wb => new WebHostBuilderConfigurator().Configure(wb)))
    using (HttpClient client = factory.CreateClient())
    {
        HttpResponseMessage response = await client.GetAsync("/message");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        Assert.Equal("Hello", await response.Content.ReadAsStringAsync());
    }
}
```

看起来不错，但是发现测试运行的时候却失败了。并伴有诡异的异常信息：

```
System.InvalidOperationException : No method 'public static IWebHostBuilder CreateWebHostBuilder(string[] args)' found on 'WebApp.Program'. Alternatively, WebApplicationFactory`1 can be extended and 'protected virtual IWebHostBuilder CreateWebHostBuilder()' can be overridden to provide your own IWebHostBuilder instance.
```

哦，真神奇，它怎么找到 `WebApp.Program` 的？我只告诉了它 `Startup` 而并没有提供任何 `Program` 类型的信息啊？而这个时候，如果我们老老实实的恢复 `WebApp.Program` 类中的 `CreateWebHostBuilder` 方法，那么测试就顺利通过了。

这是为什么呢？原来让测试环境尽可能的 Match 执行环境是我们共同的心愿，WebApplicationFactory<T> 希望能够自动的帮我们解决这个问题，于是它做了如下的假定：

* `Startup` 所在的程序集应当就是应用程序入口（`Main`）所在的程序集；
* 应用程序入口所在的类（`Program`），里面会包含整个创建和配置 `IWebHostBuilder` 的过程；
* 创建和配置 `IWebHostBuilder` 的过程是由应用程序入口所在类的 `CreateWebHostBuilder` 方法完成的。

只要符合这三个假定，那么你尽可不费吹灰之力就达到了产品测试配置一致的目的。而如果不符合这个假定将让测试在默认状态下执行失败。具体的代码请参考 [这里](https://github.com/aspnet/AspNetCore/blob/v2.2.1/src/Mvc/src/Microsoft.AspNetCore.Mvc.Testing/WebApplicationFactory.cs#L278) 和 [这里](https://github.com/aspnet/AspNetCore/blob/v2.2.1/src/Shared/Hosting.WebHostBuilderFactory/WebHostFactoryResolver.cs#L14)。从 `WebHostFactoryResolver` 里面可以看出，除了 `CreateWebHostBuilder` 方法之外，`BuildWebHost` 也是一个 Convension，只不过主要是为了向前兼容的目的。

在真实的项目中，很可能是不满足这三个条件的，那么怎么办呢？还好我们可以通过集成 `WebApplicationFactory<T>` 并重写 `CreateWebHostBuilder` 方法来解决这个问题：

```cs
public class MyWebApplicationFactory : WebApplicationFactory<Startup>
{
    protected override IWebHostBuilder CreateWebHostBuilder()
    {
        var webHostBuilder = new WebHostBuilder();
        new WebHostBuilderConfigurator().Configure(webHostBuilder);
        return webHostBuilder;
    }
}
```

并相应的将测试更改为：

```cs
[Fact]
public async Task should_get_response_text_using_web_app_factory()
{
    using (var factory = new MyWebApplicationFactory())
    using (HttpClient client = factory.CreateClient())
    {
        HttpResponseMessage response = await client.GetAsync("/message");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        Assert.Equal("Hello", await response.Content.ReadAsStringAsync());
    }
}
```

就可以了。

最后，需要提醒的是 `WebApplicationFactory<T>` 的 `T` 是 `TEntryPoint` ，是入口所在的程序集的类型。虽然平常大家都喜欢写 `Startup`。

## 总结

请飞到文章开头~ :-D