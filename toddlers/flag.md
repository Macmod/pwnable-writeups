# Flag

> Papa brought me a packed present! let's open it.

This implies the binary is packed. Let's take a look.

```bash
$ gdb flag
>>> b main
No symble table is loaded. Use the "file" command.
```

We don't have any simbols, which makes reversing quite hard (although possible).
Let's take a look at the entry point address.

```bash
$ readelf -h flag | grep Entry
Entry point address:               0x44a4f0
$ gdb flag
>>> b *0x44a4f0
>>> r
Breakpoint 1, 0x000000000044a4f0 in ?? ()
>>> x/i $rip
=> 0x44a4f0:	call   0x44a770
```

There's a call to `0x44a770`. If we step into the call with `si` and keep hitting `ni` to go to the next instructions we can see only 8 instructions get executed directly after the call and we can't seem to find anything interesting inspecting the registers with `x` in the process.

Time for a different approach. Let's try and figure out how it's packed.

`$ strings flag > out`

Now let's open `out`. We are looking for metadata about the packer.

If we scroll down for some time or just search for `pack` we find this line:

> $Info: This file is packed with the UPX executable packer http://upx.sf.net $

Let's open UPX's page.

> UPX achieves an excellent compression ratio and offers **very fast decompression**. Your executables suffer no memory overhead or other drawbacks for most of the formats supported, because of in-place decompression.

Which means we can use it to unpack our binary. Let's install UPX and read the help page.

```bash
$ upx --help

                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2013
UPX 3.91        Markus Oberhumer, Laszlo Molnar & John Reiser   Sep 30th 2013

Usage: upx [-123456789dlthVL] [-qvfk] [-o file] file..

Commands:
  -1     compress faster                   -9    compress better
  --best compress best (can be slow for big files)
  -d     decompress                        -l    list compressed file
  -t     test compressed file              -V    display version number
  -h     give this help                    -L    display software license
...
```

Now we can unpack it and try again:

```bash
$ upx -d flag
$ gdb flag
>>> disas main
Dump of assembler code for function main:
   0x0000000000401164 <+0>:	push   rbp
   0x0000000000401165 <+1>:	mov    rbp,rsp
   0x0000000000401168 <+4>:	sub    rsp,0x10
   0x000000000040116c <+8>:	mov    edi,0x496658
   0x0000000000401171 <+13>:	call   0x402080 <puts>
   0x0000000000401176 <+18>:	mov    edi,0x64
   0x000000000040117b <+23>:	call   0x4099d0 <malloc>
   0x0000000000401180 <+28>:	mov    QWORD PTR [rbp-0x8],rax
   0x0000000000401184 <+32>:	mov    rdx,QWORD PTR [rip+0x2c0ee5]        # 0x6c2070 <flag>
   0x000000000040118b <+39>:	mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040118f <+43>:	mov    rsi,rdx
   0x0000000000401192 <+46>:	mov    rdi,rax
   0x0000000000401195 <+49>:	call   0x400320
   0x000000000040119a <+54>:	mov    eax,0x0
   0x000000000040119f <+59>:	leave  
   0x00000000004011a0 <+60>:	ret    
End of assembler dump.
```

There's a flag variable now which gets stored in `rdx` before calling strcpy at `0x00040195`. We can inspect it with x/s and get the flag:

`>>> x/s flag`
