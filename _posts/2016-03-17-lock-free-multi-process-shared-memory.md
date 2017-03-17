---
layout: post
title:  "lock-free-multi-process-shared-memory"
date:   2016-03-17 19:12:03
categories: ipc
tags:  os lock-free
---

* content
{:toc}

### How to make lock free ipc communication between 
a single writer and a single reader via shared-memory ?

#### OPTION 1:  
two fifo q1 and q2 used to simlate biodirectional data flows bwtween p1 and p2. 
 
no lock required at all. queue refers to linux lock free queue.  

q1: t1 as writer and t2 as reader so single data flow p1--->p2  
q2: t1 as reader and t2 as writer so single data flow p1<---p2

when writer writes all buffer, return error code of ENOBUF to 
caller just as socket does

#### OPTION2:  
这种单独单写的连原子操作都不需要，更不要说mutex之类的了。

假设A,B两进程通信

首先，将共享内存区域分成n块，如何分自己定义。

其次，每块内存指定一个byte做标志，

定义为: 0为A可读写状态，B不可读写；  
       1为B可读写状态，A不可读写。  
A的行为：
          遍历n块内存，找到一个标志为0的：
          接收：
          查看是否有B发来的消息，有，则接收，设置为可发送状态；
          发送：消息复制到该块区域，设置标志为1；
B的行为:
          遍历n块内存，找到一个标志为1的：
          接收：
          查看是否有B发来的消息，有，则接收，设置为可发送状态；
          发送：消息复制到该块区域，设置标志为0；


