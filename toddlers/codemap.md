# Codemap

This is a funny challenge.

We are told that the binary has a lot of information inside the heap. When we run it on Windows, we get this:

> I will make 1000 heap chunks with random size  
> each heap chunk has a random string  
> press enter to start the memory allocation  
>   
> the allocated memory size of biggest chunk is 99879 byte  
> the string inside that chunk is X12nM7yCJcu0x5u  
> log in to pwnable.kr and anwer some question to get flag.

When we connect to the daemon with `nc pwnable.kr 9021`, we get two questions:

1. What's inside the 2nd biggest chunk in the heap?
2. What's inside the 3rd biggest chunk in the heap?

But what does `codemap` mean? A quick google search brings up two possibilities:

- A VisualStudio feature for debugging
- An IDA plugin to facilitate analysis of complex binaries

Out of these two, only the latter is made, among others, by `daehee`, pwnable's owner! *I see what you did there.*

Let's read about it at [codemap.kr](http://codemap.kr/):

> Codemap is a binary analysis tool for "run-trace visualization" provided as IDA plugin.  
> Codemap uses 'breakpoints' for tracing the program. If the program hits a breakpoint, Codemap breakpoint handler is invoked as a callback function, then proper action for trace is taken and program continues.

Alright. Then we have to install `IDA` and `codemap`:

```bash
$ git clone https://github.com/c0demap/codemap
$ cd codemap; python install.py
Codemap install complete! run IDA Pro now.
```

Now let's open our binary in IDA.

As instructed, we set a breakpoint at `0x00403E65` with F2. Then we hit F4 to run it.

![ida-breakpoint](https://cloud.githubusercontent.com/assets/6147168/21038204/e7666852-bdba-11e6-8888-e4e990cbef6d.PNG)

We come back to IDA and use Alt+1 to start `codemap`.

A graph appears on the top of the page and we see information about the registers right below.

Let's put together a simple test query:

```sql
select eax from *traceX* order by eax desc
```

Where *traceX* is the default table `codemap` has already filled in the query textbox.

We get this:

![ida-eax](https://cloud.githubusercontent.com/assets/6147168/21038203/e765c5d2-bdba-11e6-80ff-787851fd6eb9.PNG)

We can see there are all kinds of values for eax within a range from `0x1b` (27) to `0x18627` (99879)
But wait... `ebx` looks like the allocated strings and the maximum `eax` matches the biggest chunk size!

We can now probably figure out the strings inside the 2nd and 3rd biggest chunks sending the right query:

```sql
select ebx from traceX order by eax desc limit 3
```

![ida-answer](https://cloud.githubusercontent.com/assets/6147168/21038202/e763d222-bdba-11e6-960c-d23066a73723.PNG)

We can now hover on top of the second and third points to see their `ebx` values:
`roKBkoIZGMUKrMb` and `2ckbnDUabcsMA2s`.

