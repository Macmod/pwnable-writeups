# Leg

In this challenge we have to learn some ARM to figure out the return values of `key1`, `key2` and `key3`, which summed give the password for the flag.

First we gotta know what a return value looks like in ARM.

Let's take a look at the provided disassembly:

```asm
[...]
0x00008d68 <+44>:	bl	0x8cd4 <key1>
0x00008d6c <+48>:	mov	r4, r0
0x00008d70 <+52>:	bl	0x8cf0 <key2>
0x00008d74 <+56>:	mov	r3, r0
0x00008d78 <+60>:	add	r4, r4, r3
0x00008d7c <+64>:	bl	0x8d20 <key3>
0x00008d80 <+68>:	mov	r3, r0
0x00008d84 <+72>:	add	r2, r4, r3
0x00008d88 <+76>:	ldr	r3, [r11, #-16]
0x00008d8c <+80>:	cmp	r2, r3
0x00008d90 <+84>:	bne	0x8da8 <main+108>
[...]
```

After branching to the key functions, `r0` is always stored in some register, then it must be the return value. After `key1`, the return is stored in `r4`. Then, after `key2`, it's stored in `r3` and `r3` is added to `r4`. Finally, after `key3`, it's again stored in `r3` and now the final result, `r4`+`r3`, is stored in `r2`.

Then some value is loaded into `r3` (the provided password) and `r2` is compared to `r3`.

Sounds good, right?

Now let's take a look at the code for each key:

## key1
```asm
[...]
0x00008cdc <+8>:	mov	r3, pc
0x00008ce0 <+12>:	mov	r0, r3
[...]
```

The return value is `pc`, the `Program Counter`. What's that? Let's read on [StackOverflow](http://stackoverflow.com/questions/18330902/program-counter-in-arm-assembly):

> For ARM mode:  
> When using R15 as the base register you must remember it contains an address 8 bytes on from the address of the current instruction.  
> For Thumb mode:  
> The value of the PC will be 4 bytes greater than the address of this instruction, but bit 1 of the PC is forced to 0 to ensure it is word aligned.

Thus it's the address of the current instruction `+8`, which here is `0x00008ce4` (36068).

## key2
```asm
[...]
0x00008cfc <+12>:	add    r6, pc, #1
0x00008d00 <+16>:	bx     r6
0x00008d04 <+20>:	mov    r3, pc
0x00008d06 <+22>:	adds   r3, #4
[...]
0x00008d10 <+32>:	mov    r0, r3
[...]
```

Here we need some extra research. First we store `pc + 1` (`0x00008d05`) into `r6`, then we have a `bx r6`.
Let's read about `bx` in the [manual](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0040d/Cabdcdci.html):

> Syntax  
> The syntax of BX is one of:  
> 1. Thumb (BX Rn)  
> 2. ARM (BX{cond} Rn)  
> Where:  
> Rn is a register in the range r0 to r15 that contains the address to branch to. The value of bit 0 in this register determines the processor state:  
> if bit 0 is set, the instruction at the branch address is executed in Thumb state  
> if bit 0 is clear, the instruction at the branch address is executed in ARM state.

We now know the instruction at the branch address, `0x00008d04`, is going to be executed in Thumb state, which from our previous reading means the `pc` is going to be 4 bytes on from the current instruction. Then it stores our new `pc` (`0x00008d08`) into `r3`, adds 4 to `r3` to get `0x00008d0c` and returns this value (36108).

## key3
```asm
[...]
0x00008d28 <+8>:    mov r3, lr
0x00008d2c <+12>:   mov r0, r3
[...]
```

Here the return value is `lr`, the `Link Register`. It stores the return address of the function, which is `0x00008d80` (36224), from the first disassembly.

We can now sum 36224+36108+36068 to get our answer: 108400.
