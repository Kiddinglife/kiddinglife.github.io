---
layout: post
title:  "Algorithms-4th-Recursion"
date:   2018-03-16 19:12:03
categories: algo
tags:  recursion 
---

* content
{:toc}

Important rules of thumb
 1. The recursion has a base case—we always include a conditional statement as the first statement in the program that has a  return .
 2. Recursive calls must address subproblems that are smaller in some sense so that recursive calls converge to the base case.
 3. Recursive calls should not address subproblems that overlap. 
 
```java
// Recursive implementation of binary search
public static int rank(int key, int[] a, int lo, int hi)
{
  // apply thumb 1 that recursion has a base case.
  // Index of key in a[], if present, is not smaller than lo and not larger than hi.
  if (lo > hi) return -1;
  // apply thumb 2 that the difference between lo and hi is decreased
  int mid = lo + (hi - lo) / 2;
  if (key < a[mid])
    // apply thumb 3 that the portions of the array referenced by the two subproblems are disjoint.
   return rank(key, a, lo, mid - 1);
  else if (key > a[mid]) 
    // apply thumb 3 that the portions of the array referenced by the two subproblems are disjoint.
   return rank(key, a, mid + 1, hi);
  else 
   return mid;
}
```
