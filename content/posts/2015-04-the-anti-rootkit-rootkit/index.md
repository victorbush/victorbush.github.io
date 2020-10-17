---
title: "The Anti-Rootkit Rootkit"
date: 2015-04-21T11:21:23-05:00
draft: false
---

I was recently analyzing a piece of malware that used a user-mode rootkit to hide itself in the process list. This makes the malware difficult to actively debug. I wondered if I could change the tides by using my own simple rootkit to UN-hide the malware. Turns out it’s possible.

## Brief

All analysis is performed in an isolated virtual machine for safety. This project utilized the following:

* Oracle VirtualBox
* Windows XP SP3 (x86)
* Visual Studio 2010
* Windows Driver Kit Version 7.1.0

I can’t guarantee that the procedures outlined here will work with newer (or 64-bit) versions of Windows. These techniques use offsets into kernel structures that are (most likely) specific to certain versions of Windows – you would need determine the appropriate offsets for your target platform. WinDbg can be used for such a task.

Is this project actually useful for anything? I don’t know, but you feel cool doing it.

## Source Code

The project source can be found here: [https://bitbucket.org/victorbush/dkom.prochide/overview](https://bitbucket.org/victorbush/dkom.prochide/overview)

## Primary References

There are two primary references I used for this project:

* Malware Analyst’s Cookbook [1]
* Alex Ionescu’s blog on OpenRCE [2]

These got me going in the right direction. There’s also a nice Blackhat presentation on DKOM by Jaime Butler that has relevant information [3].

## Windows Processes

Each process in Windows is given an _EPROCESS structure. These structures are in non-paged areas of kernel-allocated memory [1]. These _EPROCESS structures track many things about the process, such as the process id, parent process, creation time, etc. The table below shows the relevant objects inside the _EPROCESS structure and their offsets.

{{< img src="eprocess.png" link="eprocess.png" caption="_EPROCESS Structure" >}}

Important for our purposes is the ActiveProcessLinks object. This object contains two pointers, a forward link (Flink) and backward link (Blink). These pointers form a doubly-linked list of all running processes. The Flink of one process points to the Flink of the next process – this repeats back to the first process in the list, forming a circle. The Blink of one process points to the Flink of the previous process (note that it does NOT point to the Blink of the previous process).

Here is an example of how Flinks and Blinks point. The value boxes are color coordinated with the memory addresses they point to.

{{< img src="eprocess_ex.png" link="eprocess_ex.png" caption="Example of Flink and Blink pointers." >}}

## Hiding a Process

To hide a process, one can modify the Flinks and Blinks of _EPROCESS blocks to remove a process from the active process chain. Take the previous example, for instance. Let’s hide Process B.

{{< img src="eprocess_ex_2.png" link="eprocess_ex_2.png" caption="Example of a hidden process using modified Flink/Blink pointers." >}}

Notice the two values that have been changed – Process A’s Flink was updated to point to Process C, and Process C’s Blink was updated to point to Process A. Process B is now hidden.

The question is why does the process continue to run? Because the scheduling for Windows is done based on threads, *not* processes [3].

## Manipulating the _EPROCESS Structure

The _EPROCESS structure resides in kernel memory. Typically one would need to write a driver to get kernel-level access. However, there is a native API function called ZwSystemDebugControl that allows us to read and write kernel memory from user-mode applications [1].

The procedure for using ZwSystemDebugControl is outlined nicely in the *Malware Analyst’s Cookbook* [1] with code cross-reference to the OpenRCE blog [2].

### 1. Get debug privileges

First we need to get proper access by using the RtlAdjustPrivilege function with the SE_DEBUG_PRIVILEGE option.

```
BOOLEAN old;
RtlAdjustPrivilege(SE_DEBUG_PRIVILEGE, TRUE, FALSE, &old);
```

### 2. Find the kernel execute module in memory

The kernel execute module exports a global variable called PsInitialSystemProcess. As you might have guessed, it is the initial system process – more specifically, it points to the address of the _EPROCESS block for that process. It serves as our entry point into the process chain.

The NtQuerySystemInformation function can get the SystemModuleInformation structure, which will contain the address to the kernel execute module.

```
RTL_PROCESS_MODULES moduleInfo;
NtQuerySystemInformation(SystemModuleInformation, &moduleInfo, sizeof(moduleInfo), NULL);
```

We can now extract the base address of the kernel execute module:

```
ntoskrnlBase = (ULONG)moduleInfo.Modules[0].ImageBase;
```

Now we can get a handle to the module with LoadLibraryA using the extracted address.

### 3. Get PsInitialSystemProcess

With the handle to the kernel module, get the address of the PsInitialSystemProcess variable with GetProcAddress.

```
ntoskrnl = LoadLibraryA(ntoskrnlFileName);
varAddr = (ULONG)GetProcAddress(ntoskrnl, "PsInitialSystemProcess") – (ULONG)ntoskrnl + ntoskrnlBase;
```

Now that we have the address to PsInitialSystemProcess, we can read it’s memory to get the address to the _EPROCESS block for the initial system process. This is done using the NtSystemDebugControl function. The OpenRCE blog provides a wrapper function called ReadKernelMemory around NtSystemDebugControl that makes it easier to use.

```
NTSTATUS
ReadKernelMemory(IN PVOID BaseAddress, OUT PVOID Buffer, IN ULONG Length) {
    NTSTATUS Status;
    SYSDBG_VIRTUAL DbgMemory;
 
    DbgMemory.Address = BaseAddress;
    DbgMemory.Buffer = Buffer;
    DbgMemory.Request = Length;
 
    Status = NtSystemDebugControl(SysDbgReadVirtual,
        &DbgMemory,
        sizeof(DbgMemory),
        NULL,
        0,
        NULL);
    return Status;
}
```

We use this function to read the PsInitialSystemProcess variable:

```
ReadKernelMemory((PVOID)varAddr,
        &PsInitialSystemProcess,
        sizeof(PsInitialSystemProcess));
```

### 4. Do the un-hiding

We take as input the virtual address of the process we want to unhide and the process id of a running process (one that isn’t hidden). We will use this running process to insert our hidden process back in the process chain by using and modifying the values of its Flink and Blink pointers.

Check out the source for the full unhide function, but here is a snippet:

```
// Store address to my flink variable (not this is not the value stored in my Flink)
myFlinkAddr = addr + activeProcLinksOffset;
    
// Get address to EPROCESS block of process that we are going to insert our process AFTER
if (!getEprocessAddr(procIdPrev, &prevProcAddr)) {
    printf("Error: could not find process id %lu.\r\n", procIdPrev); return FALSE; 
}
prevProcFlinkAddr = prevProcAddr + activeProcLinksOffset;
 
// Get address to the Flink of the process that we are going to insert our process BEFORE
// This is pointed to by the Flink of prevProc
status = ReadKernelMemory((PVOID)(prevProcFlinkAddr),
    &nextProcFlinkAddr,
    sizeof(nextProcFlinkAddr));
if (!NT_SUCCESS(status)) { printf("Error reading Flink.\r\n"); return FALSE; }
 
// Update the prev process Flink value to point to my Flink addr
status = WriteKernelMemory((PVOID)(prevProcFlinkAddr),
    &myFlinkAddr,
    sizeof(myFlinkAddr));
if (!NT_SUCCESS(status)) { printf("Error updating prev process Flink.\r\n"); return FALSE; }
 
// Update my Flink value to point to next process Flink addr
status = WriteKernelMemory((PVOID)(myFlinkAddr),
    &nextProcFlinkAddr,
    sizeof(nextProcFlinkAddr));
if (!NT_SUCCESS(status)) { printf("Error updating my process Flink.\r\n"); return FALSE; }
 
// Update next proc Blink value to point to my Flink addr
status = WriteKernelMemory((PVOID)(nextProcFlinkAddr + 0x4),
    &myFlinkAddr,
    sizeof(myFlinkAddr));
if (!NT_SUCCESS(status)) { printf("Error updating next process Blink.\r\n"); return FALSE; }
 
// Update my Blink value to point to prev proc Flink addr
status = WriteKernelMemory((PVOID)(myFlinkAddr + 0x4),
    &prevProcFlinkAddr,
    sizeof(prevProcFlinkAddr));
if (!NT_SUCCESS(status)) { printf("Error updating my process Blink.\r\n"); return FALSE; }
```

## Quick Test

The dkom_prochide source includes a few functions that come in handy:

* Hide a process given a process id.
* Un-hide a process given the address to the _EPROCESS block of a hidden process.
* Get the address to the _EPROCESS block of a process given a process id.
* Get the process id of a process given an address to its _EPROCESS

Here is a quick test to verify everything is working. Grab a copy of Process Hacker (http://processhacker.sourceforge.net/) as it will come in handy.

First, start up a process – in this case, Solitaire.

{{< img src="sol-start2.png" link="sol-start2.png" caption="Solitaire start up." >}}

Let’s find the address to the _EPROCESS block for that process.

```
dkom_prochide geteprocaddr 2812
```

{{< img src="geteprocaddr.png" link="geteprocaddr.png" caption="Get the _EPROCESS address." >}}

Now, let’s hide the process.

```
dkom_prochide hide 2812
```

{{< img src="hide.png" link="hide.png" caption="Hide the process." >}}

The process no longer shows up in the process list.

{{< img src="hidden.png" link="hidden.png" caption="Process is hidden." >}}

You can get the process id for the hidden process by specifying its _EPROCESS address.

```
dkom_prochide getprocidbyaddr 0x86a7cda0
```

{{< img src="getprocidbyaddr.png" link="getprocidbyaddr.png" caption="Get process id by _EPROCESS address." >}}

Now let’s un-hide the process. We’ll re-insert the process into the process chain after process id 252.

```
dkom_prochide unhide 0x86a7cda0 252
```

{{< img src="unhide.png" link="unhide.png" caption="Un-hide the process." >}}

The process is now back in the list:

{{< img src="unhidden.png" link="unhidden.png" caption="The process is now un-hidden." >}}

## Using our Anti-Rootkit Rootkit

The test example is great, but let’s look at some real-world usage. Unfortunately, I did not code the ability to search for hidden processes and retrieve the addresses to their _EPROCESS blocks automatically. This may be possible – whether or not you can do it in user-mode with native APIs, I do not know. However, there are tools we can use when analyzing malware that have a hidden process that we want to debug.

For this example, we will look at a real piece of malware. It unpacks itself at runtime, puts the unpacked code into memory in a new thread, and hides its process. We want to debug this process while it is running. I’m not sure if there are debuggers that can attach to hidden processes or not. OllyDbg could not (unless there’s a plugin, I’m not sure). Anyway, let’s continue.

Process Hacker has the ability to search for hidden processes.

{{< img src="prochacker-hide.png" link="prochacker-hide.png" caption="Process Hacker Hidden Processes." >}}

Click the “Scan” button to scan for hidden processes. They will show up highlighted in red. Below is the hidden malware process.

{{< img src="malware-hidden.png" link="malware-hidden.png" caption="The hidden malware process." >}}

Double clicking on an entry in the list will give you plenty of details about the process; however, it does not give you the address to the _EPROCESS block, which we need to un-hide the process.

Since we are analyzing in a virtual machine, we can dump the memory of the machine and analyze it in a frozen state. The Volatility framework (https://github.com/volatilityfoundation/volatility) is a great memory forensics framework for doing just that.

First, we need to dump the virtual machine’s memory.

For VirtualBox, get your virtual machine running to the point where you want to dump the memory. In the host system, use the following command to dump the memory to a file on the host system.

```
vboxmanage debugvm “Name of Virtual Machine” dump guestcore –filename memdump.elf
```

{{< img src="vmdump.png" link="vmdump.png" caption="VirtualBox memory dump." >}}

Now use the Volatility framework to analyze the dumped memory. The psscan process searches memory dumps for _EPROCESS structures (rather than following the process chain) to find the list of processes. You need to make sure to use the –V switch to display virtual memory addresses instead of physical ones.

```
python vol.py psscan –f “path-to-mem-dump-file” -V
```

{{< img src="vol-psscan.png" link="vol-psscan.png" caption="Volatility psscan command." >}}

Search the results for the process you are trying to unhide, and note it’s virtual address.

{{< img src="vol-entry.png" link="vol-entry.png" caption="The hidden process found by psscan." >}}

Now that we have the address, we can use our dkom_prochide utility to un-hide the process.

```
dkom_prochide unhide 0x86b4c178 252
```

{{< img src="malware-unhide.png" link="malware-unhide.png" caption="Un-hide the malware process." >}}

Now the malware’s process is back in the process list.

{{< img src="malware-unhidden.png" link="malware-unhidden.png" caption="Malware is now un-hidden." >}}

The process also shows up in OllyDbg’s attach window, so we can attach to and debug the process.

{{< img src="ollyattach.png" link="ollyattach.png" caption="Malware now in OllyDbg attach window." >}}

## Hide the Debugger from Malware

The same malware we just looked at constantly scans the process list for certain process names it has on a blacklist and kills those processes (OllyDbg is one of them). Now, you could rename the executable. In OllyDbg’s case, there are some plugins that don’t work when you do this. So instead, why not hide OllyDbg with our little rootkit?

{{< img src="ollyhide.png" link="ollyhide.png" caption="OllyDbg hidden." >}}

Now we can run the malware, perform our un-hide process, and debug with Olly!

## Parting Words

That’s it, have fun, be safe. Also, obligatory disclaimer: I take no responsibility for anything you do with the code provided.

## References

[1] M. Ligh, S. Adair, B. Hartstein and M. Richard, Malware Analyst’s Cookbook and DVD: Tools and Techniques for Fighting Malicious Code, Indianapolis, IN: Wiley, 2010.

[2] A. Ionescu, “Tips & Tricks Part 2 – Putting ZwSystemDebugControl to good use”. http://www.openrce.org/blog/view/354/Tips_&Tricks_Part_2-_Putting_ZwSystemDebugControl_to_good_use.

[3] J. Butler, “DKOM (Direct Kernel Object Manipulation)”. http://www.blackhat.com/presentations/win-usa-04/bh-win-04-butler.pdf.