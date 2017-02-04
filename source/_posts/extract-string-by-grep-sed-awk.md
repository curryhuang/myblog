---
layout: post
category: shell
title: 使用grep/sed/awk提取字符串
tags: [grep, sed, awk, linux]
date: 2015/07/18
---
最近需要对日志进行分析，顺便学习了一下以前一直都没弄得太明白的awk和sed。

### 任务描述

日志是一个tsv文件，这里以sample.log为例，每一行大致长成下面这样，需要查看某个api的最长10次查询时间。根据记录格式，需要提取最后一个字段`ts=`之后的字符串，然后进行排序，排序和求top比较容易，`sort -nr | head -n 10`即可，比较麻烦的是从字符串中提取数字。

```
[2015-07-18 11:00:01,807]\t[service_name]\t[INFO]\t[172.0.0.1]\t[api/mock_api1]\t[ts=239]
```

### 解决方案

Google了一番以后，发现grep/sed/awk均可以，下面分别说一下每一种方案。

*以下方案均在CentOS6上测试通过*

#### grep

```bash
grep "api/mock_api1" sample.log |\
grep -Eo '\[ts=[0-9]+\]' |\
grep -Eo '[0-9]+'
```
grep参数说明：

`E` 使用正则表达式

`o` 只返回匹配的部分

分两部匹配，第一步提取出`[ts=xxxx]`，第二步提取出数字

#### sed

```bash
grep "api/mock_api1" sample.log |\
grep -Eo '\[ts=[0-9]+\]' |\
sed -r 's/\[ts=([0-9]+)\]/\1/g'
```
sed参数说明：

`r` 使用正则表达式

使用sed也需要通过grep先匹配出最后一个字段，再使用sed提取数字部分，`\1`表示正则表达式中的分组1

#### gawk

```bash
grep "api/mock_api1" sample.log |\
gawk -F'\t' 'match($6, /\[ts=([0-9]+)\]/, arr) { print arr[1] }'
```

gawk参数说明：

`F` 指定分隔符，这里指定为`tab`

match的函数参数分别为：待匹配字符串，模式，匹配后的字符串分组

awk比较强大，可以直接提取出字段信息，这里使用的是gawk，gawk兼容awk语法，经测试，同样的语法awk无法满足需求，应该是gawk和awk实现有细微区别。

### 总结

如果仅仅是提取一个字段信息，三个工具都可以满足需求，相比起来，awk的功能更强大，完成字符串提取，grep和sed的能力更多依赖正则表达式，如果需要同时处理一行记录中的多个字段，比如找出访问请求最长的10个ip，grep和sed就需要很复杂的正则表达式去匹配。
