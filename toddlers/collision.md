# Collision

Let's take a look at the code:

```c
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
    int* ip = (int*)p;
    int i;
    int res=0;
    for(i=0; i<5; i++){
        res += ip[i];
    }
    return res;
}

int main(int argc, char* argv[]){
    if(argc<2){
        printf("usage : %s [passcode]\n", argv[0]);
        return 0;
    }
    if(strlen(argv[1]) != 20){
        printf("passcode length should be 20 bytes\n");
        return 0;
    }

    if(hashcode == check_password( argv[1] )){
        system("/bin/cat flag");
        return 0;
    }else
        printf("wrong passcode.\n");
    return 0;
}
```

We need a 20-byte password resulting in `0x21DD09EC` after going through some hashing function.

```bash
$ readelf -h col | head
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
```

Since it's a 32-bit executable, pointers to `int` will have a size of 4 bytes.

Thus our hashing function takes our 20 bytes, casts the `char*` into an `int*` (5 chunks of 4 bytes) and sums the integers relative to each chunk.

Since our hashing function is not perfect, there will probably be many collisions. All we need is to find one.
Let's check the value of `0x21DD09EC` and divide it by five:

```bash
$ python -c "print(0x21DD09EC)"
568134124
$ python -c "print(0x21DD09EC/5)"
113626824.8
```

Our result is not a round number. But it doesn't have to be anyways. Let's use 4 chunks of 113626824 and one for the remainder, then the flag is ours:

```bash
$ python -c "print(0x21DD09EC - 113626824*4)"
113626828
$ python -c "print(hex(113626824), hex(113626828))"
0x6c5cec8 0x6c5cecc
$ ./col $(python -c "print('\xc8\xce\xc5\x06'*4+'\xcc\xce\xc5\x06')")
```

