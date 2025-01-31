---
title: Easy As GDB PicoCTF Challenge
date: 2025-01-30 00:00:00 +0000
categories: [Reverse Engineering, PicoCTF]
tags: [picoctf, reverse-engineering, crackmes]
---

# Static Analysis

![screenshot_30012025_172713](https://github.com/user-attachments/assets/0cd6c227-e88c-40e8-9f2a-58b0fb2435d6)

### You Know Whats Next, Binja Time!

![screenshot_30012025_190415](https://github.com/user-attachments/assets/6a2c859f-ad53-474b-86b8-9ac01ec2a182)

### `sub_565558c4` :

![screenshot_30012025_190835](https://github.com/user-attachments/assets/fda3ecb6-a6af-497a-aa99-60d2453fedf1)

* This checks our flag hence the reason it is in an `if` statement comparing it to 1 `true` in the `main` function.

### Comparing the lowest 8 bits of `rax` to the lowest 8 bits of `rdx`

![screenshot_30012025_194915](https://github.com/user-attachments/assets/d8f11837-2e45-4e0c-9590-b88b0695bd47)

### Python Script For Bruteforcing The Flag In GDB

```python
import gdb
import string
from queue import Queue, Empty


MAX_FLAG_LEN = 0x200

class Checkpoint(gdb.Breakpoint):
    def __init__(self, queue, target_hitcount, *args):
        super().__init__(*args)
        self.silent = True
        self.queue = queue
        self.target_hitcount = target_hitcount
        self.hit = 0

    def stop(self):
        res = []
        self.hit += 1
        #print(f"\nhit {self.hit}/{self.target_hitcount}")
        if self.hit == self.target_hitcount:
            al = gdb.parse_and_eval("$al")
            dl = gdb.parse_and_eval("$dl")
            self.queue.put(al == dl)
        return False

class Solvepoint(gdb.Breakpoint):
    def __init__(self, *args):
        super().__init__(*args)
        self.silent = True
        self.hit = 0

    def stop(self):
        #gdb.execute("q")
        self.hit += 1
        return False


gdb.execute("set disable-randomization on")
gdb.execute("delete")
sp = Solvepoint("*0x56555a71")
queue = Queue()


flag = ""
ALPHABET = string.ascii_letters + string.digits + "{}_"

for i in range(len(flag), MAX_FLAG_LEN):
    for c in ALPHABET:
        bp = Checkpoint(queue, len(flag) + 1, '*0x5655598e')
        gdb.execute("run <<< {}{}".format(flag, c))
        try:
            result = queue.get(timeout = 1)
            bp.delete()
            if result:
                flag += c
                print("\n\n{}\n\n".format(flag))
                break
        except Empty:
            print("Error: Empty queue!")
            gdb.execute("q")

    if sp.hit > 0:
        print("Found flag: {}".format(flag))
        gdb.execute("q")
```

### This is where the Solvepoint is being set

![screenshot_30012025_193808](https://github.com/user-attachments/assets/c10fe09e-328d-4b45-b404-b21fd43961b8)

### I dont have binja paid version 😢

![screenshot_30012025_194147](https://github.com/user-attachments/assets/6fa57324-d59f-4b9c-b978-d3c92356a8ac)


### The script is done!

![screenshot_30012025_194254](https://github.com/user-attachments/assets/51182eef-153f-43ed-af16-69e03805dd52)
