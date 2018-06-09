---
title: 【我好像遇到了假的 C#】2018年06月10日--variance
date: 2018-06-10 22:55:05
tags:
- variance
- interface
- csharp
---

Hello 大家好。北京今天下雨了，气温从 35 度降到了 20 度，感觉舒适了不少。明天估计会有不少同学趁着凉爽出门吧。出门前也别忘了用 5 分钟看段子哦。我们先来回顾一下上一次的脑筋急转弯吧：

<!--more-->

## 上一期问题的解答

假设有如下的继承关系：

```cs
class BaseClass { }
class DerivedClassA : BaseClass { }
class DerivedClassB : BaseClass { }
```

并且，不管用什么方法，得到了一个基类型的数组，而且确定至少包含两个元素：

```cs
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

首先正确的答案是经典的马克思主义式答案：*It depends...*

为什么呢？取决于你怎么得到这个数组的。例如我们可以这样得到基类型的数组：

```cs
var derivedTypeArray = new DerivedClassA[2];
var baseClassArray = (BaseClass[])derivedTypeArray;
```

不论如何进行强制类型转换（注意，这里就是强制类型转换，而不包括值转换）。它的❤️都是 `DerivedClassA` 类型的，因此如果你想用 `DerivedClassB` 的实例对 `DerivedClassA` 类型的项赋值，那么就会产生运行时错误。

```
System.ArrayTypeMismatchException: Attempted to access an element as a type incompatible with the array.
```

希望这对你来说是个惊喜。但是这个转换非常古怪，它为什么能够从语法上成功呢？其他的集合也可以这样么？其实，如果你想对 `List<T>` 做这样的事在类型转换的时候就分分钟死给你看了，不用等到赋值的时候。那么它们的区别在哪里呢？

<img src="{{root_url}}/images/blog/funny_csharp_variance_interface_type.jpg" style="text-align:center" alt="variance"/>

在 .NET Framework（嗯嗯，就是那个只支持 Windows 的那个 Framework）4.0 里引入了对泛型的逆变和协变的支持。这样就可以允许泛化接口和泛型（尤其是框架定义的哪些类型，例如 `IEnumerable<T>`） 像人们期待的那样工作了。例如，我完全可以将一个 `IReadOnlyList<DerivedClassA>` 类型转换为一个 `IReadOnlyList<BaseClass>`，并使用其方法而不造成任何问题：

```cs
IReadOnlyList<DerivedClassA> derivedTypeList = 
  new List<DerivedClassA> { new DerivedClassA() };
var baseClassList = (IReadOnlyList<BaseClass>)derivedTypeList;
BaseClass item = baseClassList[0];
Console.WriteLine(item);
```

但是 `Array` 作为 4.0 之前就出现并广泛应用的数据结构。由于历史原因，`Array` 在设计时支持了协变性。因而如果 `B` 是 `A` 的子类，则 `B[]` 可以转换为 `A[]`。但是协变只对返回值是安全的。对于输入来说就难讲了。因此在使用数组的时候一定要特别小心。

## 本期问题

哎哎？好像还没有解答完呢？那如何得到基类型数组才能够让测试通过呢？好问题！那么就请各位实现这个方法来通过测试吧：

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

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>