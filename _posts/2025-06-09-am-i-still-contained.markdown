---
layout: post
title: "Am I Still Contained?"
date: 2025-06-09 10:30:00 +0100
comments: false
categories: seccomp
---

This exploration started, as many do, with "huh that's odd". Specifically I was looking at the output of [amicontained](https://github.com/genuinetools/amicontained) around filtered syscalls.

```bash
Seccomp: filtering
Blocked Syscalls (54):
        MSGRCV SYSLOG SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT OPEN_BY_HANDLE_AT SETNS KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD
```

Looking at the SYSCALLS that were listed as blocked, I noticed that there wasn't any mention of IO_URING but I know that Docker [blocks io_uring syscalls in the default profile](https://github.com/moby/moby/pull/46762), so what's going on?

## Looking at the source code

I decided to take a look at the source code to see what was going on and why it might not be working. In the [seccompIter function](https://github.com/genuinetools/amicontained/blob/568b0d35e60cb2bfc228ecade8b0ba62c49a906a/main.go#L187) I found what looks like a relevant point. A for loop that iterates over each syscall one at a time.

```go
for id := 0; id <= unix.SYS_RSEQ; id++ 
```

The end point for the loop was a syscall called `SYS_RSEQ` and thanks to a very helpful lookup table [here](https://filippo.io/linux-syscall-table/) I could see that that's syscall `334`, and the IO_URING syscalls are 425-427, so we can see why they're not being flagged, the loop doesn't go that high!

## Fixing the problem

Whilst I'm not a professional developer by any stretch of the imagination (\<GEEK REFERENCE\> I'd liken myself to a rogue with the use magic device skill trying to get a wand of fireballs working by hitting the end of it \</GEEK REFERENCE\>), I decided to take a stab at fixing the code to get it to include the IO_URING syscalls (and any other ones with higher numbers).

We could just increase the maximum number on the for loop, but that does run into a problem, which is that there's a weird gap in the syscall numbers between 334 and 424. It appears that this was done to [sync up syscall numbers in different processor architectures](https://stackoverflow.com/a/63713244/537897), so we can just add a section to the code to skip those blank numbers.

The next tricky part is, it turns out making syscalls directly can sometimes cause the process to exit or hang. The original code has a number of [blocks designed to skip tricky syscalls](https://github.com/genuinetools/amicontained/blob/568b0d35e60cb2bfc228ecade8b0ba62c49a906a/main.go#L190)

```go
		// these cause a hang, so just skip
		// rt_sigreturn, select, pause, pselect6, ppoll
		if id == unix.SYS_RT_SIGRETURN || id == unix.SYS_SELECT || id == unix.SYS_PAUSE || id == unix.SYS_PSELECT6 || id == unix.SYS_PPOLL {
			continue
		}
```

Here the approach ended up being a bit trial and error on what syscalls caused problems. Also an interesting aside is that this shows a limitation of this approach to enumerating syscalls, it's not possible to get a definitive list as you can't probe for every possible syscall!

With that largely working, it was just a question of extending the really long [syscallName](https://github.com/genuinetools/amicontained/blob/568b0d35e60cb2bfc228ecade8b0ba62c49a906a/main.go#L243) function that has a case statement giving names for every syscall. This was also the only part of this that LLMs could help with (they got the main problem wildly wrong), and even here they only got most of it right.

After all that it looks like this largely works. As the original repository seems unmaintained, I've put a fork [here](https://github.com/raesene/amicontained) with the updated code.

## Results

Using the updated code in a Docker container we can see that the number of blocked syscalls has increased from 54 to 68, including the IO_URING ones that started this!

```bash
Blocked Syscalls (68):
        SYSLOG SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT OPEN_BY_HANDLE_AT SETNS KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD IO_URING_SETUP IO_URING_ENTER IO_URING_REGISTER OPEN_TREE MOVE_MOUNT FSOPEN FSCONFIG FSMOUNT FSPICK PIDFD_GETFD PROCESS_MADVISE MOUNT_SETATTR QUOTACTL_FD LANDLOCK_RESTRICT_SELF SET_MEMPOLICY_HOME_NODE
```

## Conclusion

This one was interesting for a number of reasons. First up was a good reminder that you can't rely on tools always working the way they used to, as the underlying systems change. The second one was that I learned quite a bit about the limitations of closed box testing of syscalls, and also as a side lesson, the current limitations of LLMs when dealing with relatively obscure lower level tech.