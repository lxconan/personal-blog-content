---
title: 【我好像遇到了假的 C#】2018年06月20日--generic
date: 2018-06-20 07:55:57
tags:
- csharp
- generic
---

Hello 大家好，我们除【我好像遇到了假的 C#】这个系列之外，也在酝酿着一些成系统的系列哦。敬请期待。今天的问题来源于我从 [Eric Lippert](https://blogs.msdn.microsoft.com/ericlippert/2011/02/03/curiouser-and-curiouser/) 的博客上的一篇文章。

<!--more-->

## 上期问题回顾

上期中，我们的问题是。给定以下三种形式的条件编译代码。它们有什么不同呢？我们一个一个来看

方式一：

```cs
#if DEBUG

WriteLog(GetComplicatedDiagnosticData());

#endif // #if DEBUG
```

这种写法是比较理想的。因为在 Release Build 模式下，`GetComplicatedDiagnosticData` 根本不会执行。但是这种写法有它的弱点。即，我们必须在每一处 WriteLog 的地方都使用 `#if ... #endif` 包裹。

方式二：

```cs
void WriteLog(string message) 
{
#if DEBUG
  // Writting log to somewhere
#endif
}
...

WriteLog(GetComplicatedDiagnosticData());
```

这种写法就比较糟糕了。因为不论是那种配置，GetComplicatedDiagnosticData 都会执行。从而可能影响应用程序的性能。

方法三：

```cs
[Conditional("DEBUG")]
void WriteLog(string message) 
{
  // Writting log to somewhere
}

WriteLog(GetComplicatedDiagnosticData());
```

第三种写法看似和第二种一样，实际上编译器却会将代码如第一种写法那样进行处理。因此 `GetComplicatedDiagnosticData` 方法是不会执行的。

> 补充一个小问题，如果采用方法三。并且 `WriteLog` 是一个 `public` 类型的 `public` 方法。并位于程序集 *A* 中。若程序集 *B* 引用了程序集 *A*。并且程序集 *B* 的某个方法调用了 *A* 的 `WriteLog` 方法。那么请问，编译器会执行条件编译吗？

## 本期问题

应用程序中有很多看起来像鸡生蛋蛋生鸡的问题。例如这个：

```cs
class Blah<T> where T : Blah<T> { }
```

首先，上述写法是没有任何语法错误的。这种嵌套的感觉就像是：

<img src="{{root_url}}/images/blog/funny_csharp_generic.jpg" style="text-align:center" alt="re"/>

更变态的是这个：

```cs
class Foo<TA, TB, TC, TD>
{
  class Bar : Foo<Bar, Bar, Bar, Bar>
  {
    Bar.Bar.Bar bar;
  }
}
```

请问 `bar` 的类型是什么呢？

好了本期的内容就到这里。如果你觉得这篇文章很好玩儿，或者对你有帮助。也欢迎你分享给其他的人。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>