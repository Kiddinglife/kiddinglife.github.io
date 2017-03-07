---
layout: post
title:  "Frog Jump Floor"
categories: algo
tags:  iteration
---

* content
{:toc}

```c++
static int jump_floor (int number)
{
  /**
   * 一只青蛙一次可以跳上1级台阶，也可以跳上2级。求该青蛙跳上一个n级的台阶总共有多少种跳法.
   * A frog can jump 1 or 2 levels at a time and
   * please calculate how mnay ways to jump to the top of an 'n' level stair
   * Train of Thoughts:
   * when level 1, 1 way to jump
   * when level 2, 2 ways to jump
   * when level 3, 3 ways to jump
   * when level 4, 5 ways to jump
   * when level 5, 8 ways to jump
   * ....
   * when level n, f(n-1)+f(n-2) ways to jump
   */
  if (number <= 0)
    return 0;
  if (number == 1)
    return 1;
  if (number == 2)
    return 2;
  int first = 1, second = 2, third = 0;
  for (int i = 3; i <= number; i++)
  {
    third = first + second;
    first = second;
    second = third;
  }
  return third;
}
TEST_F(mpath,test_jump_floor)
{
  ASSERT_EQ(jump_floor(5), 8);
}
```


 


