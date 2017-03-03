---
layout: post
title:  "Network: Endian Formate"
categories: network
tags:  endian memory 
---

* content
{:toc}

#### Binary expression (be) and Value expression (ve) are different things.

Basically, be is the store format in memory while ve is the human-readable values that be represents.

eg. short a = 128

its value expression is looking like this: 00000000 10000000

its binary expression is looking like this:

little endian 10000000 00000000

big endian 00000000 10000000

the aim of this blog is to highlight a misunderstanding I used to have. That is I mixed up the be and ve. Take the example above, I used to thought little endian 10000000 00000000 should be calculated as value of 2^10 because the bit 1 is at the index of 10.

Actually, the correct way to calculate the value by cpu is like this: because the first byte 100000 is stored in low address so it should be high byte in value expression, and the second byte 000000 should be the low byte in value expression, take attention on the bold fonts, this is highlighted because it was where I made mistakes. So, the final value expression should be 00000000 10000000, and it value is therefore 2^8 that equals to the old value.
