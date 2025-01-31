---
title: Reverse Cipher PicoCTF 
date: 2025-01-30 00:00:00 +0000
categories: [Reverse Engineering, PicoCTF]
tags: [picoctf, crackme, reverse-engineering]
description: We have recovered a binary and a text file. Can you reverse the flag?
---

## Intresting 
![screenshot_30012025_225145](https://github.com/user-attachments/assets/12e1d9da-7d13-4998-94f8-783d0cccced2)

### Binja Time

![screenshot_30012025_231146](https://github.com/user-attachments/assets/87927ced-9045-4bfd-8e64-e3b24c36e2aa)

* Nice we see two comparisons right off the bat, `if (flag_file == 0)` and `if (rev_file == 0)`.
* We can see that the cipher is happening inside of the second `for` loop.
  
### The Cipher Logic

  ```c
  for (int32_t i_1 = 8; i_1 s<= 0x16; i_1 += 1)
  char rax_7 = *(&buf + sx.q(i_1))
  char var_9_3
            
  if ((i_1 & 1) != 0)
     var_9_3 = rax_7 - 2
  else
     var_9_3 = rax_7 + 5
            
  fputc(c: sx.d(var_9_3), fp: rev_file)
  ```

### Crafting up a little c program to reverse this sucka and print the flag!!
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <unistd.h>

void memory_map(const char *filename, unsigned char **mapped_data, size_t *size) {
    int fd = open(filename, O_RDONLY);
    if (fd == -1) {
        perror("Error opening file");
        exit(EXIT_FAILURE);
    }

    *size = lseek(fd, 0, SEEK_END); 
    *mapped_data = mmap(NULL, *size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (*mapped_data == MAP_FAILED) {
        perror("Error memory mapping file");
        close(fd);
        exit(EXIT_FAILURE);
    }

    close(fd);  
}

int main() {
    const char *filename = "rev_this";
    unsigned char *mapped_data;
    size_t size;

    memory_map(filename, &mapped_data, &size);

    // Print the first 8 characters
    for (int i = 0; i < 8; i++) {
        printf("%c", mapped_data[i]);
    }

    // Print characters from position 8 to 22 with transformation
    for (int i = 8; i < 23; i++) {
        if ((i & 1) == 0) {
            printf("%c", mapped_data[i] - 5);
        } else {
            printf("%c", mapped_data[i] + 2);
        }
    }

    // Print the 24th character (index 23)
    printf("%c\n", mapped_data[23]);

    if (munmap(mapped_data, size) == -1) {
        perror("Error unmapping memory");
        exit(EXIT_FAILURE);
    }

    return 0;
}
```

### Retreiving the flag
![screenshot_30012025_232721](https://github.com/user-attachments/assets/da733162-bd3f-4e21-8f06-376e39e8a6a3)
