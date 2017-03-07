---
layout: post
title:  "Replace Empty Space"
categories: algo
tags:  stack
---

* content
{:toc}


```c++
static void replace_empty_space_in_string (char *str, int length)
{
  /**
   * Given a string and please repalce all empty char of " " with "%20"
   * eg. input "hello world", should output "hello%20world"
   *
   * train of thought:
   * for char in str from right to left:
   *    if char is ' ':
   *       replace char with %20
   *       decrment index by 2 as we copy two more chars
   *    else:
   *       copy char
   */
  if (str == NULL || length < 0)
    return;
  int newlen = 0;
  int str_len = strlen (str) + 1;
  for (int i = 0; i < str_len; i++)
  {
    if (str[i] == ' ')
      newlen += 2;
  }
  newlen += str_len;
  if (newlen > length)
    return;
  for (; str_len >= 0 && newlen >= 0; str_len--, newlen--)
  {
    if (str[str_len] == ' ')
    {
      str[newlen] = '0';
      newlen--;
      str[newlen] = '2';
      newlen--;
      str[newlen] = '%';
    }
    else
      str[newlen] = str[str_len];
  }
}
TEST_F(mpath,test_replace_empty_space)
{
  char str[1024];
  const char* msg = "hello world";
  strcpy (str, msg);
  replace_empty_space_in_string (str, 1024);
  ASSERT_STREQ(str,"hello%20world");
}
```
