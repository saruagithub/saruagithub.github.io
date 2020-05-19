---
title: hadoopBook第四版chapter1
date: 2020-05-06 20:55:07
tags:
- 大数据
categories:
- 数据挖掘
---



### Chapter1 hadoop基础

大数据的流行，数据存储与读取问题（单一硬盘读取慢，从多个中并行读更快），但单一硬盘会丢失数据，因此需要同一份数据复制到不同机器。

同时在读取数据的时候还需要分析，因此MapReduce提供了一种编程模型，该模型从磁盘读取和写入中抽象出问题，并将其转换为对键和值集的计算。

MapReduce不适合实时查询，更适合离线使用。Hadoop延伸了很多工具，Hbase对单行数据读可以批操作；Yarn是集群资源管理系统，允许任何分布式程序在Hadoop集群中的数据上运行。交互式SQL，分布式查询引擎。交互处理，spark支持更灵活的数据探索，更适合机器学习。实时Storm流处理，实时计算。Solr搜索，对HDFS的文档建立索引。



### Cha2 weather数据

挖掘天气数据，天气传感器每小时在全球许多地方收集数据，并收集大量的日志数据，这是使用MapReduce进行分析的理想选择，因为我们要处理所有数据，并且数据是半结构化且面向记录的。例子，找出每年的最高温度。

#### 脚本

如果脚本在多个机器之间分块执行，会有协作和可靠性问题存在。Hadoop则可以避免这些问题，利用Hadoop，将处理写MapReduce作业，经过一些本地小规模测试之后，我们可以在机器集群上运行。

```shell
#!/usr/bin/env bash 
for year in all/* 
do   
    echo -ne `basename $year .gz`"\t”   
    gunzip -c $year | \     
        awk '{ temp = substr($0, 88, 5) + 0;            
        q = substr($0, 93, 1);            
        if (temp !=9999 && q ~ /[01459]/ && temp > max) max = temp }          
        END { print max }’ 
done
```



#### MapReduce

input：原始天气数据 NCDC，文本输入该格式为数据集中的每一行提供文本值。

Map：功能就是提取年份和温度，过滤缺失错误的记录。map为reduce（找最高温）做数据准备。













