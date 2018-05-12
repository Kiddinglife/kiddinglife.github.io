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
 - The recursion has a base case—we always include a conditional statement as the first statement in the program that has a  return .
 - Recursive calls must address subproblems that are smaller in some sense, 
 so that recursive calls converge to the base case. In the code below,
 the difference between the values of the fourth and the third arguments always decreases.
 - Recursive calls should not address subproblems that overlap. In the code below,
the portions of the array referenced by the two subproblems are disjoint.
