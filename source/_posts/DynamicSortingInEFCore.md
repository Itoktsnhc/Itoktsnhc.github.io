---
title: EF Core的一种动态排序实现
date: 2021-07-09 15:07:55
tags: EFCore
---
当我们使用EF的时候，有时候需要到处理运行时才可以确定的排序规则字符串，例如  
 `-Name,Type,-Age` 这样的排序规则串代表按照Name降序，然后按照Type升序，最后按照Age倒序。可以使用 [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core) 来实现。  
 ``` c#
class Person //entity定义
{
  public string Name { get; set; }
  public int Age { get; set; }
  public string Type { get; set; }
}
//query:IQueryable<T>; sortStr类似`-Name,Type,-Age`

//首先将IQueryable<T> 转换为 IOrderedQueryable<T>
var orderedQuery = query.OrderBy(_ => 0);

//获取类型的所有属性，映射为 (全大写)->原属性的字典
var props = typeof (Person)
  .GetProperties()
  .ToDictionary(s => s.Name.ToUpper(), s => s.Name);

//确定排序条件
var sorts = sortStr.Split(',', StringSplitOptions.RemoveEmptyEntries)
  .Select(s => 
  {
    var tmp = s.Trim().ToUpper();
    return (!tmp.StartsWith("-"), tmp.TrimStart('-'));
  });

//按照可用的属性名
foreach(var (isAsc, sort) in sorts) {
  if (props.TryGetValue(sort, out var propName)) 
  {
    orderedQuery = orderedQuery.ThenBy(propName + (isAsc ? " ASC" : " DESC"));
  }
}
query = orderedQuery;

 ```
 