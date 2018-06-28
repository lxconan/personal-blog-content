---
title: 【我好像遇到了假的 C#】2018年06月29日--datetime
date: 2018-06-29 08:05:08
tags:
- csharp
- generic
- datetime
---

<img src="{{root_url}}/images/blog/funny_csharp_datetime.jpg" style="text-align:center" alt="cui"/>

大家好，我上周去了异世界。这周我又回来了。时间又继续开始运行了。我们先回顾一下上次的问题吧。然后会按照惯例带来一个新的问题。今天的问题，是关于万年坑，日期和时间的。

<!--more-->

## 上期问题回顾

上期的问题，是给定以下的代码。

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

这个问题实际上是一个编译期的类型展开。首先我们考虑稍微简单一点的情况：

```cs
class Foo<TA, TB, TC, TD>
{
  class Bar : Foo<Bar, Bar, Bar, Bar>
  {
    Bar bar;
  }
}
```

此时，`Bar` 是 `Foo<Bar, Bar, Bar, Bar>` 的派生类，而其中的类型参数的类型为 `Foo<TA, TB, TC, TD>.Bar`，因此不难得到展开。（我竟然也用了 “不难”，这个词，其实我更想用 “显然”，啊哈哈哈~）

```cs
class Foo<TA, TB, TC, TD>
{
  class Bar : Foo<
                   Foo<TA, TB, TC, TD>.Bar,
                   Foo<TA, TB, TC, TD>.Bar,
                   Foo<TA, TB, TC, TD>.Bar,
                   Foo<TA, TB, TC, TD>.Bar
                 >
  {
    Foo<
      Foo<TA, TB, TC, TD>.Bar,
      Foo<TA, TB, TC, TD>.Bar,
      Foo<TA, TB, TC, TD>.Bar,
      Foo<TA, TB, TC, TD>.Bar
    >.Bar bar
  }
}
```

因此，如果继续深入一层：

```cs
class Foo<TA, TB, TC, TD>
{
  class Bar : Foo<Bar, Bar, Bar, Bar>
  {
    Bar.Bar bar;
  }
}
```

则展开为：

```cs
class Foo<TA, TB, TC, TD>
{
  class Bar : Foo<
                   Foo<TA, TB, TC, TD>.Bar,
                   Foo<TA, TB, TC, TD>.Bar,
                   Foo<TA, TB, TC, TD>.Bar,
                   Foo<TA, TB, TC, TD>.Bar
                 >
  {
    Foo<
      Foo<
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar
      >.Bar,
      Foo<
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar
      >.Bar,
      Foo<
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar
      >.Bar,
      Foo<
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar,
        Foo<TA, TB, TC, TD>.Bar
      >.Bar
    >.Bar bar;
  }
}
```

如果我这么写下去的话，如果有人给我稿费的话，我分分钟就又成为了人生赢家！但是没有人给我稿费。而且写得太长大家也不会喜欢看。因此三层展开式大家就脑补吧~

## 本期问题

本期问题非常简单，给定两个 `DateTime` 请求出它们的之间的间隔：

```cs
public static TimeSpan(DateTime end, DateTime start)
{
  // TODO: 成为人生赢家吧
  throw new NotImplementedException();
}
```

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>