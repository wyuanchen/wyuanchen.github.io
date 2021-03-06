---
title: Lucene-Field.Store的Field.Index属性笔记
date: 2017-01-28 10:58:54
categories: 后台
tags:
- 全文检索
- Lucene
---
Field有两个属性可选：存储和索引。 
**通过存储属性你可以控制是否对这个Field进行存储；** 
**通过索引属性你可以控制是否对该Field进行索引。**

这两个属性的正确组合很重要。 

| Field.Index | Field.Store | 说明 |
|:-:|:-:|:-:|
| TOKENIZED| YES| 被分词索引且存储 |
|TOKENIZED |NO| 被分词索引但不存储 |
|NO |YES| 这是不能被搜索的，它只是被搜索内容的附属物。如URL等 |
|UN_TOKENIZED| YES/NO| 不被分词，它作为一个整体被搜索,搜一部分是搜不出来的 |
|NO| NO| 没有这种用法|     

<!-- more -->


    Field.Store.YES:存储字段值（未分词前的字段值） 
    Field.Store.NO:不存储,存储与索引没有关系 
    Field.Store.COMPRESS:压缩存储,用于长文本或二进制，但性能受损 
    Field.Index.ANALYZED:分词建索引 
    Field.Index.ANALYZED_NO_NORMS:分词建索引，但是Field的值不像通常那样被保存，而是只取一个byte，这样节约存储空间 
    Field.Index.NOT_ANALYZED:不分词且索引 
    Field.Index.NOT_ANALYZED_NO_NORMS:不分词建索引，Field的值去一个byte保存 
    TermVector表示文档的条目（由一个Document和Field定位）和它们在当前文档中所出现的次数 
    Field.TermVector.YES:为每个文档（Document）存储该字段的TermVector 
    Field.TermVector.NO:不存储TermVector 
    Field.TermVector.WITH_POSITIONS:存储位置 
    Field.TermVector.WITH_OFFSETS:存储偏移量 
    Field.TermVector.WITH_POSITIONS_OFFSETS:存储位置和偏移量