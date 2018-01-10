---
title: pandas 常用操作
date: 2017-01-28 10:49:40
categories: 数据处理
tags: Pandas
---
### 删除操作   

- 删除列： 
`df=df.drop('column_label',axis=1)`
- 删除行： 
`df=df.drop('row_label')`
- 删除重复的行: `df=df.drop_duplicates(['column_label_one','column_label_two'])`

### 查看  

- 查看行数
`len(df) or len(df.index) or df.shape[0]`
- 列数
`len(df.columns) or df.shape[1]`
- 数据类型
`df.dtypes`   

<!-- more -->

### 重命名  

- 列标签的重命名
`df.rename(columns={"old label": "new label"})​`
- 行标签的重命名
`df.rename(index={"old label": "new label"}​`

### 时间序列的操作  

- 将时间字符串转换成datetime数据
`dt['StartTime'] = pd.to_datetime(dt['StartTime'])`

### 排序   

- 按值排序，可指定列名和排序方式，默认的是升序排序
`dt.sort(['StartTime'], inplace=True) or dt.sort(['StartTime'])` 
- 照索引（行名）或者列名进行排序,指定axis=0表示按索引（行名）排序，axis=1表示按列名排序，并可指定升序或者降序：
`df.sort_index(axis=1, ascending=False)`

### 读写操作   

- 读csv
`pd.read_csv('input.csv') | pd.read_table('input.csv', sep=',')`

    - 参数 header = None pandas分配默认列名
    - 参数 name = ['a', 'b', 'c'] 指定列名
    - 参数 index_col='idx 指定索引
    - 参数 shiprows = [0, 2, 4] 跳过文件部分行
    - 参数 nrows = 20 只读取文件前xx行
    - 参数 chunksize = 10000 指定每次读取行数，分块读取，返回TextParse对象

- 写csv
`pd.to_csv('output.csv')`

    - 参数 na_rep = 'NULL' 缺失值输出为指定标记值，默认为空字符串
    - 参数 index = False, header = False 禁止输出行和列的标签, 默认输出
    - 参数 cols=['a', 'b'] 指定输出以部分列，并以指定顺序排序



