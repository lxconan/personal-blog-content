---
title: 【我好像遇到了假的 C#】2018年06月18日--conditional
date: 2018-06-17 09:31:41
tags:
- csharp
- diagnostics
- conditional
- attribute
---

Hello 大家好，端午假期明天就要可喜可贺的结束了（对于壕除外，因为假期仅仅过了一半还不到:-D），于是我们来更新一把，继续学习吧。今天带来了一个关于诊断的问题哦。

<!--more-->

## 上期问题解答

我们还是先来看看上期的问题把，上一期问题的描述很简单：找出一个字符串的长度（字符数目）。这道题让我又有了人生赢家的感觉。

<img src="{{root_url}}/images/blog/funny_csharp_conditional.jpg" style="text-align:center" alt="winner"/>

我们要实现一个函数：

```cs
static int GetCharacterLength(string text)
{
  throw new NotImplementedException();
}
```

来统计字符串中字符的数目。难道，不就是 `text.Length` 么？但是又有一种写下答案就会挂的既视感。嗯嗯，这还得从 .NET Framework 对字符的表示说起。.NET Framework 中的字符内部是以 Unicode（UTF-16）在内存中表示的。请问，一个 UTF-16 字符是多少位呢？

> 16 位，2 字节！

嗯~~你也是这么觉得么？实际上这是不准确的，一个 UTF-16 字符最少会有 16 位两个字节。按照 Wiki 上的解释：

> The encoding (UTF-16) is **variable-length**, as code points are encoded with one or two 16-bit code units

即一个 UTF-16 字符有可能是 16 位，也有可能是 32 位。实际上，Unicode 代表的字符数目超过了 16 位能够容纳的编码。其 *完全体* 为 UTF-32。但是在实际使用中，大多数字符都处于 [BMP](https://en.wikipedia.org/wiki/Plane_(Unicode)#Basic_Multilingual_Plane)（Basic Multilingual Plane）上，即从 `0x0000-0xffff` 编码范围内。而对应大部分的英语国家来说，大部分字符并未超出之前 ASCII 的范围，因此出现了 UTF-8、UTF-16 这种 “压缩” 编码形式。因此，一个 Unicode 字符，在 UTF-8 下可能由 1 到 4 个字节表示；而在 UTF-16 下可能由 2 个或者 4 个字节表示。

而 `string.Length` 属性返回的是什么呢？仅仅是 `char` 的数目。如果字符串中间出现了需要由 4 个字节表示的字符，那么 `Length` 的值就不准确了。因此，对于以下测试用例

```cs
static IEnumerable<object[]> TestCases() =>
  new[]
  {
    ...
    new object[]{char.ConvertFromUtf32(0x2A601) + "1234", 5}
  };
```

测试就会失败。为了能够正确确定字符的数目，应当使用 `char.IsHighSurrogate` 判断第一个遇到的字符是否是一个 Surrogate Pair 的开始部分。

```cs
static int GetCharacterLength(string text)
{
  if (text == null) throw new ArgumentNullException(nameof(text));
  int length = 0;
  for (int index = 0; index < text.Length; ++index)
  {
    if (char.IsHighSurrogate(text[index]))
    {
      ++index;
    }
    ++length;
  }

  return length;
}
```

> 这里可以补充一个小问题：请问在一个 Big-endian 的 Architecture 下，High Surrogate 还会是第一个 16 位数据么？

## 本期问题

本期问题很有意思。我们在 Debug 构建的情况下，往往会添加一些输出代码来帮助我们诊断问题。但是我们并不希望在 Release 构建中执行这些代码。下面有三种诊断代码书写模式，请问，这三种方式有什么不一样呢？

方式一：

```cs
#if DEBUG

WriteLog(GetComplicatedDiagnosticData());

#endif // #if DEBUG
```

方式二：

```cs
void WriteLog(string message) 
{
#if DEBUG
  // Writting log to somewhere
#endif
}

...

WriteLog(GetComplicatedDiagnosticData());
```

方法三：

```cs
[Conditional("DEBUG")]
void WriteLog(string message) 
{
  // Writting log to somewhere
}

WriteLog(GetComplicatedDiagnosticData());
```

<img src="{{root_url}}/images/blog/funny_csharp_barcode.jpeg" style="text-align:center" alt="wechat-app-barcode"/>