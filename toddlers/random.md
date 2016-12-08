# Random

```c
#include <stdio.h>

int main(){
    unsigned int random;
    random = rand();	// random value!

    unsigned int key=0;
    scanf("%d", &key);

    if((key ^ random) == 0xdeadbeef){
        printf("Good!\n");
        system("/bin/cat flag");
        return 0;
    }

    printf("Wrong, maybe you should try 2^32 cases.\n");
    return 0;
}
```

Our random value was not seeded with `srand`, therefore it's not going to be random.
Let's write a check program into /tmp/rand:

```c
#include <stdlib.h>
#include <stdio.h>

void main() {
    printf("%d\n", rand());
}
```

Output: `1804289383`

We can confirm it doesn't change if we run multiple times.

Let's XOR it with `0xdeadbeef`:

```c
#include <stdlib.h>
#include <stdio.h>

void main() {
    printf("%d\n", rand() ^ 0xdeadbeef);
}
```

Output: `-1255736440`

We can now input this number into the program and get the flag.
