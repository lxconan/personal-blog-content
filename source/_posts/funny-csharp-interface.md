---
title: 【我好像遇到了假的 C#】2018年06月04日--interface
date: 2018-06-04 00:48:13
tags:
- csharp
- interface
- polymorphism
---

Hello，大家好，又见面了。不知道什么时候才能添加评论~嗯，所以，有事儿可以知乎（https://www.zhihu.com/people/liu-xia-80/activities） 上找，但是我也不能保证及时回复~x_x。

我们还是先来看看上一期的问题吧。

<!--more-->

## 上一期的问题解答

我们还是要把问题贴一下。这样大家就不用来回翻了（我真是个好人）。上一期的问题是有一个介绍加入非法组织的代理：

```cs
class Akatsuki 
{
  Action<StringBuilder> basicAction;
  Action<StringBuilder> subscribedAction;

  public Akatsuki(
    ref Action<StringBuilder> basicAction,
    ref Action<StringBuilder> subscribedAction)
  {
    this.basicAction = basicAction;
    this.subscribedAction = subscribedAction;

    // (1) basicAction += subscribedAction;
    // (2) this.basicAction += this.subscribedAction;
  }
}
```

加入组织的代码如下：

```cs
Action<StringBuilder> basic = sb => {};
Action<StringBuilder> subscribed = sb => { sb.Append("I have joined -_<"); };
new Akatsuki(ref basic, ref subscribed);

var finalMessageBuilder = new StringBuilder();
basic.Invoke(finalMessageBuilder);
Console.WriteLine(finalMessageBuilder.ToString());
```

其中（1）和（2）是两种不同的实现，每次只能选择其一。那么（1）和（2）的输出分别是什么呢？

这道题目里面埋了几个坑，我们一起分析一下。对于（1），也就是：

```cs
basicAction += subscribedAction;
```

我们先得分清楚，谁是谁。首先 `basicAction` 指的是参数（argument），而不是 `this.basicAction` 字段（field）。而同理 `subscribedAction` 也是参数。由于 `basicAction` 是用 ByRef 的参数传递，因此 `+=` 真正的改变了传入的 `subscribed` 引用。因此第（1）种情况下，你顺利的加入了“晓”这个组织。

<img src="{{root_url}}/images/blog/funny_csharp_uqiha.jpeg" style="text-align:center" alt="org"/>

然后我们再来看看（2）

```cs
this.basicAction += this.subscribedAction;
```

首先 `basicAction` 是字段（field），并且就是一个普通的引用。那么问题就在于 `+=` 这个操作对于委托（delegate）来说到底是一个修改，还是创建一个新的对象了。来 Reference：

https://docs.microsoft.com/en-us/dotnet/api/system.delegate?view=netframework-4.7.2

> Delegates are immutable; once created, the invocation list of a delegate does not change.（Delegate 是一个不可变的对象。一旦创建，其调用列表就不会再改变了。）

因此，`+=` 操作不是一个更新操作。那么显然（2）什么都不会输出。这个坑很有意思，很多老手一不小心也能栽进去。看来组织也不是随便进的。

## 本期的题目

假设我们要放飞自我，就要做只鸟。但是是一个有些复杂的鸟。首先我们有接口：

```cs
public interface IFlyable
{
  string Fly();
}
```

然后我们供奉了祖宗

```cs
class Bird : IFlyable
{
  string IFlyable.Fly()
  {
    return "Bird.Fly()";
  }
}
```

最后，到了我们自己这一代，**飞**这件事情搞得有点复杂：

```cs
class Parrot : Bird, IFlyable
{
  string IFlyable.Fly()
  {
    return "Parrot.Fly() explicite";
  }

  public string Fly()
  {
    return "Parrot.Fly() implicite";
  }
}
```

你猜，以下程序的输出是什么呢？（温馨提示：正确的食用姿势是先用我们的神经网络运行一下这个程序，然后再上机器验证。不过基本上你现在应该在地铁上/餐馆里…身边没有电脑，我也是多虑了）

```cs
var parrot = new Parrot();
var bird = (Bird)new Parrot();

Console.WriteLine(parrot.Fly());
Console.WriteLine(((IFlyable)bird).Fly());
```

好了，这就是今天的内容了。如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>