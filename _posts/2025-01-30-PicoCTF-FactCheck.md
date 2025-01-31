---
title: FactCheck PicoCTF Challenge
date: 2025-01-30 00:00:00 +0000
categories: [Reverse Engineering, PicoCTF]
tags: [picoctf, reverse-engineering, crackmes]
description: Reverse Engineering The easiest binary in the world, lol! Load up the ctf and follow along.
---

# By Far The Easiest Crackme Ever To Exist

![screenshot_30012025_152709](https://github.com/user-attachments/assets/64cd9916-efd4-4a4d-986a-bd77134b287f)

### Inspecting inside of binja we see :

![screenshot_30012025_152209](https://github.com/user-attachments/assets/0232a37a-f9a8-477c-90fd-8bcdcbe73200)

### What does this main function do??

* Does a shit ton of c++ string handling!
* At the last operator we see a concatenating `'}'` which in hex is `0x7d`. Nice maybe the end of the ctf flag???
* Set a bp at the first instruction after that last call to the operator.
* the flag is being stored in the RAX register as we see.

  ```shell
  flag = picoCTF{wELF_d0N3_mate_239b483f}
  ```
