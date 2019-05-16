---
layout: post
title: "记一次Flink Checkpoint超时问题排查-扩展:InputQueueLength最大值计算方式"
date: 2019-04-25
updated: 2019-04-25
description: 本文主要关于Flink Checkpoint超时问题排查以及相关参数InputQueueLength最大值计算方式分析，基于Flink1.7.1版本。
categories:
- BigData
tags:
- Flink
---
> 本文主要关于Flink Checkpoint超时问题排查过程，基于Flink1.7.1版本。  
  
最近在使用Flink进行流处理开发，开发时间不长，但是经历了很长时间的调优过程。最近发现Checkpoint经常出现超时失败的情况，所以对这一现象进行了排查。  
  
针对Flink Checkpoint超时问题，已经有一个大神最近刚刚写了关于这个的排查思路给了很多的参考，我也跟着他的思路，来进行进一步排查。参考文章如下:[Flink常见Checkpoint超时问题排查思路](https://www.jianshu.com/p/dff71581b63b)，这个文章最开始对于checkpoint超时做了分析，后来给出了一般的排查方式，我也是对照他的排查方式来查找问题的。下面文章我先针对此问题排查再跟一次，然后再对关键metrics的监控阈值进行下分析。  
  
# Flink Checkpoint超时问题排查  
最近程序经常出现checkpoint超时失败的问题，因此去排查了一下问题，查找资料的时候看到了上文提到的文章，[Flink常见Checkpoint超时问题排查思路](https://www.jianshu.com/p/dff71581b63b)，在他最初的分析中指出“可能是因为Barrier对齐原因导致的checkpoint超时”，我就对这这个思路去排查了下问题。  
首先，可以看到下图checkpoint开始失败了  
  
  
  
  
  
  
  
  
  
  
  
  

   
  
---    
至此，本篇内容完成。  
