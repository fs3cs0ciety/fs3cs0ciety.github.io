---
title: Bbbbloat PicoCTF Challenge
date: 2025-01-30 00:00:00 +0000
categories: [Reverse Engineering, PicoCTF]
tags: [picoctf, reverse-engineering, crackmes]
description: Reverse engineering a stripped linux executable
---

## Always Do One Of These Before Anything
![screenshot_30012025_154832](https://github.com/user-attachments/assets/827c6d04-1c28-4b28-9231-84bc83794e47)

### Stripped binaries are always fun to read ...
![screenshot_30012025_155406](https://github.com/user-attachments/assets/8c62ad03-a697-4637-b708-6e318e6d9db2)

* Immediatly looking into `main()`, we see a comparison happening

```c
000014d0        if (var_48 != 0x86187)
0000158a            puts(str: "Sorry, that's not it!")
```

* Nice, so if `var_48` is not qual to `0x86187` call `puts()`. Instresting ...
* `0x86187` is `549255` in decimal, its the programs favorite number!!!

### Just incase you didnt know we can do this 
![screenshot_30012025_160427](https://github.com/user-attachments/assets/405d93b0-a1b7-437b-ac5f-c22b490ff63d)

### Lets Attempt Getting The Flag!
![screenshot_30012025_161233](https://github.com/user-attachments/assets/00c900c2-b673-4a4d-83f2-647ac81effc4)


*Coolest looking cat ever*

![image](https://github.com/user-attachments/assets/f2168ea8-c655-4179-830b-0873b9c74bb3)
