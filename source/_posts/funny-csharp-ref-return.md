---
title: 【我好像遇到了假的 C#】2018年05月31日 -- ref return
date: 2018-05-31 00:04:38
tags: 
- csharp
- ref return
---

明天就是六一儿童节了。在这个大喜的日子里。每日 CLR 段子开张了。其中【我好像遇到了假的 C#】是其中的一个栏目。你可以在其中找到有趣的程序片段。当期的答案会在下期公布哦。当然没有什么奖品，大家自己在学习中获得满足感吧。

每一期的【我好像遇到了假的 C#】的代码大部分是以测试的形式给出的。很可惜，给出的测试一定是失败的。大家可以凭着自己丰富的工作经验判断一下如何才能改正这个测试呢？今天的测试是关于 `ref` 的（要实际运行测试的话，记得使用 C# 7.x 哦）：

<!--more-->

给定如下方法：

```cs
static ref int GetElementAt(int[] array, int index)
{
  return ref array[index];;
}
```

怎么才能修正下面的测试呢？

```cs
[Fact]
public void please_fix_this_sad_test()
{
  int[] numbers = {1, 2, 3, 4, 5, 6};

  ref int thirdElementRef = ref GetElementAt(numbers, 2);
  thirdElementRef = 100;

  // 注意哦，你只能修改以下两行代码。
  const int firstExpectation = default;
  int[] secondExpectation = { };

  Assert.Equal(firstExpectation, numbers[2]);
  Assert.Equal(secondExpectation, numbers);
}
```

祝大家儿童节快乐！如果您觉得本文对您有帮助，也欢迎分享给其他的人。我们一起进步。欢迎关注我的微信公众号：

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>