---
layout: post
title:  "C++ Concurrency in Action 1"
categories: read
tags:  thread c++
---

* content
{:toc}

### 1.2 Why use concurrency ?
There are two main reasons to use concurrency in an application: 
- Separation of concerns
  - Separation of concerns is almost always a good idea when writing software; by grouping __related__ bits of code together and keeping __unrelated__ bits of code apart, you can make your programs easier to understand and test, and thus less likely to contain bugs. 
  - Without the explicit use of concurrency you either have to write a __task-switching__ framework or __actively__ make calls to __unrelated__ areas of code during an operation. 
  - Using threads in this way generally makes the logic in each thread much simpler, because the __interactions__ between them can be limited to __clearly identi-fiable points__, rather than having to __intersperse__ the logic of the different tasks
  - In this case, the number of threads is __independent__ of the number of CPU cores
available, because the division into threads is based on the __conceptual design__ rather than an attempt to increase throughput.
- Performance
  - task parallelism 
divide a single task into parts and run each in parallel, thus reducing the
total runtime.
  - data parallelism
each thread performs the same operation on different parts of the data.
