---
title: 'where, group by, having, order by 顺序'
date: 2016-04-03 21:17:44
categories: 数据库
tags: MySQL
---

## 过滤分组  

对分组过于采用HAVING子句。HAVING子句支持所有WHERE的操作。HAVING与WHERE的区别在于WHERE是过滤行的，而HAVING是用来过滤分组。  

另一种理解WHERE与HAVING的区别的方法是，WHERE在分组之前过滤，而HAVING在分组之后以每组为单位过滤。   

<!-- more -->

## 分组与排序  

一般在使用GROUP BY子句时，也应该使用ORDER BY子句。这是保证数据正确排序的唯一方法。  

SQL SELECT语句的执行顺序：  

1. from子句组装来自不同数据源的数据；   

2. where子句基于指定的条件对记录行进行筛选；   

3. group by子句将数据划分为多个分组；  

4. 使用聚集函数进行计算；   

5. 使用having子句筛选分组；  

6. 计算所有的表达式；  

7. 使用order by对结果集进行排序；   

8. select 集合输出。   

## 例子  

```sql
select 考生姓名, max(总成绩) as max总成绩

from tb_Grade

where 考生姓名 is not null

group by 考生姓名

having max(总成绩) > 600

order by max总成绩
```

在上面的示例中 SQL 语句的执行顺序如下：   

1. 首先执行 FROM 子句, 从 tb_Grade 表组装数据源的数据    

2. 执行 WHERE 子句, 筛选 tb_Grade 表中所有数据不为 NULL 的数据  

3. 执行 GROUP BY 子句, 把 tb_Grade 表按 "学生姓名" 列进行分组    
 
4. 计算 max() 聚集函数, 按 "总成绩" 求出总成绩中最大的一些数值   

5. 执行 HAVING 子句, 筛选课程的总成绩大于 600 分的   

6. 执行 ORDER BY 子句, 把最后的结果按 "Max 成绩" 进行排序   



