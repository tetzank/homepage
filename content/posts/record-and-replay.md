---
title: "Record And Replay"
date: 2019-08-31T19:53:49+02:00
---

Developers spent a significant percentage of their working time debugging their code.
A fact many people do not really like to acknowledge as it feels like being unproductive.
Software is complex.
One usually does not get it right the first time.
Therefore, it is very important to have good debugging tools supporting the developer in his quest to find the root cause of the issue.
Being able to step backwards in the program execution helps tremendously as we can retrace the execution flow from the crash or assertion failure back to the operation which corrupted the program state.



# Support in GDB

GDB has built-in support for recording the execution flow and [reversible debugging][gdbmanual].
Let us look at the following tiny program to learn the basic commands and get a feeling where reversible debugging might be helpful.

{{< highlight "C" >}}
#include <stddef.h>

int main(){
	long array[32];

	for(size_t i=0; i<sizeof(array); i++){
		array[i] = i;
	}

	return 0;
}
{{< / highlight >}}

This program does not compute anything at all, but it has a simple bug which illustrates the benefits of being able to step backwards in the program execution flow.
We compile it with the following command.

```
$ gcc -g -static -fno-stack-protector bug.c -o bug
```

We use `-g` to enable debug information, `-static` to build a static executable without any links to shared libraries like the libc, and `-fno-stack-protector` to disable any stack protection the compiler might add by default.
Some linux distributions enable stack protection by default to any compilation done on the platform.
We want to keep it simple which is why we disable it.

The `-static` is only necessary to work around a current limitation of GDB.
The resolver of dynamic symbols from shared libraries uses an instruction which is currently unsupported by the recording facilities in GDB.
A limitation which is hopefully fixed in future versions of GDB.

Now that we dealt with the compilation, let's start GDB and the recording.

```
$ gdb ./bug

-- snip --

(gdb) start
Temporary breakpoint 2 at 0x401bc0: file bug.c, line 25.
Starting program: /home/franky/programming/github/homepage/code/record-and-replay/bug 

Temporary breakpoint 2, main () at bug.c:25
25              for(size_t i=0; i<sizeof(array); i++){
(gdb) record
(gdb) c
Continuing.

[1]+  Stopped                 gdb ./bug
```

Recording can only be enabled on a running process.
We set a temporary breakpoint on _main_ and start execution until we reach it, all done with the command `start`.
Then, we enable recording of the execution flow with `record` and continue execution with `c` which is a short form of the `continue` command.

Well, now we got thrown out of the interactive GDB session for some reason.
We can bring it back to the foreground with the shell command `fg`.

```
$ fg
gdb ./bug
Process record: failed to record execution log.

Program stopped.
0x0000000000000023 in ?? ()
```

The program stopped at a very low address which GDB cannot map to any source code location.
It looks like the stack got corrupted somewhere.
If you get a coredump in such a corrupted state, you are most often out of luck to reason in any way about it and find the root cause of the memory corruption.

Luckily, we recorded the execution flow which means we have some magic up our sleeves.
First, let's step backwards until we reach a known location in our program.
`reverse-stepi` or just `rsi` steps a single instruction backwards.

```
(gdb) rsi
0x0000000000401bef in main () at bug.c:30
30      }
```

There we are! At the very end of _main_.
As we already suspected a stack corruption, let's have a look at the top of the stack.

```
(gdb) p /x $rsp
$1 = 0x7fffffffe638
(gdb) x 0x7fffffffe638
0x7fffffffe638: 0x00000023
```

The register _rsp_ points to the top of the stack.
We print the content of _rsp_ in hexadecimal with `p /x $rsp` which gives us the address of the current top of the stack.
We examine the memory location with `x 0x7fffffffe638`.
Voilà! There is the bogus low address again.

The return instruction at the end of _main_ reads the return address from the top of the stack to jump back to the caller of _main_, but at some point, this address was overwritten with this bogus value, leading to a segfault.
To find the location when the return address gets overwritten, we can make good use of watchpoints.

```
(gdb) set can-use-hw-watchpoints 0
(gdb) watch *0x7fffffffe638
Watchpoint 3: *0x7fffffffe638
(gdb) rc
Continuing.

Watchpoint 3: *0x7fffffffe638

Old value = 35
New value = 4203520
0x0000000000401bd2 in main () at bug.c:26
26                      array[i] = i;
(gdb) p i
$2 = 35
```

Hardware watchpoints do not work when we step backwards in a recorded execution flow.
We have to disable them.
Then, we set a watchpoint on the return address at the memory address `0x7fffffffe638`.
If any instruction modifies it, GDB will break on it.
Now, we continue stepping backwards in time until we break on the watchpoint with `reverse-continue` or just `rc`.

And there we are! At the root cause of the stack memory corruption!
When writing to `array` which is stored on the stack, we overwrite the return address.
A classic stack buffer overflow.
`i` is out of bounds, writing at position 35 in an array with 32 elements.

I guess, a lot of people already saw the problem in the code listing.
`sizeof(array)` returns the size in bytes and not the number of elements of the array.
The fix is up to the reader.

The overhead induced by the recording in GDB is the main reason why it is used very rarely.
It is not practical for any non-trivial program.
Nevertheless, it laid the ground work for other projects to base their work on.
The [rr-project][rr] tries to make recording and reverse execution efficient.


# The RR-Project

The main tool provided by the rr-project is called `rr`.
A debug session with `rr` is split in two phases.
First, the program execution is recorded and stored completely to disk.

```
$ rr record ./bug
```

If you get a fatal error talking about /proc/sys/kernel/perf_event_paranoid being too large, just write a smaller value to this file as root (`echo 1 >/proc/sys/kernel/perf_event_paranoid`) or alternatively use `rr record -n ./bug`.

Next, the recording can be replayed even on a different machine as often as you want, behaving exactly the same way as when recorded.
The program is not really executed during replay, e.g., I/O operations are not done again.
All side effects are simulated such that the debugged program behaves in the same way.

```
$ rr replay

-- snip --

0x0000000000401a90 in _start ()
(rr) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x0000000000000023 in ?? ()
```

`rr replay` implicitly opens the last recording and stops at the entry point of the program which is at the symbol _\_start_.
We continue execution with `c` to reach the end of the recording.

There we are again.
Getting a segfault when trying to execute code at 0x23.
You can use the same commands as in GDB to step backwards in the program execution, examine the stack and set watchpoints.
You do not have to disable hardware watchpoints in `rr`.
They work just fine.
Try it out for yourself.

The separation in the two phases _record_ and _replay_ makes it easier to ship this information from the user to the developers.
A recording is much better than backtraces and coredumps.
Furthermore, if you face a bug which only happens sometimes, you only have to record it once and be able to reproduce it every time by replaying the recording.

If you want to find out more about the inner workings of `rr` check out their [website][rr] and [paper][rrpaper].


# Conclusion

GDB and rr are not the only debugging tools which support recording and reversible debugging.
[UndoDB](https://www.undo.io/) is a commercial debugger for linux supporting efficient record and replay.
Microsoft calls it "[Time Travel Debugging](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview)".
It seems to be the right time to get familiar with these novel tools and features to improve the productivity of our debug sessions.
Debugging is usually not fun.
So, let's get it done quickly.


# References

- [GDB manual][gdbmanual]
- [rr-project website][rr]
- [rr-project paper][rrpaper]


[gdbmanual]: https://sourceware.org/gdb/onlinedocs/gdb/Reverse-Execution.html
[rr]: https://rr-project.org/
[rrpaper]: https://arxiv.org/abs/1610.02144
