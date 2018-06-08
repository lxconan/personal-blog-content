---
title: 【我好像遇到了假的 C#】2018年06月08日--variance
date: 2018-06-08 01:12:42
tags:
- csharp
- array
- variance
---

Hello，大家好，现在 Infrastructure 已经搞定了。终于可以安心的更新了。我们先来回顾一下上一期的问题。然后我们会带来一个脑筋急转弯似的新问题：

<!--more-->

## 上一期问题的解答

上一期的问题强化了 `interface` 实现上的知识点。给定

```cs
public interface IFlyable
{
  string Fly();
}

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

`parrot.Fly()` 是无法调用显式实现的，因此它调用的是 `Bird.Fly()`。因此第一行的输出为 `Bird.Fly()`。而第二行中，`bird` 的 most derived 类型为 `Parrot` 因此其接口转换的调用会调用 `Parrot` 的显式实现。因此输出为 `Parrot.Fly() explicite`。

从这两期的例子我们可以看到。接口实现是有各种各样的问题的。尤其是我们不希望子类重新实现接口方法（某些情况下则是直接隐藏基类的接口方法）。那么如果我们希望在基类中隐式实现成员，则可以将成员标记为 `virtual`，而如果要显式实现成员，则可以使用如下的模式：

```cs
public class BaseClass : IBehavior
{
  void IBehavior.Perform() => Perform();
  protected virtual void Perform() => ... /* do actual work */;
}
```

这样子类可以直接重写虚方法：

```cs
public class Derived : BaseClass
{
  protected override void Perform() => ... /* overrided work */;
}
```

这样即便是 `Derived` 类型进行了显式接口调用也会由于 `virtual protected` 方法的存在而体现多态性。

## 本期的题目

<img src="{{root_url}}/images/blog/funny-csharp-array-variance.jpg" style="text-align:center" alt="variance"/>

假设有如下的类型继承结构：

```cs
class BaseClass { }
class DerivedClassA : BaseClass { }
class DerivedClassB : BaseClass { }
```

假设给定一个基类型的数组：

```cs
// 不管用什么方法，得到了一个基类型的数组，而且确定至少包含两个元素
BaseClass[] baseClassArray = ...; 
```

那么以下的测试一定能够成功吗？

```cs
var derivedClassA = new DerivedClassA();
var derivedClassB = new DerivedClassB();
baseClassArray[0] = derivedClassA;
baseClassArray[1] = derivedClassB;
Assert.Same(derivedClassA, baseClassArray[0]);
Assert.Same(derivedClassB, baseClassArray[1]);
```

好了，这就是今天的内容了。如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>