---
title: 【我好像遇到了假的 C#】2018年06月03日--delegate
date: 2018-06-03 00:39:40
tags:
- csharp
- delegate
- immutable
---

Hello，大家好，一天一条真是太难熬了，真想一次发个十条八条的。先来讨论一下上次的题目吧。

## 上一期问题的解答

上一次的题目是书写一个获得数组第一个元素的函数。嗯~这道题一开始简单到让我有了人生赢家的错觉。

<!--more-->

<img src="{{root_url}}/images/blog/funny_csharp_delegate_winner.jpeg" style="text-align:center" alt="winner"/>

飞快的书写了实现：

```cs
public static object First(Array array)
{
  if (array == null) throw new ArgumentNullException(nameof(array));
  return array.GetValue(0);
}
```

完结撒花。但是~一般情况下既然题目都已经出到这个份上一定不会让你这么容易就全身而退的。你，猜中了！考虑以下的输入：

```cs
First(new int[,] {{1, 2, 3}, {4, 5, 6}});
```

瞬间打脸，华丽的 `ArgumentException`。因为数组，不一定是一维的呦。

<img src="{{root_url}}/images/blog/funny_csharp_delegate_haha.jpeg" style="text-align:center" alt="haha"/>

因此我们需要考虑多维数组的情况。代码变成了：

```cs
public static object First(Array array)
{
  if (array == null) throw new ArgumentNullException(nameof(array));
  int[] indexers = new int[array.Rank];
  return array.GetValue (indexers);
}
```

这肯定是正确答案的！因为这个代码就在 CSharp 7 In A Nutshell 这本书的官方范例里！这本书的中文版我正在紧张的翻译中，感兴趣的小伙伴敬请期待哦（看我无耻的插入广告）。然而，对于如下的调用：

```cs
Array array = Array.CreateInstance(
  typeof(int), new [] {3}, new [] {2});
array.SetValue(1, 2);
array.SetValue(2, 3);
array.SetValue(3, 4);

First(array);
```

再次报以 `ArgumentOutOfRangeException` 的惊喜！知道以上代码什么意思的同学举手。嗯，CLR 并不是为了 C# 生的，很多的语言中都支持下标并非从 0 开始的数组。因此上述代码创建了一个一维数组，这个数组的长度是 3，并且下标是从 2 开始的。我们为数组下标为 2、3、4 的元素赋值为 1、2、3。而我们的 `First` 代码中假定了数组的下标一定是从 0 开始的，于是就遇见了惊喜。

于是，我们就又开始修补这个问题。

```cs
public static object First(Array array)
{
  if (array == null) throw new ArgumentNullException(nameof(array));
  int[] indexers = new int[array.Rank];
  for (int dim = 0; dim < array.Rank; ++dim) {
    indexers[dim] = array.GetLowerBound(dim);
  }
  return array.GetValue (indexers);
}
```

完了么？其实~还没有，因为 `Array` 某一维度的长度，可以是 `0`！也就是说对着以下调用：

```cs
First(new int[0]);
```

今天的惊喜可能太多了点……。我们还是可以填坑的：

```cs
public static object First(Array array)
{
  if (array == null) throw new ArgumentNullException(nameof(array));

  // 代码怀味道显现，但是我们不重构了
  for (int dim = 0; dim < array.Rank; ++dim) {
    if (array.GetLength(dim) == 0) {
      throw new InvalidOperationException();
    }
  }

  int[] indexers = new int[array.Rank];
  for (int dim = 0; dim < array.Rank; ++dim) {
    indexers[dim] = array.GetLowerBound(dim);
  }
  return array.GetValue (indexers);
}
```

怎么样。是不是简单的问题本身有可能就是深坑巨沟呢？当然我们给大家设置了重重的限制条件的目的并不是为了告诉大家请在 PROD Code 中这样写。而是引出 C# 数组的一些容易被忽略的概念。毕竟，`Array` 实现了 `IEnumerable`，如果真的要写 PROD Code，也会采用：

```cs
public static object First(Array array) {
  if (array == null) throw new ArgumentNullException(nameof(array));

  IEnumerator enumerator = array.GetEnumerator();
  if (!enumerator.MoveNext()) {
      throw new InvalidOperationException();
  }

  return enumerator.Current;
}
```

这种简单高效又美好的方式来写。

## 本期的题目

“晓” 是一个邪（友）恶（爱）的组织。今天我们有一个类来专门帮助我们加入这个组织。它的代码是：

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

注意，（1）和（2）是两种不同的实现，每次只能选择其一。现在我们进行如下的调用：

```cs
Action<StringBuilder> basic = sb => {};
Action<StringBuilder> subscribed = sb => { sb.Append("I have joined -_<"); };
new Akatsuki(ref basic, ref subscribed);

var finalMessageBuilder = new StringBuilder();
basic.Invoke(finalMessageBuilder);
Console.WriteLine(finalMessageBuilder.ToString());
```

你猜，在（1）的时候，输出会是什么呢？在（2）的时候输出会是什么呢？为什么呢？

如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>