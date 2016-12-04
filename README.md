# Pwnable Writeups
My personal writeups for [pwnable.kr](http://pwnable.kr/play.php).

Only Toddler's Bottle challenges are included ~~because I didn't solve the others yet~~ out of respect for Rule 3:

> 3\. Challenges in Toddler's Bottle are allowed to freely post the solutions online. However, please refrain from posting solution for challenges in other categories. But if you insist, post easy ones (solved by many people) and do not spoil too much details for the sake of fun.

----
## Tips

Suppose you are stuck but don't want to spoil the fun.

Here are some quick tips that may help you along the way:

### fd
Read wikipedia's article on [file descriptors](https://en.wikipedia.org/wiki/File_descriptor).

### collision
Find values that result in the hash after being summed up. Remember to input the result as [little endian](https://en.wikipedia.org/wiki/Endianness).

### bof
Read about buffer overflows in the classic [Smashing the Stack for Fun and Profit](http://insecure.org/stf/smashstack.html). Also, LiveOverflow's [playlists](https://www.youtube.com/watch?v=T03idxny9jE&index=13&list=PLhixgUqwRTjxglIswKp9mpkfPNfHkzyeN) are awesome.

### flag
You can't reverse a packed binary.

### passcode
Read about the Procedure Linkage Table (PLT) and the Global Offset Table (GOT).

This [article](http://blog.isis.poly.edu/exploitation%20mitigation%20techniques/exploitation%20techniques/2011/06/02/relro-relocation-read-only/) and this entry on [exploit-db](https://www.exploit-db.com/papers/13203/) are also very enlightening.

### random
Random values need proper seeding, otherwise they become [predictable](http://stackoverflow.com/questions/1108780/why-do-i-always-get-the-same-sequence-of-random-numbers-with-rand).

### input
Read about [command substitution](http://www.tldp.org/LDP/abs/html/commandsub.html), [I/O redirection](http://www.tldp.org/LDP/abs/html/io-redirection.html) and [netcat](https://www.g-loaded.eu/2006/11/06/netcat-a-couple-of-useful-examples/).

### leg
Learn a bit about [ARM](http://simplemachines.it/doc/arm_inst.pdf) to figure out the return values. Here's a [great manual](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289850039.htm).

### mistake
As the site says, read about [C operator's precedence](http://www.difranco.net/compsci/C_Operator_Precedence_Table.htm) to find out the mistake.

### shellshock
Read wikipedia's article on [shellshock](https://en.wikipedia.org/wiki/Shellshock_(software_bug)).

### coin1
Read about [binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm) (for the problem) and [sockets](https://docs.python.org/2/library/socket.html) (to programatically interact with the game).

### blackjack
It's nothing fancy, just a common logic mistake. Try to trick the game.

### lotto
It's nothing fancy, just a common logic mistake. Some very simple bruteforcing is needed (less than 50 tries).

### cmd1
Read wikipedia's article on [$PATH](https://en.wikipedia.org/wiki/PATH_(variable)).

### cmd2
Be creative with [bash](http://ss64.com/bash/). There's more than one solution.

### uaf
Read [this beginner's guide on Use-After-Free](http://garage4hackers.com/content.php?r=143-Beginners-Guide-to-Use-after-free-Exploits-IE-6-0-day-Exploit-Development) and [this whitepaper on Dangling Pointers](https://www.blackhat.com/presentations/bh-usa-07/Afek/Whitepaper/bh-usa-07-afek-WP.pdf).

### codemap
TODO

### memcpy
TODO

### asm
Read about shellcode creation. If you feel you don't quite get the SmashTheStack article yet, read this newbie-friendly guide:

[Writing 64-Bit Shellcode (Part 1)](http://null-byte.wonderhowto.com/how-to/writing-64-bit-shellcode-part-1-beginner-assembly-0161593/) & [Writing 64-Bit Shellcode (Part 2)](http://null-byte.wonderhowto.com/how-to/writing-64-bit-shellcode-part-2-removing-null-bytes-0161591/)

### unlink
TODO

----
## Thanks
![pusheen](https://media.tenor.co/images/550650fe51ac8b77091ce7292b7641ee/raw)

Special thanks to Ingrid Spangler for introducing me to this great hobby.
