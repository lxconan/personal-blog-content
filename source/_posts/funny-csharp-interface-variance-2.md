---
title: 【我好像遇到了假的 C#】2018年06月11日--variance continued
date: 2018-06-11 08:17:17
tags:
- csharp
- interface
- variance
---

Hello 大家好，我们这几天在 variance 的路上越走越远了。今天我们带来的还是另外一个 variance 的问题。首先还是来解决一下上一期的问题吧。

<!--more-->

## 上期问题解答

```cs
class BaseClass { }
class DerivedClassA : BaseClass { }
class DerivedClassB : BaseClass { }

static BaseClass[] ToBaseClassArray(DerivedClassA[] subclassArray)
{
  #region Please modify the code below to pass the test

  return subclassArray;

  #endregion
}

[Fact]
public void should_not_violate_ploymorphism()
{
  DerivedClassA[] subClassArray = {new DerivedClassA()};
  BaseClass[] baseClassArray = ToBaseClassArray(subClassArray);

  var derivedClassB = new DerivedClassB();

  baseClassArray[0] = derivedClassB;

  Assert.Same(derivedClassB, baseClassArray[0]);
}
```

这个测试显然是希望保持基类和派生类的转换规则。虽然上期解释了很多但是做起来还是很简单的，那就是真正的将数组转换为一个 `BaseClass` 数组。确保每一个元素的类型确实是 `BaseClass`。它的做法有很多。例如（忽略所有的参数检查）：

```cs
subclassArray.Cast<BaseClass>().ToArray();
```

## 本期问题

本期我们继续来应用 variance。我们知道 variance 只能够用在接口和委托上。类是不可以的。例如这样写是语法错误：

```cs
class GenericType<out T> { ... }
```

假设现在我们要自己实现一个 **栈** 类：`MyStack<T>`，它的定义是：

```cs
class MyStack<T>
{
  public T Pop() { ... }
  public void Push(T value) { ... }
}
```

单单从这两个方法就可以知道，即使我们抽出一个接口，例如  `IMyStack<T>` 也是不能同时满足 `Pop` 的返回类型和 `Push` 的输入参数 `T` 的。总不能这样写~

```cs
class MyStack<in out T> { ... }    // 这是什么鬼？
```

<img src="{{root_url}}/images/blog/funny_csharp_interface_variance_2.jpg" style="text-align:center" alt="variance"/>

但是我们现在又希望 `MyStack` 类正确的体现相应的逆变和协变功能，那么我们该怎么做呢？

好了，今天的内容就到这里了。祝大家周一愉快（ㄟ( ▔, ▔ )ㄏ）。如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>