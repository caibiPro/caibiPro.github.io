---
title: Git diff输出格式之unified format
author: mingqing
date: 2023-03-10 15:39:00 +0800
categories: [Tools, Git]
tags: [git, diff]
math: true
mermaid: true
---
## diff Output Formats

### Context Format(`-c`)

> The context output format shows several lines of context around the lines that differ. It is the standard format for distributing updates to source code.

即`-c`选项时，这种对比文件时，感觉不是很直观；主要的场景是，用diff来生成代码补丁，代码差异行上下有上下文，**方便补丁程序patch来进行差异代码定位**。

### Unified Format(`-U`)

> The unified output format is a variation on the context format that is more compact because it omits redundant context lines.

即`-u`选项时，这种对比文件时，感觉还比较方便看；官方定义如上，即该格式是context format的变体，因为省略了冗余的上下文行，显得更加紧凑

## Unified Format Introduction

![Image.png](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/A46E718B-4699-4F40-B7B4-F38DC14226E7/D03EB46E-6238-4C5B-A158-A6E056A2F94F_2/E1Nw0i8y7fmtgIRI70CII3eKALUWto1tZlG8KVqVpiUz/Image.png)

### 文件列表

```shell
--- a/test.txt
+++ b/test2.txt
```

### 差异段列表

@@ 这一行，表示一个汇总信息，其中的-4，其实应该是-4,1。 "-"代表文件`--- a/test.txt`，4代表第四行，*test.txt*的第4行，就是`4444444444444444444`这一行，被省略的1，表示展示的*text.txt*从第4行开始的行的数量。

比如，4,2表示展示第4、5行，4,3表示展示第4、5、6行。

\+4,2同理，"+"代表文件`+++ a/test2.txt`，也就是展示4开头的两行，即第4、5行。

```shell
@@ -4 +4,2 @@
-4444444444444444444
+4444444448878784444444444
+insert1
```

- `‘+’`: A line was added here to the first line
- `'-'`: A line was removed here from the first file
- `space character`: The lines common to both files
