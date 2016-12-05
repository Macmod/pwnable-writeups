# Mistake

A very common mistake among novice programmers is misuse of operator precedence.

If we take a look at the supplied code:

```c
int fd;
if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
    printf("can't open password %d\n", fd);
    return 0;
}

printf("do not bruteforce...\n");
sleep(time(0)%20);o

char pw_buf[PW_LEN+1];
int len;
if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
    printf("read error\n");
    close(fd);
    return 0;		
}

char pw_buf2[PW_LEN+1];
printf("input password : ");
scanf("%10s", pw_buf2);
```

It reads the password, waits some of amount of time < 20s and asks for the password. We can't read the password file.

Let's take a closer look at line `if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0)`. We realize that, once < has precedence over =, fd is going to be 1, `(int)true`, or 0, `(int)false`, depending on the result of the call to open.

Open returns the file descriptor associated with the file, which is a always a positive number. Therefore fd is going to be zero, `stdin`, and the call to read later on is going to read from `stdin` instead of the password file.

```c
// xor your input
xor(pw_buf2, 10);

if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
    printf("Password OK\n");
    system("/bin/cat flag\n");
}else{
    printf("Wrong Password\n");
}
```

Then we need to know what happens when we XOR our user-provided password string with 1.
Let's use `MYPASSWORD` and write a test program to see what happens.

```c
#include <stdio.h>
#define XORKEY 1

void xor(char* s, int len){
    int i;
    for(i=0; i<len; i++){
        s[i] ^= XORKEY;
    }
}

int main() {
    char p[11] = "MYPASSWORD";
    xor(p, 10);
    printf("%s\n", p);
}
```

Output: `LXQ@RRVNSE`

Now we can get our flag.

`$ (echo "MYPASSWORD"; sleep 1; echo "LXQ@RRVNSE") | ./mistake`
