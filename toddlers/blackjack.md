# Blackjack

From inspecting the original program, we can see that there are some mistakes, as one user of the forum pointed out:

> You're initially dealt 2 cards, before you hit or stay, not one. If you go over 21, and the dealer goes over 21, you do not win. Only the player with the highest hand, but still under 21, can win when the dealer goes bust.

But `pwnable` says they only give flags to millionaires. Thus unless these mistakes are very easily exploitable this is gonna take some time.

Let's read the `betting` function:

```c
if (bet > cash) //If player tries to bet more money than player has
{
  printf("\nYou cannot bet more money than you have.");
  printf("\nEnter Bet: ");
  scanf("%d", &bet);
  return bet;
}
```

If we bet more than we have the program prompts for a second amount and then happily accepts our input without re-checking.

We just have to win once!

Better yet, if we go to the beginning of the program:

```c
int k;
int l;
int d;
int won;
int loss;
int cash = 500;
int bet;
int random_card;
int player_total=0;
int dealer_total;
```

Everything is an `int` and the `betting` function doesn't check for positive numbers, which means we can just bet a negative number and lose. If we just hit S all the time, we are guaranteed to lose.
