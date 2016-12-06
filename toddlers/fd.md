# Fd

Let's take a look at the supplied code:

```c
int fd = atoi( argv[1] ) - 0x1234;
int len = 0;
len = read(fd, buf, 32);
if(!strcmp("LETMEWIN\n", buf)){
    printf("good job :)\n");
    system("/bin/cat flag");
    exit(0);
}
```

It reads from some file descriptor we control and compares the value to `LETMEWIN`.

Well, the file descriptor `0` refers to `stdin`, then we just have to provide `0`+`0x1234` to be able to input LETMEWIN.

```bash
$ python -c "print(0x1234)"
4660
$ ./fd 4660
LETMEWIN
```
