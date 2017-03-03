---
layout: post
title:  "Algo: Is right Pop Sequence ?"
categories: algo
tags:  c++ stack
---

* content
{:toc}


```c++
#include <stack>
#include <vector>
static bool is_pop_order (std::vector<int>& pushV, std::vector<int>& popV)
{
  /*
   * 输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。
   * 假设压入栈的所有数字均不相等。例如序列1,2,3,4,5是某栈的压入顺序，
   * 序列4，5,3,2,1是该压栈序列对应的一个弹出序列，但4,3,5,1,2就不可能是该压栈序列的弹出序列。
   * （注意：这两个序列的长度是相等的）
   * ------------------------------
   * Given two integer sequences, one is push sequence of stack.
   * Please justify if the other one is a possible popup sequence of stack.
   * Assume all pushed numbers are different. eg. 1,2,3,4 is stack's push sequence,
   * 4,5,3,2,1 is a possible popup sequence while 4,3,5,1,2 is not a possible popup sequence
   *
   * Given pushV 1234, popV  4321
   * train of thought:
   * Use another stack to simulate pushs and pops
   * for ele0 in popV:
   *    for ele1 in pushV:
   *       if ele1 is not ele0:
   *          push to stack tmp  # tmp stack can be 123 when visiting all eles in pushV()
   *       if ele1 is last ele in popV:
   *          compare each ele in tmep stak with ele0:
   *            if not equal:
   *               then return false
   *            else:
   *               pop tmp stack
   * return true if temp stack empty else false         
   */
  if (pushV.empty ())
    return false;
  std::stack<int> tmp;
  int pushsize = pushV.size ();
  int popsize = popV.size ();
  int j = 0;
  for (int i = 0; i < popsize; i++)
  {
    if (j < pushsize)
    {
      while (popV[i] != pushV[j])
      {
        tmp.push (pushV[j]);
        j++;
      }
      j++;
    }
    else
    {
      if (popV[i] != tmp.top ())
        return false;
      tmp.pop ();
    }
  }
  return tmp.empty ();
}

TEST_F(mpath,test_is_pop_order)
{
  std::vector<int> pushV =
    { 1, 2, 3, 4, 5 };
  std::vector<int> popV =
    { 4, 5, 3, 2, 1 };
  bool ret = is_pop_order (pushV, popV);
  ASSERT_TRUE(ret);

  popV.clear ();
  popV =
  { 4,3,5,2,1};
  ret = is_pop_order (pushV, popV);
  ASSERT_FALSE(ret);
}
```
