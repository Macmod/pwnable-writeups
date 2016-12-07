# Input

We need to pass 5 stages. The first 3 deal only with input tricks, the fourth involves a file and the fifth involves sockets.

Let's take a look:

## Stage 1

```c
// argv
if(argc != 100) return 0;
if(strcmp(argv['A'],"\x00")) return 0;
if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
printf("Stage 1 clear!\n");	
```

First we need 99 parameters (the first index of argv is the command itself).

```bash
$ mkdir /tmp/inp
$ echo "./input $(seq -s' ' 99)" > /tmp/inp/st1
$ chmod +x /tmp/inp/st1
$ cat /tmp/inp/st1
./input 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99
```

Then we need some specific parameters. The hex representations of `A` and `B` are 0x41 and 0x42. Thus we need the 65th parameter to be `\x00` and the 66th parameter to be `\x20\x0a\x0d`. We can input characters from ascii using `$''` substitutions. Let's change our /tmp/inp/st1.

```bash
./input 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 $'\x00' $'\x20\x0a\x0d' 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99
```

And now:

```bash
$ /tmp/inp/st1
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
```

## Stage 2

```c
// stdio
char buf[4];
read(0, buf, 4);
if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
read(2, buf, 4);
if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");
```

Now we have to send some bytes to `stdin` (0) and `stderr` (2).
Let's write our desired strings to files, copy /tmp/inp/st1 into /tmp/inp/st2 and include the redirections.

```bash
$ echo -ne "\x00\x0a\x00\xff" > /tmp/inp/input
$ echo -ne "\x00\x0a\x02\xff" > /tmp/inp/error
```

```bash
./input 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 $'\x00' $'\x20\x0a\x0d' 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 < /tmp/inp/input 2< /tmp/inp/error
```

```bash
$ /tmp/inp/st2
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
/tmp/inp/st2: line 1: 46968 Segmentation fault [...]
```

We got a segfault, but that's because our env for the next stage doesn't exist yet.

## Stage 3

```c
if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
printf("Stage 3 clear!\n");
```

We have to set the `\xde\xad\xbe\xef` env variable to `\xca\xfe\xba\xbe`. Let's again copy our previous /tmp/inp/st2 into /tmp/inp/st3 and now include our env.

```bash
env $'\xde\xad\xbe\xef'=$'\xca\xfe\xba\xbe' ./input 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 $'\x00' $'\x20\x0a\x0d' 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 < /tmp/inp/input 2< /tmp/inp/error
```

```bash
$ /tmp/inp/st3
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
```

## Stage 4

```c
// file
FILE* fp = fopen("\x0a", "r");
if(!fp) return 0;
if( fread(buf, 4, 1, fp)!=1 ) return 0;
if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
fclose(fp);
printf("Stage 4 clear!\n");
```

Now we need to create a file with name `\x0a` containing 4 null bytes.
Since we can only write to `/tmp/*`, let's copy our /tmp/inp/st3 into /tmp/inp/st4 and change our executable path to ~/input. From now on we'll run the program from the /tmp/inp/ folder.

```bash
env $'\xde\xad\xbe\xef'=$'\xca\xfe\xba\xbe' ~/input 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 $'\x00' $'\x20\x0a\x0d' 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 < /tmp/inp/input 2< /tmp/inp/error
```

```bash
$ cd /tmp/inp/
$ echo -en "\x00\x00\x00\x00" > $'\x0a'
$ ./st4
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
bind error, use another port
```

And again the error relates to the next stage.

## Stage 5

```c
// network
int sd, cd;
struct sockaddr_in saddr, caddr;
sd = socket(AF_INET, SOCK_STREAM, 0);
if(sd == -1){
  printf("socket error, tell admin\n");
  return 0;
}
saddr.sin_family = AF_INET;
saddr.sin_addr.s_addr = INADDR_ANY;
saddr.sin_port = htons( atoi(argv['C']) );
if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
  printf("bind error, use another port\n");
      return 1;
}
listen(sd, 1);
int c = sizeof(struct sockaddr_in);
cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
if(cd < 0){
  printf("accept error, tell admin\n");
  return 0;
}
if( recv(cd, buf, 4, 0) != 4 ) return 0;
if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
printf("Stage 5 clear!\n");
```

It receives a port to listen on in the 'C'th parameter (67th parameter), receives 4 bytes and compares them to `\xde\xad\xbe\xef`. We just have to provide some port and use `netcat` to pipe in our bytes.

Let's copy /tmp/inp/st4 into /tmp/inp/st5 and include some acceptable port, like 1337.

```bash
env $'\xde\xad\xbe\xef'=$'\xca\xfe\xba\xbe' ~/input 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 $'\x00' $'\x20\x0a\x0d' 1337 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 < /tmp/inp/input 2< /tmp/inp/error
```

Now we run the program while waiting to send our bytes in background:

```bash
$ ./st5 & (sleep 2; echo -en "\xde\xad\xbe\xef" | nc 0 1337)
[1] 61347
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
Stage 5 clear!
[1]+  Done                    ./st5
```

Since it calls `/bin/cat flag` and we don't have the flag in our /tmp directory, lastly we need to symlink it:

```bash
$ ln -s ~/flag flag
$ ./st5 & (sleep 2; echo -en "\xde\xad\xbe\xef" | nc 0 1337)
```

