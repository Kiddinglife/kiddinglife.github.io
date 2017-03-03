---
layout: post
title:  "Algo: Reverse List"
categories: algo
tags:  c++ list ordering
---

* content
{:toc}

```c++
struct ListNode
{
    int val;
    ListNode* next;
    ListNode (int x) :
        val (x), next (NULL)
    {
    }
};
static ListNode* reverse_list (ListNode* pHead)
{
  /**
   * 输入一个链表，反转链表后，输出链表的所有元素。
   * ------------------------------
   * Given a list. Please reverse it and then print all elements in it.
   *
   * train of thought:
   * order sublist and put the first unorder ele in the front of ordered sublist
   * 1 234  1 is init ordered sublist, 234 is unordered sublist
   * we put the first ele 2 in unordered sublist to the front of ordered sublist 1, we have
   * 21 34 21 is ordered sublist, 34 is unordered sublist
   * we put the first ele 3 in unordered sublist to the front of ordered sublist 21, we have
   * 321 4  321 is init ordered sublist, 4 is unordered sublist
   * we put the first ele 4 in unordered sublist to the front of ordered sublist 321, we have
   * 4321
   * done!
   */
  if (pHead == NULL)
    return NULL;
  ListNode *tail = pHead, *nxt = pHead->next;
  while (nxt != NULL)
  {
    // insert nxt to front of head
    tail->next = nxt->next;
    nxt->next = pHead;
    pHead = nxt;
    nxt = tail->next;
  }
  return pHead;
}
TEST_F(mpath,test_reverse_list)
{
  ListNode one = ListNode (1);
  ListNode two = ListNode (2);
  ListNode three = ListNode (3);
  one.next = &two;
  two.next = &three;
  ListNode* head = reverse_list (&one);
  ASSERT_EQ(3, head->val);
  ASSERT_EQ(2, head->next->val);
  ASSERT_EQ(1, head->next->next->val);
}
```
