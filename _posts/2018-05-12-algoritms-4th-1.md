---
layout: post
title:  "SWIG-1: Introductions"
date:   2016-03-16 19:12:03
categories: tool
tags:  swig script-language c/c++
---

* content
{:toc}

Important rules of thumb  
  - The recursion has a base case—we always include a conditional statement as the first statement in the program that has a  return .
  - Recursive calls must address subproblems that are smaller in some sense, 
 so that recursive calls converge to the base case. In the code below,
 the difference between the values of the fourth and the third arguments always decreases.
  - Recursive calls should not address subproblems that overlap. In the code below,
the portions of the array referenced by the two subproblems are disjoint.
