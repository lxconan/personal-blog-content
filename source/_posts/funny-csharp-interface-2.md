---
title: 【我好像遇到了假的 C#】2018年06月06日--interface
date: 2018-06-06 00:55:35
tags:
- csharp
- interface
- polymorphism
---

Hello 大家好，经过了周二的电视台休息。我们周三的更新又来了。首先我们来回顾一下上一期的问题。

## 上一期的问题解答

在上一期我们放飞了自我，变成了一只有点儿复杂的鹦鹉。而其中的点主要就是在接口的实现上。（警告：接下来的过程，可能有些绕，有些让人难过😣）

<!--more-->

<img src="{{root_url}}/images/blog/funny_csharp_bird.jpeg" style="text-align:center" alt="bird"/>

接口和类不同，一个类可以同时实现多个接口。但是这也造成了一个问题，如果两个接口的定义签名一致，那么这个签名实现了哪一个接口呢？那么如何调用另一个接口的相同签名的方法呢？当然，解决方案就是接口是可以 *显式实现* 的。如果有

```cs
interface IFlyable {
  string Fly();
}
```

和

```cs
interface ISkyNavigator {
  string Fly();
}
```

那么就可以在实现类上通过显式接口实现来消除这种二义性：

```cs
class Bird : IFlyable, ISkyNavigator {
  public string Fly() => "Bird.Fly()";
  string ISkyNavigator.Fly() => "ISkyNavigator.Fly()";
}

...

var bird = new Bird();
bird.Fly();                  // Bird.Fly()
((IFlyable)bird).Fly();      // Bird.Fly()
((ISkyNavigator)bird).Fly(); // ISkyNavigator.Fly()
```

但是如果有多个类型实现了接口而且类型之间还有继承关系。事情就变得比较复杂了。在最简单的情况下，基类实现了接口，并将方法标记为 `virtual` 的。这样子类就非常容易的重写该实现并实现多态性了。

```cs
public interface IFlyable
{
  string Fly();
}

public class Bird : IFlyable
{
  public virtual string Fly() => "Bird.Fly()";
}

public class Parrot : Bird
{
  public override string Fly() => "Parrot.Fly()";
}
```

这样如果我们进行如下的调用：

```cs
var parrot = new Parrot();
parrot.Fly();                   // Parrot.Fly()
((Bird)parrot).Fly();           // Parrot.Fly()
((IFlyable)parrot).Fly();       // Parrot.Fly()
```

可是，如果基类在实现的时候并没有将其标记为 `virtual`。那么事情就有些微妙了：

```cs
public class Bird : IFlyable
{
  public string Fly() => "Bird.Fly()";
}

public class Parrot : Bird
{
  public string Fly() => "Parrot.Fly()"; // 隐藏了基类的方法
}
```

此时，`Parrot` 隐藏了基类的实现，多态性也就直接破坏了：

```cs
var parrot = new Parrot();
parrot.Fly();                   // Parrot.Fly()
((Bird)parrot).Fly();           // Bird.Fly()
((IFlyable)parrot).Fly();       // Bird.Fly()
```

但是，如果 `Parrot` 同时也实现了 `IFlyable`，那么事情就变得又不一样了：

```cs
public class Parrot : Bird, IFlyable
{
  public string Fly() => "Parrot.Fly()";
}
```

由于 `Parrot` 实现了 `IFlyable`，因此，`((IFlyable)parrot).Fly()` 会调用 `Parrot.Fly` 实现：

```cs
var parrot = new Parrot();
parrot.Fly();                   // Parrot.Fly()
((Bird)parrot).Fly();           // Bird.Fly()
((IFlyable)parrot).Fly();       // Parrot.Fly()
```

可见，结果稍稍不同，但是多态性还是被破坏的状态。

刚才我们展示的是基类隐式实现接口造成的方法覆盖。而如果基类显示实现了 `IFlyable` 接口的话：

```cs
public class Bird : IFlyable
{
  string IFlyable.Fly() => "Bird.Fly()";
}
```

如果 `Parrot` 没有直接声明实现 `IFlyable`，则所有类型转换为 `IFlyable` 的调用都会调用 `Bird` 的相应方法：

```cs
public class Parrot : Bird
{
  public string Fly() => "Parrot.Fly()"; // 和 Bird 的相应方法没有关系
}

...

parrot.Fly();                      // Parrot.Fly()
((IFlyable)((Bird)parrot)).Fly();  // Bird.Fly()
((IFlyable)parrot).Fly();          // Bird.Fly()
```

但是，如果 `Parrot` 声明实现 `IFlyable`，那么由于基类显式实现，不论如何进行类型转换，也不可能调到 `Bird` 中定义的 `Fly` 方法了（但是注意，这和多态还是有区别的。例如，你无法使用 `base` 关键字调用基类的方法。有些书里称之为重新实现）。

```cs
public class Parrot : Bird, IFlyable
{
  public string Fly() => "Parrot.Fly()";
}

...

parrot.Fly();                      // Parrot.Fly()
((IFlyable)((Bird)parrot)).Fly();  // Parrot.Fly()
((IFlyable)parrot).Fly();          // Parrot.Fly()
```

综上所述可以发现如果接口、基类、派生类的所有者并非一个机构那么类型的继承、实现就会出现非常多的问题。有的还会破坏多态性。因此接口的实现了类的继承要特别小心的进行设计。我们在下一期还会继续这个话题。

最后公布一下答案。我们的题目是：

```cs
public interface IFlyable
{
  string Fly();
}

class Bird : IFlyable
{
  string IFlyable.Fly()
  {
    return "Bird.Fly()";
  }
}

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

...

var parrot = new Parrot();
var bird = (Bird)new Parrot();

Console.WriteLine(parrot.Fly());
Console.WriteLine(((IFlyable)bird).Fly());
```

`parrot.Fly()` 直接调用了 `Parrot` 类型的共有方法 `public string Fly()`。因此其输出为 `Parrot.Fly() implicite`。由于 `Parrot` 显式实现了 `IFlyable` 因此 `((IFlyable)bird).Fly()` 实际上使用的还是 `Parrot` 的显式实现。因此输出为 `Parrot.Fly() explicite`。

## 本期的题目

为了巩固一下刚才绕的云里雾里的知识，我们继续行进在放飞自我的路上。假设这次的定义如下：

```cs
public interface IFlyable
{
  string Fly();
}
```

并且有如下实现：

```cs
class Bird : IFlyable
{
  public string Fly()
  {
    return "Bird.Fly()";
  }
}

class Parrot : Bird, IFlyable
{
  string IFlyable.Fly()
  {
    return "Parrot.Fly() explicite";
  }
}
```

那么以下的输出是什么呢？

```cs
var parrot = new Parrot();
var bird = (Bird)new Parrot();

Console.WriteLine(parrot.Fly()); 
Console.WriteLine(((IFlyable)bird).Fly()); 
```

好了，这就是今天的内容了。如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>