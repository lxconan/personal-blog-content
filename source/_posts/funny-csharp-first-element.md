---
title: 【我好像遇到了假的 C#】2018年06月02日--first element in array
date: 2018-06-02 00:35:06
tags:
- csharp
- array
---

Hello 大家好，儿童节过得怎么样？今天我们会解答一下昨天的问题，然后带来了一道还算正常一点，但是仍然很有趣的题目哦。

<!--more-->

## 上期问题解答

上期我们介绍了一个你绝对不要在 PROD Code 中书写的方法：

```cs
static void TryModifyReadonlyRef(in int value)
{
  unsafe
  {
    fixed (int* ptr = &value)
    {
      *ptr = 2;
    }
  }
}
```

这个方法可以将语义上为 `readonly ref` 的参数的值改变：

```cs
int value = 5;
TryModifyReadonlyRef(in value);
// 现在 value 是 2 了。
```

我们并不是追逐奇技淫巧，只是想向大家展示 *unsafe* 意味着什么。用人话说 *unsafe* 意味着请不要管我，我知道我要干什么。因此上述代码编译器甚至根本不会提出任何警告来。

当然只有上述解释是不够的。我们还是要看一看相关的 spec。目前 C# 语言的 Spec 正式版本还是停留在 5 的。C#6-7 的 spec 目前仅停留在 draft 阶段。以讨论的形式存档在

https://github.com/dotnet/csharplang

其中关于 `unsafe` 的说明位于：

https://github.com/dotnet/csharplang/blob/master/spec/unsafe-code.md

在 overview 的部分，提到了在 *unsafe code* 中，可以声明并操作指针，在指针和整数之间转换，利用变量的地址。而由于 *unsafe code* 必须标记为 `unsafe` 这也就意味着开发人员不会 *不小心* 书写了这种代码。

在指针类型部分可以看到，C# 只有一种指针类型 `T*`（这和 C/C++ 是不同的，例如 `T *`, `T const *`, `T * const`, `T const * const`），并可以执行复引用、访问结构体成员、索引、`++`/`--` 等 9 种操作。因此，对于 unsafe context 来说，它并不区分变量的指针，还是一个 `in` 参数的指针。而它就真的操作了~

总之 *unsafe code* 威力巨大，并且编译器一般不会对你进行温馨提示。 书写 *unsafe* 代码一定三思而行。（如果你考虑的是性能，那么可以考虑新的 `Span` 会不会帮到你）。

## 本期问题

```cs
public static object First(this Array array)
{
  #region Please implement the method

  /*
   * 请书写一个方法，获得数组的第一个元素。
   * - 不许用 LINQ 中的任何方法。例如 First()
   * - 不许用 IEnumerable 中的任何方法。
   * - 不许用数组提供的 FindXXX，或者 IndexOfXXX 这样的方法。
   * - 过程中不得拷贝数组。
   */

  throw new NotImplementedException();

  #endregion
}
```

往往看上去最简单的问题也会有深坑巨沟。

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>