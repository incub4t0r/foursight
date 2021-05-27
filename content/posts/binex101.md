---
title: "Binex 101"
date: 2020-05-03T17:05:44-04:00
draft: false
showToc: true
description: "My solution to the Binary Exploitation 101 challenge on ACICTF"
tags: ["acictf", "c"]
---

Only a 50 point problem but I really enjoyed the concepts of C programming vulnerabilities and the math behind it.

## TL;DR

Source code outlined vulnerability of integer overflow, calculated numbers to put into program to return -1 and got flag.

## Breaking it down

As with any challenge, I started with reading the source code provided, which turned out to be a good idea and a good hint provided. 

The source code given outlines the integer vulnerability of C programming, where once a stored integer passes a certain max value (2147483647), then the number will start to equal negative numbers. The only way I understood it was that our calculated numbers will start too look like 1->MAX, then MAX overflows into -MAX, then -MAX->-1, then -1 overflows again to 0 and positive 1.

Using this logic, we can do some quick maffs to calculate two numbers two overflow the integers near -1, and is outlined below

```
Multiply the max integer (2147483647) by 2 and then sqrt

Floor the the result
```

We can netcat back to the hosted problem, and input two of the same floored calculations.


Flag should be present


---
## Solve
```
Exploiting bugs in programs can definitely be difficult. Not only do you need a certain amount of reverse engineering required to identify vulnerabilities, but you also need to weaponize that vulnerability somehow. To get new hackers started, we included our annotated source code along with the compiled program.
If you don't know where to start, download the source code and open it in a program with syntax highlighting such as notepad++ or gedit. If you don't have the ability to use either of those, you can always use vim.

You can connect to the problem at telnet challenge.acictf.com 19919 or nc challenge.acictf.com 19919

```
## Hints
```
Signed integers on modern computers generally use something called "Two's Complement" for representing them. If this is your first time dealing with integers at this level, it is probably worth taking some time to get a basic understanding of them. In particular, you will need to understand what the largest positive number looks like, what -1 looks like, and how overflow is generally "handled".

We've also included debug symbols in the binary and disabled compiler optimizations. Once you understand how the C code works from the source code, it is probably worth opening the compiled binary in something like Ghidra to see both what the assembly looks like and how the recovered C code compares to the source code. Most of the other binary exploitation problems do not give you access to the raw source code.

While many binary exploitation situations involve "non-standard" inputs (such as feeding shellcode as input to the name of something), this challenge does not. Once you understand the vulnerability, you can trigger it through normal interaction with the challenge. If you are having trouble on the math side, treating the binary representation of your 'target' number as an unsigned integer may be helpful.

If you are new to binary exploitation (or C code), we really recommend reading the source file in its entirety as the comments try to explain many of the key concepts for this category of problems. For this specific problem, anyone not familiar C should definitely read the source file because the behavior of s.numbers[-1] is very different between C and some other popular languages (e.g. Python).
```
---

## Work + Notes
binex101Solution


multiply the max integer by 2 and then sqrt
floor the integer and then when prompted, present as both integers. negative number will flow to -1


goes up to max int, turns negative max int, goes to smallest negative int, then flows back over to smallest positive int