# Memcpy

```c
char* slow_memcpy(char* dest, const char* src, size_t len){
    int i;
    for (i=0; i<len; i++) {
        dest[i] = src[i];
    }
    return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
    size_t i;
    // 64-byte block fast copy
    if(len >= 64){
        i = len / 64;
        len &= (64-1);
        while(i-- > 0){
            __asm__ __volatile__ (
                    "movdqa (%0), %%xmm0\n"
                    "movdqa 16(%0), %%xmm1\n"
                    "movdqa 32(%0), %%xmm2\n"
                    "movdqa 48(%0), %%xmm3\n"
                    "movntps %%xmm0, (%1)\n"
                    "movntps %%xmm1, 16(%1)\n"
                    "movntps %%xmm2, 32(%1)\n"
                    "movntps %%xmm3, 48(%1)\n"
                    ::"r"(src),"r"(dest):"memory");
            dest += 64;
            src += 64;
        }
    }

    // byte-to-byte slow copy
    if(len) slow_memcpy(dest, src, len);
    return dest;
}

int main(void){
    // [...]
    src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

    size_t sizes[10];
    int i=0;

    // setup experiment parameters
    for(e=4; e<14; e++){	// 2^13 = 8K
        low = pow(2,e-1);
        high = pow(2,e);
        printf("specify the memcpy amount between %d ~ %d : ", low, high);
        scanf("%d", &size);
        if( size < low || size > high ){
            printf("don't mess with the experiment.\n");
            exit(0);
        }
        sizes[i++] = size;
    }

    sleep(1);
    printf("ok, lets run the experiment with your configuration\n");
    sleep(1);

    // run experiment
    for(i=0; i<10; i++){
        size = sizes[i];
        printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
        dest = malloc( size );

        memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
        t1 = rdtsc();
        slow_memcpy(dest, src, size);		// byte-to-byte memcpy
        t2 = rdtsc();
        printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

        memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
        t1 = rdtsc();
        fast_memcpy(dest, src, size);		// block-to-block memcpy
        t2 = rdtsc();
        printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
        printf("\n");
    }

    printf("thanks for helping my experiment!\n");
    printf("flag : ----- erased in this source code -----\n");
    return 0;
}
```

It seems understandable. There's a `fast_memcpy`, which copies 64-byte blocks using some instructions we don't know, and a `slow_memcpy` which simply performs a byte-to-byte copy. The `fast_memcpy` only works

Let's compile and run it locally. Remember to compile with `-m32` as instructed in the code:

```bash
$ gcc -lm -m32 memcpy.c -o memcpy
$ ./memcpy
[...]
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 64
specify the memcpy amount between 128 ~ 256 : 128
specify the memcpy amount between 256 ~ 512 : 256
specify the memcpy amount between 512 ~ 1024 : 512
specify the memcpy amount between 1024 ~ 2048 : 1024
specify the memcpy amount between 2048 ~ 4096 : 2048
specify the memcpy amount between 4096 ~ 8192 : 4096
ok, lets run the experiment with your configuration
[...]
experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 827
[1]    20632 segmentation fault (core dumped)  ./memcpy
```

We got a segfault, but why? Let's check:

```bash
$ for i in {1..10}; do echo $((2**(i+2))); done > input
$ gdb memcpy
>>> disas fast_memcpy
[...]
   0x080486e9 <+33>:	movdqa xmm0,XMMWORD PTR [eax]
   0x080486ed <+37>:	movdqa xmm1,XMMWORD PTR [eax+0x10]
   0x080486f2 <+42>:	movdqa xmm2,XMMWORD PTR [eax+0x20]
   0x080486f7 <+47>:	movdqa xmm3,XMMWORD PTR [eax+0x30]
   0x080486fc <+52>:	movntps XMMWORD PTR [edx],xmm0
   0x080486ff <+55>:	movntps XMMWORD PTR [edx+0x10],xmm1
   0x08048703 <+59>:	movntps XMMWORD PTR [edx+0x20],xmm2
   0x08048707 <+63>:	movntps XMMWORD PTR [edx+0x30],xmm3
[...]
>>> b *fast_memcpy+33
Breakpoint 1 at 0x80486e9
>>> r < input
Breakpoint 1, 0x080486e9 in fast_memcpy ()
>>> p/x $eax
$1 = 0xf7fca000
>>> p/x $edx
$2 = 0x804d060
>>> c
Breakpoint 1, 0x080486e9 in fast_memcpy ()
>>> p/x $edx
$3 = 0x804d0a8
```

The source/destination addresses are valid, but `experiment 4` successfully copies to `0x804d060` while `experiment 5` does not to `0x804d0a8`. What's happening?

As we don't understand `fast_memcpy` yet, we should read about [MOVDQA](http://www.felixcloutier.com/x86/MOVDQA.html) and [MOVNTPS](http://www.felixcloutier.com/x86/MOVNTPS.html) before proceeding:

## MOVDQA

> Moves 128 bits of packed integer values from the source operand (second operand) to the destination operand.  
> When the source or destination operand is a memory operand, the operand must be aligned on a 16-byte boundary or a general-protection exception (#GP) will be generated.

## MOVNTPS

> Moves the packed single-precision floating-point values in the source operand (second operand) to the destination operand (first operand) using a non-temporal hint to prevent caching of the data during the write to memory.  
> The memory operand must be aligned on a 16-byte (128-bit version) or 32-byte (VEX.256 encoded version) boundary otherwise a general-protection exception (#GP) will be generated.

Wait: `0x804d0a8` is not a multiple of 16! According to [glibc](http://www.delorie.com/gnu/docs/glibc/libc_31.html), the default alignment for 32-bit is always 8, not necessarily 16, as opposed to 64-bit, so this must be it!

We can check this compiling *without* the `m32` flag to confirm it works:

```bash
$ gcc -lm memcpy.c -o memcpy64
$ ./memcpy64 < input
[...]
experiment 10 : memcpy with buffer size 4096
ellapsed CPU cycles for slow_memcpy : 20742
ellapsed CPU cycles for fast_memcpy : 1160

thanks for helping my experiment!
flag : ----- erased in this source code -----
```

All we have to do is shift the size of the allocated chunks in our input to guarantee the correct alignment. Let's add 8 to the 4th chunk size so that our 5th chunk is placed into an aligned address.

```bash
$ cat input
8
16
32
72
128
256
512
1024
2048
4096
$ ./memcpy < input
[...]
experiment 6 : memcpy with buffer size 256
ellapsed CPU cycles for slow_memcpy : 2750
[1]    3065 segmentation fault (core dumped)  ./memcpy < input
```

We got a segfault at the 6th experiment now. Let's again add 8 to the 5th chunk size and keep repeating this process.

```bash
$ sed -i s/128/136/ input
$ ./memcpy < input
[...]
experiment 7 : memcpy with buffer size 512
ellapsed CPU cycles for slow_memcpy : 3205
[1]    13867 segmentation fault (core dumped)  ./memcpy < input
$ sed -i s/256/264/ input
$ ./memcpy < input
[...]
experiment 8 : memcpy with buffer size 1024
ellapsed CPU cycles for slow_memcpy : 6849
[1]    23437 segmentation fault (core dumped)  ./memcpy < input
$ sed -i s/512/520/ input
$ ./memcpy < input
[...]
experiment 9 : memcpy with buffer size 2048
ellapsed CPU cycles for slow_memcpy : 11373
[1]    25957 segmentation fault (core dumped)  ./memcpy < input
$ sed -i s/1024/1032/ input
$ ./memcpy < input
[...]
experiment 10 : memcpy with buffer size 4096
ellapsed CPU cycles for slow_memcpy : 18641
[1]    31010 segmentation fault (core dumped)  ./memcpy < input
$ sed -i s/2048/2056/ input
$ ./memcpy < input
[...]
thanks for helping my experiment!
flag : ----- erased in this source code -----
```

We can now use our input file in pwnable's server to get the flag.
