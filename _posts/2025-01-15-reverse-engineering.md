---
title: "Reverse Engineering"
date:  2025-01-15 00:00:00 +0000
categories: [Reverse Engineering]
tags: [Reverse Engineering]
---

# ANTI-DEBUG PE REVERSE-ENGINEERING

First off let me just say that ive been up for days trying to solve this fuckin crackme humza made. He is fucked for making this PE wicked in a debugger, and it was my first attempt at anti-debug analysis. The learning process was amazing though, cant knock that.

---

### Upon opening the Fucked PE in BINJA and inspecting the main function in the disassembly:

![Screenshot 2024-12-24 152241](https://github.com/user-attachments/assets/246b148e-a888-4914-bfab-c45527c0cf51)

Bam there it is staring back at us the reason our debugger goes to shit when executing this executable. So basically, whenever there is a debugger attached to the process when executing it, the program will know it is being debugged by a debugger and use an API call like `IsDebuggerPresent()` and `CheckRemoteDebuggerPresent()`; in kernel32.dll.

We see `NtSetInformationThread` and `NtQueryInformationThread` when scrolling through this main function code block and we know whenever these api's are called there most likely causing the debugger to stop debugging the process and cause it to crash the debugger. We know `NtSetInformationThread` has a parameter called `THREADINFOCLASS`, which contains `ThreadHideFromDebugger = 0x11`.

Why the fuck would windows let us use these api's??? Well, see here is why it exists: Whenever you attach a debugger to a remote process a new thread is created and if it was a normal thread the debugger would endlessly loop as it attempts to stop its own execution. Under the hood when a debugging thread is created Windows calls `NtSetInformationThread` with the flag set to `(1)` allowing the process to be debugged and continue as aspected. 

---

### Example of some c++ code for this method being used:

```c++
/*Timb3r's Code From GH*/

#include <stdio.h>
#include <windows.h>

enum THREADINFOCLASS { ThreadHideFromDebugger = 0x11 };

typedef NTSTATUS (WINAPI *NtQueryInformationThread_t)(HANDLE, THREADINFOCLASS, PVOID, ULONG, PULONG);
typedef NTSTATUS (WINAPI *NtSetInformationThread_t)(HANDLE, THREADINFOCLASS, PVOID, ULONG);

NtQueryInformationThread_t fnNtQueryInformationThread = NULL;
NtSetInformationThread_t fnNtSetInformationThread = NULL;

DWORD WINAPI ThreadMain(LPVOID p) {
        while(1) {

                if(IsDebuggerPresent())
                        asm("int3");
                Sleep(500);
        }
        return 0;
}


int main(void)
{
        DWORD dwThreadId = 0;
        HANDLE hThread = CreateThread(NULL, 0, ThreadMain, NULL, 0, &dwThreadId);

        HMODULE hDLL = LoadLibrary("ntdll.dll");
        if(!hDLL) return -1;

        fnNtQueryInformationThread = (NtQueryInformationThread_t)GetProcAddress(hDLL, "NtQueryInformationThread");
        fnNtSetInformationThread = (NtSetInformationThread_t)GetProcAddress(hDLL, "NtSetInformationThread");

        if(!fnNtQueryInformationThread || !fnNtSetInformationThread)
                return -1;

        ULONG lHideThread = 1, lRet = 0;

        fnNtSetInformationThread(hThread, ThreadHideFromDebugger, &lHideThread, sizeof(lHideThread));
        fnNtQueryInformationThread(hThread, ThreadHideFromDebugger, &lHideThread, sizeof(lHideThread), &lRet);

        printf("Thread is hidden: %s\n", val ? "Yes" : "No");
 
        WaitForSingleObject(hThread, INFINITE);
        return 0;
}
```
---

### Finding The Hidden System Threads!

Well now I know, ok I need to find this hidden thread somehow ... Low and behold c++ skills and some help from the lovely GuidedHacking forums. We can create a tool that will basiclly get system threads by using GetSystemTime, which is a funcion that retrieves sys timing info. The tool takes two snapshots of the timing info and compares them to spot if the difference is to large and flags them.

In the code we are going to pass three FILETIME pointers to the function: idle time, kernel time, and user time. We must initialize two SYSTEM_PROCESS_INFORMATION structs that store info about a process, such as the NUMBER OF THREADS USED!!!!

Mainly The two snapshots are used to calculate any spotted differences between the timing info by iterating through the process of both snapshots and matches them. The CPU time is being stored and then we are tracking the timing differences for each running process and then storing the overall CPU time difference by adding execution time from hidden processes. There is specific thresholds that cannot be exceeded or you will be flagged for hiding system threads.

---

---
### Driver Range Check Function
```c++
bool threadfinder::driver_range_check() {
	int* proc_list = nt::get_proc_list();
	SYSTEM_PROCESS_INFORMATION* proc = 0;
	for (proc = (SYSTEM_PROCESS_INFORMATION*)proc_list; proc->NextOffset != 0; proc = (SYSTEM_PROCESS_INFORMATION*)((char*)proc + proc->NextOffset)) {
		if (proc->ProcessId == 4) {
			for (int j = 0; j < proc->ThreadCount; j++) {
				if (!is_in_range((uintptr_t)proc->ThreadInfos[j].StartAddress)) {
					printf("Detection: %p not in range!", (uintptr_t)proc->ThreadInfos[j].StartAddress);
					return true;
				}

			}
			break;
		}
	}
	return false;
}
```
---

---
### 

This function retrieves the process list and loops through all the processes until the next process isn't 0. It checks to make sure that the process ID = 4, indicating that it is indeed a sys process. if it is equal to 4, it iterates through every thread in the process. is_in_range is then called to indicate if the start address of the thread is valid yk, and if the drivers threads do not fall into place, meaning it is indeed hidden bud -_-. 

---

### Defeating main.exe 

Well we see that the tool works.


![Screenshot 2024-12-24 171602](https://github.com/user-attachments/assets/5d6ea3a3-8f3c-4049-ba43-c062dcbdba09)

Next I started the program along with the main.exe as well and retrieved the flag!!!

![Screenshot 2024-12-24 172312](https://github.com/user-attachments/assets/dd0c127c-814b-4a43-afc9-0c1e70cecb3c)


---
### Credits/Sources

* [SystemThreadFinder](https://github.com/weak1337/SystemThreadFinder/tree/main)
* [Guided Hacking](https://guidedhacking.com)
* [Humza](https://humzak711.github.io/)

---
