---
layout: post
title:  "CCIA-3-1-shared-data-between-threads"
date:   2017-03-12 10:02:02
categories: read
tags:  thread c++
---

* content
{:toc}

## 3.1 Problems with sharing data between threads
If all shared data is read-only, there’s no problem. However, if data is shared between threads, and **one or more threads** start modifying the data, there’s a lot of potential for trouble.

One concept that’s widely used to help programmers reason about their code is that of invariants—statements that are always true about a particular data structure, These invariants are often broken during an update, especially if the data structure is of any complexity or the update requires modification of more than one value.

Consider a doubly linked list, the steps in deleting an entry from such a list are shown below:
 1. Identify the node to delete (N). a
 2. Update the link from the node prior to N to point to the node after N. b
 3. Update the link from the node after N to point to the node prior to N. c
 4. Delete node N. d

if one thread is reading the doubly linked list while another is removing a node, it’s quite possible for the reading thread to see the list with a node only partially removed (because only one of the links has been changed, as in step b of figure 3.1), so the **invariant is broken**. 

The consequences of this broken invariant can vary; if the other thread is just reading the list items from left to right in the diagram, it will skip the node being. On the other hand, if the second thread is trying to delete the rightmost node in the diagram, it might end up permanently corrupting the data structure and
eventually crashing the program. this is an example of one of the most common causes of bugs in concurrent code: **a race condition**.

### 3.1.1 Race Condition
In concurrency, a race condition is anything where the outcome depends on the **relative ordering** of execution of operations on two or more threads; the threads race to perform their **respective operations**.

the term race condition is usually used to mean a **problematic race condition**; **benign race conditions** aren’t so interesting and aren’t a cause of bugs. The C++ Standard also defines the term data race to mean the specific type of race condition that arises because of concurrent modification to a single object (see section 5.1.2 for details); data races cause the dreaded undefined behavior and it is problemic race condition.  

Take steps in deleting an entry from such a list above as example. Assume thread a operation is to visit each element and thread b operation is to delete element. There are 3 outcomes with diferent relative-ordering executions:
 1.  benign race condition outcome: thread a reads deleting element -> thread b deletes left link step b-> thread b deletes right link step c 
 2.  problemic race condition outcome: thread b deletes left link step b -> thread a reads deleting element -> thread b deletes right link step c 
 3.  benign race condition outcome: thread a reads deleting element -> thread b deletes right link step c ->  thread a reads deleting element  

### 3.1.2 Problematic race conditions
Problematic race conditions typically occur where completing an operation requires modification of two or more distinct pieces of data, such as the two link pointers in the above example. 

what happens is that the operation must access two separate pieces of data, these must be modified in separate instructions, and another thread could potentially access the data structure when only one of them has been completed.

Because race conditions are generally timing sensitive, they can often disappear entirely when
the application is run under the debugger, because the debugger affects the timing
of the program, even if only slightly.

### 3.1.3 Avoid Problematic race conditions
 -  The simplest option is to wrap your data structure with a protection mechanism, to ensure that only the thread actually performing a modification can see the intermediate states where the invariants are broken. From the point of view of other threads accessing that data structure,such modifications either haven’t started or have completed.

 - Another option is to modify the design of your data structure and its invariants so
that modifications are done as a series of indivisible changes, each of which preserves
the invariants. This is generally referred to as **lock-free** programming and is difficult to get
right.

 - Another way of dealing with race conditions is to handle the updates to the data
structure as a transaction, just as updates to a database are done within a transaction.
The required series of data modifications and reads is stored in a transaction log and
then committed in a single step.

The most basic mechanism for protecting shared data provided by the C++ Standard is the mutex, so we’ll look at that in my next blog. See you then!












