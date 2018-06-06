---
title: 【我好像遇到了假的 C#】2018年06月01日--in parameter
date: 2018-06-01 00:09:08
tags: 
- csharp
- in parameter
---

Hello 大家儿童节快乐。

## 上期问题解答

上一期的问题涉及到了 C# 7.0 的新特性：`ref returns`。我把核心的代码放在这里：

```cs
static ref int GetElementAt(int[] array, int index)
{
  return ref array[index];;
}

...
int[] numbers = {1, 2, 3, 4, 5, 6};

ref int thirdElementRef = ref GetElementAt(numbers, 2);
thirdElementRef = 100;
```

<!--more-->

大家如果自己运行了程序的话可以知道，答案是 `numbers` 在程序执行之后变成了 `{1, 2, 100, 4, 5, 6}`。`ref return` 只能返回 `local ref`，关于这个部分的详细信息，建议大家阅读这篇文章：

https://blogs.msdn.microsoft.com/mazhou/2017/12/12/c-7-series-part-7-ref-returns/

这个问题还有很多变种，例如，如果我们将程序调用从：

```cs
ref int thirdElementRef = ref GetElementAt(numbers, 2);
```

变为

```cs
int thirdElementRef = GetElementAt(numbers, 2);
```

那么会有什么不一样呢？

## 本期问题

好了，今天我们来看一个更好玩儿的问题。在 C# 7.2 中引入了 `in` 参数。`in` 参数可以看做一个只读的 `ref`。因此，以下的程序是语法错误：

```cs
static void TryModifyReadonlyRef(in int value)
{
  value = 2;
}
```

好了，今天的测试来了，大家想一想，如何才能修好这个测试呢？给定如下的函数：

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

请修改指定的位置让测试通过：

```
[Fact]
public void do_you_think_if_the_in_parameter_be_changed()
{
  int value = 5;
  TryModifyReadonlyRef(in value);

  // 只能修改这一行的值哦。
  const int expect = default;
  Assert.Equal(expect, value);
}
```

再次祝大家儿童节快乐！希望天下的宝宝都能够健康成长。

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>