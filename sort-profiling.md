The FreeBSD sort program reads lines from input or from a file and produces the same lines in sorted order. NetBSD has an equivalent program, as does the GNU Project. Recently, we received a bug report that described how FreeBSD sort was much slower than NetBSD sort and GNU sort (gsort).

The submitter gave some sample data:

```
time sort -n -k 1 -k 2 -k 3 pc-gff-stripped.bed > /dev/null                   # NetBSD
    1.409u 2.548s 0:03.95 99.7%     0+0k 0+0io 0pf+0w

time sort -n -k 1 -k 2 -k 3 pc-gff-stripped.bed > /dev/null                   # FreeBSD
    24.574u 0.795s 0:25.37 99.9%	55+172k 0+0io 0pf+0w

time gsort --parallel=1 -n -k 1 -k 2 -k 3 pc-gff-stripped.bed > /dev/null     # GNU
    3.005u 0.149s 0:03.16 99.3%	152+177k 6+0io 3pf+0w
```

Indeed, that's significantly slower! Intrigued, I first took a look at the source code for sort, but nothing seemed out of the ordinary. This meant I would need to use some profiling tools to identify the root of the problem.

Since I was testing sort with large files, I wondered if the program was spending a lot of time allocating memory, so I wanted to look at the number of allocations that the program was doing. This could be done by counting the number of calls to malloc, and also related functions like realloc. The dtrace tool allowed me to do this.

```
# dtrace -n 'pid$target::*alloc:entry {@ = count()}' -c "sort -n -o /dev/null test.txt"
```

This dtrace call produced one number, which is the total number of all calls to malloc, realloc, etc. For a 999999-line test file, I had a result of 40019544. That's four allocations per line in the input file! For reference, both NetBSD sort and GNU sort had fewer than 100 on the same input. Since allocations are expensive, we could definitely have a significant speed-up if we can allocate a pool of memory at once rather than individually for each line. Was the problem solved?

Not quite. I reached for my always-handy debugging tool, the humble print statement. I put one right before the call to `procfile`, the function where the majority of allocations take place during the initial setup, and one right after. Unfortunately, I could now see that that these four-million mallocs were occuring in the first few seconds, whereas the entire duration of the program was about 40 seconds. Reducing the number of mallocs could shave off about 5 seconds on this input, but it wasn't the bulk of the issue. I still had no idea which functions in the program were taking the longest time, and using dtrace to count function calls couldn't help.

The next thing I tried was the Callgrind tool in Valgrind. This profiling tool records counts of instructions executed, sorted according to which function the instruction occurs in. If we assume that each instruction takes the same amount of time, the function with the most instructions executed is also the function that spends the most time in the program. Although this isn't true, it is a good enough estimation.

Callgrind can be run as follows:

```
$ valgrind --tool=callgrind sort -n -o /dev/null test.txt
```

This generates a file called `callgrind.out.PID`. Callgrind also has a tool, `callgrind_annotate`, to display the contents of this file in a useful way.

```
$ callgrind_annotate callgrind.out.PID
```

I got some output that looked like this:

```
81,626,980,838 (25.33%)  ???:__tls_get_addr [/libexec/ld-elf.so.1]
59,017,375,040 (18.31%)  ???:___mb_cur_max [/lib/libc.so.7]
36,767,692,280 (11.41%)  ???:read_number [/usr/home/c/Repo/freebsd-src/usr.bin/sort/sort]
```

This means that about 25% of all instructions executed during execution of the program occurred in calls to the function `__tls_get_addr`, a remaining 18% occurred in `___mb_cur_max`, and finally, 11% occurred in `read_number`. Note that these counts are exclusive; for example, if `read_number` calls `___mb_cur_max`, the instructions in `___mb_cur_max` won't be added to the instructions in `read_number`. 

Just what are these functions? The two most-used functions, `__tld_get_addr` and `___mb_cur_max`, don't occur anywhere in the source code. It turns out that these two functions get called when the value `MB_CUR_MAX` is used. Although this value appears to be a compile-time constant, it isn't. 

Now I was getting somewhere, but I wanted to know more. Which functions were using `MB_CUR_MAX` so often, and why? The information from Callgrind couldn't tell me which functions actually resulted in these calls to `__tls_get_addr` and `___mb_cur_max`. 

Enter the stack trace. At a particular moment in time, the stack trace contains a history of how the program got to that point. If we take a picture of the stack repeatedly, we can get an idea of how much time the program is spending in each function, and where it's being called. The tool dtrace, again, allows us to do this. 

```
# dtrace -n 'profile-997 /pid == $target/{@[ustack()] = count()}' -c "sort -n -o /dev/null test.txt" -o out.stacks
```

This usage of dtrace records a stack trace at 997Hz for the entire duration of the program. Finally, the FlameGraph tool provides a useful, interactive, and very pretty visualization of the data:

```
$ ./stackcollapse.pl out.stacks > out.stacks_folded
$ ./flamegraph.pl out.stacks_folded > stacks.svg
```

Here is the resulting SVG.

The X-axis represents number of instructions, whereas the Y-axis represents the stack. Unlike Callgrind, this visualization is inclusive; all instructions that are executed during a function call count for that function. For example, we can see that `procfile` takes about 8% of the total instructions in the program, `sort_list_dump` takes about 10%, and `sort_list_to_file` takes about 80%. We can also see that three of these functions are executed from the main function.

This visualization affirmed my earlier hypothesis that the four million mallocs, all in `procfile`, were not significantly slowing the program. I also noticed a very long bar on the graph with a familiar name: `___mb_cur_max`. This function was taking up about 30% of the program's instructions, and it was also responsible for the calls to `__tls_get_addr`. It was being called by the function `read_number`. 

I now knew a very simple but effective optimization to this program. I would cache the value of `MB_CUR_MAX` at the beginning, so that further uses would not need to call the functions `___mb_cur_max` and `__tls_get_addr`. I implemented this change and noticed a significant improvement; it almost halved the runtime! 

Unfortunately the FlameGraph also told me that further improvements to the program would not be as easy. The majority of the remaining time is spent in the sorting function used, which comes from `libc`. And the majority of the time spent in the sorting function is in `list_coll_offset`, `numcoll_impl`, and `read_number`, which are used in the comparison function. Further optimizing this program would mean optimizing these functions, which isn't as easy as simply caching a value.

Still, we got a decent performance boost out of this exercise. And by using sort's flags to change up the sorting algorithm to quicksort or mergesort rather than heapsort, we have perfomance that's almost as good as GNU sort and NetBSD sort.
