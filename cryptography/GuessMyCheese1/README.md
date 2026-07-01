# Guess My Cheese Part 1

`Category: Cryptography` | `Source: picoCTF` | `Difficulty: Medium`

> Try to decrypt the secret cheese password to prove you're not the imposter!

## First look

The challenge gives a remote program.

```bash
nc verbal-sleep.picoctf.net 63503
```

After connecting, it prints an encrypted cheese and gives two options.

```text
The super evil Dr. Lacktoes Inn Tolerant told me he kidnapped my best friend, Squeexy, and replaced him with an evil clone! You look JUST LIKE SQUEEXY, but I'm not sure if you're him or THE CLONE. I've devised a plan to find out if YOU'RE the REAL SQUEEXY! If you're Squeexy, I'll give you the key to the cloning room so you can maul the imposter...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:  AZYYVCGK
Hint: The cheeses are top secret and limited edition, so they might look different from cheeses you're used to!
Commands: (g)uess my cheese or (e)ncrypt a cheese
```

The hint mentions linear equations, so I started with the idea that the encryption might be an
affine cipher. That is a substitution cipher where each letter is turned into a number from `0`
to `25`, then encrypted with:

```text
E(x) = a*x + b mod 26
```

At this point it was only a hypothesis, I still needed a known plaintext and its ciphertext to
check it.

## Testing the encryption option

The program lets us encrypt a cheese, so I tried to use it as an oracle. It does not accept every
random string, when I tried a single letter it rejected it. So I used an actual cheese
name:

```text
CHEDDAR
```

In my run, the program returned:

```text
CHEDDAR -> LCXEEZK
```

Now I had a plaintext and a ciphertext of the same length. That is enough to test the affine
cipher idea.

## Recovering the key

For every letter, I use the usual alphabet index.

```text
A = 0, B = 1, C = 2, ..., Z = 25
```

The affine cipher uses two unknown values, `a` and `b`.

```text
cipher = (a * plain + b) mod 26
```

Instead of solving the equations by hand, I wrote a small script that tries every valid `a` and
every possible `b`, then keeps the pair that correctly encrypts `CHEDDAR` into `LCXEEZK`.

```python
import math

alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
plain = "CHEDDAR"
cipher = "LCXEEZK"

for a in range(26):
    if math.gcd(a, 26) != 1:
        continue

    for b in range(26):
        test = ""

        for ch in plain:
            x = alphabet.index(ch)
            test += alphabet[(a * x + b) % 26]

        if test == cipher:
            print(a, b)
```

The script gave:

```text
a = 19
b = 25
```

The `gcd(a, 26) == 1` check matters because decryption needs the modular inverse of `a`. If `a`
shares a divisor with `26`, there is no clean way to undo the multiplication, so I skip it.

## Decrypting the secret cheese

To decrypt, I reverse the formula.

```text
plain = a^-1 * (cipher - b) mod 26
```

Since the first script found `a = 19`, I computed its inverse once:

```text
19 * 11 = 209
209 mod 26 = 1
```

So `inv_a = 11`, and the decryption script is:

```python
alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
cipher = "AZYYVCGK"

inv_a = 11
b = 25

plain = ""

for ch in cipher:
    x = alphabet.index(ch)
    plain += alphabet[(inv_a * (x - b)) % 26]

print(plain)
```

The loop applies the reversed formula to each encrypted letter.

The decrypted cheese was:

```text
LAPPIHZR
```

It does not look like a normal cheese name, but the program already warned that the cheeses were
limited edition and could look strange.

## Getting the flag

Once I had the decrypted cheese, I went back to the menu and chose:

```text
g
```

The program then asked me to guess the cheese. I submitted:

```text
LAPPIHZR
```

That value is not the flag itself, it is only the password the program expects. Once it accepted
the cheese, it printed the flag:

```text
YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{ChEeSydd15526f}
```

```text
picoCTF{ChEeSydd15526f}
```
