---
title: 【我好像遇到了假的 C#】2018年06月14日--unicode
date: 2018-06-13 23:40:55
tags: 
- csharp
- unicode
- string
---

Hello 大家好，本星期我要和娃一起开心的玩耍，所以更新缓慢了些。上一期的问题非常有趣。不知道大家有没有得出自己的结论呢。老规矩我们还是看一下上一期的问题。本期我们转战另一个战场，嗯，字符串。谁知道字符串上可能会发生什么好玩儿的事情呢对吧。

<!--more-->

## 上期问题解答

上期的问题有些强人所难的意思。假设现在我们要自己实现一个 **栈** 类：`MyStack<T>`，它的定义是：

```cs
class MyStack<T>
{
  public T Pop() { ... }
  public void Push(T value) { ... }
}
```

但是我们现在希望 `MyStack` 类在 `Pop` 和 `Push` 方法上正确的体现相应的逆变和协变功能，那么我们该怎么做呢？

首先类是不可能定义逆变和协变的，只能是接口或委托。在这里我们肯定是使用接口了。但是在一个接口上是不可能同时定义类型参数的逆变和协变的。那怎么办呢。这让我想起了火影忍者里面卡卡西教鸣人搓团子……哦不不是搓螺旋丸的时候鸣人抱怨说怎么可能两个眼睛一只向左看一只向右看呢！此时卡卡西道出了解决问题的真谛：

<img src="{{root_url}}/images/blog/funny_csharp_unicode_variance.jpg" style="text-align:center" alt="variance"/>

因此，如果我们也想做一个大玉螺旋丸（`MyStack<T>`）那么我们也来个影分身先（做两个接口，一个协变，一个逆变）：

```cs
interface IPopable<out T> 
{
  T Pop();
}

interface IPushable<in T>
{
  void Push(T value);
}
```

此时，令 `MyStack<T>` 同时实现两个接口：

```cs
class MyStack<T>: IPopable<T>, IPushable<T>
{
  public T Pop() { ... }
  public void Push(T value) { ... }
}
```

这样对于协变来说：

```cs
var stack = new MyStack<Derived>();

// ...

var popable = (IPopable<Derived>)stack;
IPopable<Base> popableWithBase = popable;

Base item = popableWithBase.Pop();   // 完全没有问题
```

而对于逆变来说：

```cs
var stack = new MyStack<Base>();

// ...

var pushable = (IPushable<Base>)stack;
IPushable<Derived> pushableDerived = stack;

pushableDerived.Push(new Derived()); // 完全没有问题
```

## 本期问题

本期我们转战另外一个领域了——字符串。这个好像没有什么奇奇怪怪的东西吧~好了请听题哦：这个题目就是，如何计算字符串中字符的数目 O_o，对你没有听错，请实现如下的函数：

```cs
static int GetCharacterLength(string text)
{
  throw new NotImplementedException();
}
```

使其至少能够通过如下的测试：

```cs
static IEnumerable<object[]> TestCases() =>
  new[]
  {
      new object[]{"", 0},
      new object[]{"12345", 5},
      new object[]{char.ConvertFromUtf32(0x2A601) + "1234", 5}
  };

[Theory]
[MemberData(nameof(TestCases))]
public void should_calculate_text_character_length(string testString, int expectedLength)
{
  Assert.Equal(expectedLength, GetCharacterLength(testString));
}
```

好了，今天的内容就到这里了。如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>