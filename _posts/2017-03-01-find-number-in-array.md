---
layout: post
title:  "Algo: Find Integer"
categories: algo
tags:  c++ array binary-search
---

* content
{:toc}


```c++
#include <vector>
static bool find_number_from_two_decimen_array (int target,std::vector<std::vector<int>>& array)
{
  /**
   * 在一个二维数组中，每一行都按照从左到右递增的顺序排序，
   * 每一列都按照从上到下递增的顺序排序。
   * 请完成一个函数，输入这样的一个二维数组和一个整数，
   * 判断数组中是否含有该整数
   * ----------------------
   * Given a two dimension array with elements
   * in each row increasing from left to right and
   * in each column increasing from top to bottom
   * Please complete a function with parameters of
   * a two-dimension-array and an integer and justify
   * if the integer is contained in the array.
   * eg. input ({ 0, 1, 3, 4, 5},{ 1, 4, 6, 8, 11},23)
   * should output 23 is not in the given array
   *
   * train of thought:
   * for next row in all rows:
   *    for val in this row:
   *       if val == target:
   *          return found
   *       if target > val:
   *          continue to next val
   *       if target < val:
   *          go back to the left column besides current one
   *          break current loop to jump down to next row
   * outter loop done, must not found
   */
  if (array.empty ())
    return false;

  int row = 0, col = 0, rows = array.size ();
  for (; row < rows; row++)
  {
    if (array[row].empty ())
      continue;

    int cols = array[row].size ();
    int max = array[row][cols - 1];
    int min = array[row][0];

    if (target == min || target == max)
      return true;
    if (target < min)
      return false;
    if (target > max)
      continue;
    if (array[row][col] == target)
      return true;

    if (array[row][col] > target)
    {
      col = 0;
      cols--;
    }
    else
      col++;

    for (; col < cols; col++)
    {
      int v = array[row][col];
      if (v == target)
        return true;
      if (target > v)
        continue;
      if (target < v)
      {
        col--;
        break;
      }
    }
  }
  return false;
}

TEST_F(mpath, test_alg0)
{
  std::vector<std::vector<int>> arrary =
        {
          { 0, 1, 3, 4, 5, 7, 8, 11, 13, 15, 18, 21, 24, 27, 30 },
          { 1, 4, 6, 8, 11, 12, 15, 17, 18, 20, 23, 24, 27, 30 },
          { 4, 7, 8, 11, 14, 16, 18, 20, 21, 24, 27, 29, 32, 35, 89, 91, 93, 96 },
          { 5, 8, 10, 13, 15, 19, 21, 23, 24, 27, 29, 31 },
          { 6, 11, 14, 16, 18, 22, 24, 27, 29, 32, 33, 35, 94, 97, 99, 101,102 },
          { 9, 13, 16, 19, 21, 23, 25, 29, 31, 35, 38, 39, 42, 45, 48, 51,54, 56 } };
  bool find = find_number_from_two_decimen_array (22, arrary);
  ASSERT_TRUE(find);
}
```
