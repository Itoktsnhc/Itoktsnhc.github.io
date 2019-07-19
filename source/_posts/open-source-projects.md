---
title: 一些开源项目收集
date: 2019-07-12 14:17:05
tags: "reference"
---
一些开源项目的收集
<!-- more -->
# [Microsoft/Trill](https://github.com/Microsoft/Trill)  
<br>

> Trill is a high-performance one-pass in-memory streaming analytics engine from Microsoft Research. It can handle both real-time and offline data, and is based on a temporal data and query model. > Trill can be used as a streaming engine, a lightweight in-memory relational engine, and as a progressive query processor (for early query results on partial data).

# [MediatR](https://github.com/jbogard/MediatR)
<br>

> Simple mediator implementation in .NET
> In-process messaging with no dependencies.
> Supports request/response, commands, queries, notifications and  events, synchronous and async with intelligent dispatching via C# generic variance.

# [Orleans](https://github.com/dotnet/orleans)
<br>

> Orleans is a framework that provides a straight-forward approach to building distributed high-scale computing applications, without the need to learn and apply complex concurrency or other scaling patterns.

# [System.Linq.Dynamic.Core](https://github.com/StefH/System.Linq.Dynamic.Core)
<br>

> This is a .NET Core / Standard port of the Microsoft assembly for the .Net 4.0 Dynamic language functionality.
``` CSharp
var query = db.Customers
    .Where("City == @0 and Orders.Count >= @1", "London", 10)
    .OrderBy("CompanyName")
    .Select("new(CompanyName as Name, Phone)");
```

# [Recognizers-Text](https://github.com/microsoft/Recognizers-Text)
<br>

> Microsoft.Recognizers.Text provides robust recognition and resolution of entities like numbers, units, and date/time; expressed in multiple languages.
> Full support for Chinese, English, French, Spanish, Portuguese, and German. Partial support for Dutch, Japanese, and Korean. More on the way.

# [Humanizer](https://github.com/Humanizr/Humanizer)
<br>

> Humanizer meets all your .NET needs for manipulating and displaying strings, enums, dates, times, timespans, numbers and quantities

# [Scrutor](https://github.com/khellang/Scrutor)
<br>

> Assembly scanning and decoration extensions for Microsoft.Extensions.DependencyInjection

[相关blog](https://www.cnblogs.com/catcher1994/p/10316928.html)

# And so on