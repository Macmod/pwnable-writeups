# Coin1

After some thought, we realize this problem is very easy to solve by hand.

We send some range of coins, say `0 1 2 3 4 5`, and wait for the result. If it's a round number, that is, a number which equals zero modulus 10, the fake coin isn't in this range.

We can then choose the obvious approach: let's split the set of possible coins into two sets, send the first half and wait for the result.

If the fake coin is in this range, split it again and repeat the process. If it isn't in this range, split the other half and repeat the process.

Eventually we end up with 2 or 3 coins, in which cases we just have to use basic logic to figure out the right one.

The problem is: we can't do this by hand, because we need 100 fake coins in a short amount of time, otherwise the server will disconnect us, so we gotta write a program.

This can be done with or without recursion, but considering the nature of the problem (binary search) I decided to take the recursive approach:

```python
import socket
import re

class CoinGame:
    @staticmethod
    def nums(res):
        nums = re.findall(b'\d+', res)
        return [int(r) for r in nums]

    def get_recv(self):
        while 1:
            res = self.s.recv(100)

            if res != b'':
                if res.startswith(b'Correct'):
                    self.count += 1
                    print('FOUND ' + str(self.count) + '\n')
                    return None
                elif res.startswith(b'Wrong'):
                    return None
                else:
                    nums = CoinGame.nums(res)
                    return nums if len(nums) else None

    def split_sub(self, a, b):
        lenr = len(range(a, b+1))
        if lenr > 3:
            half = int(a+lenr/2 - 1)
            if self.guess_range(a, half):
                return True
            else:
                return self.split_sub(half+1, b)
        elif lenr == 3:
            if self.guess_range(a, a+1):
                return True
            else:
                return self.guess_one(a+2)
        elif lenr == 2:
            if self.guess_one(a):
                return True
            else:
                return self.guess_one(a+1)

    def guess_one(self, num):
        return self.guess_range(num, num)

    def guess_range(self, left, right):
        coins = [str(v) for v in range(left, right+1)]
        if left != right:
            test = ' '.join(coins) + '\n'
        else:
            test = str(right) + '\n'

        print(str(left) + '-' + str(right) + '\n')
        self.s.sendall(str.encode(test))

        while 1:
            ans = self.get_recv()
            if ans is not None:
                if ans[0] == 9:
                    print('FOUND BEFORE')
                    return self.guess_range(left, right)
                elif ans[0] % 10 != 0:
                    print('DEEPER')
                    return self.split_sub(left, right)
                else:
                    print('ABORT')
                    return False
            else:
                return True

    def __enter__(self):
        return self

    def __exit__(self, typ, val, tb):
        self.s.shutdown(socket.SHUT_WR)
        self.s.close()

    def __init__(self):
        self.count = 0

        self.s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.s.connect(('localhost', 9007)) # pwnable.kr's server, port 9007
        self.s.recv(1102) # skip initial message


with CoinGame() as g:
    for i in range(100):
        cmd = g.get_recv()
        if len(cmd) == 2:
            g.split_sub(0, cmd[0]-1)

    # flag
    print(g.s.recv(1024))
```

We can now send this to pwnable.kr's tmp folder using `scp` and run it there to get the flag.
