# Guess My Cheese Part 2

`Category: Cryptography` | `Source: picoCTF` | `Difficulty: Medium`

> The imposter was able to fool us last time, so we've strengthened our defenses!
> Here's our list of cheeses.

## First look

The challenge gives a cheese list and a remote program.

```text
Abbaye du Mont des Cats
Abertam
Ackawi
Acorn
Allgauer Emmentaler
Anejo Enchilado
Anthoriro
Ardi Gasna
Asiago
Balaton
...
```



```bash
nc verbal-sleep.picoctf.net 52470
```

After connecting, the program gives something that looks very different from Part 1:

```text
DRAT! The evil Dr. Lacktoes Inn Tolerant's clone was able to guess the cheese last time! I guess simple ciphers aren't good hashing methods. But now I've strengthened my encryption scheme so that now ONLY SQUEEXY can guess it...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:
a1a67408515628538f770605b4084fd221e3ca5e3020461cc3124b1d1d7bfed3

Commands: (g)uess my cheese
```

There is no encryption option anymore and the value is also 64 hexadecimal characters long, which
made me think about a hash first. My first try was simply to hash each cheese from the list and
compare it with the value from the server.

```python
import hashlib

target = "a1a67408515628538f770605b4084fd221e3ca5e3020461cc3124b1d1d7bfed3"

with open("cheese_list.txt", "r", encoding="utf-8") as f:
    cheeses = [line.strip() for line in f if line.strip()]

for cheese in cheeses:
    if hashlib.sha256(cheese.encode()).hexdigest() == target:
        print(cheese)
```

Nothing came back, so the hash was not just the cheese name by itself.

## Going back to the hints

At this point I checked the hints. SHA-256 confirmed the hash idea, and the second hint mentions
exactly 2 nibbles of hexadecimal salt.

I had to check the wording here because one nibble is 4 bits, so two nibbles are 8 bits and written in
hexadecimal, that means a salt from:

```text
00 to ff
```

Only 256 possible salts and with a list of 599 cheeses, brute forcing this is very reasonable.

## First salted attempt

My first guess was that the salt was added as two normal hex characters, like `77`.
I also had to guess where the salt was placed. It could be before or after the cheese, so I tried
both simple formats first.

```python
import hashlib

target = "a1a67408515628538f770605b4084fd221e3ca5e3020461cc3124b1d1d7bfed3"

with open("cheese_list.txt", "r", encoding="utf-8") as f:
    cheeses = [line.strip() for line in f if line.strip()]

for cheese in cheeses:
    for i in range(256):
        salt = f"{i:02x}"

        for guess in (cheese + salt, salt + cheese):
            if hashlib.sha256(guess.encode()).hexdigest() == target:
                print(cheese, salt)
```

That still did not find anything. So the salt probably was not the text `"77"`, but the actual
single byte `0x77`. I also kept the same simple placement idea, the salt should be either before
or after the cheese, not hidden somewhere else unless the hints tell us.

## Brute forcing the hash

I adjusted the script to try a raw byte salt. I also tried simple case variants of each cheese
name, because a hash changes completely if even one letter changes case.

```python
import hashlib

target = "a1a67408515628538f770605b4084fd221e3ca5e3020461cc3124b1d1d7bfed3"

with open("cheese_list.txt", "r", encoding="utf-8") as f:
    cheeses = [line.strip() for line in f if line.strip()]

for cheese in cheeses:
    for name in (cheese, cheese.lower(), cheese.upper()):
        for i in range(256):
            salt = bytes([i])

            if hashlib.sha256(name.encode() + salt).hexdigest() == target:
                print("cheese:", name)
                print("salt:", f"{i:02x}")
```

This found:

```text
cheese: bleu d'auvergne
salt: 77
```

So the matching hash was made from:

```python
hashlib.sha256(b"bleu d'auvergne" + bytes([0x77])).hexdigest()
```

## Getting the flag

I went back to the program and chose:

```text
g
```

It asked for the cheese first:

```text
bleu d'auvergne
```

Then it asked for the salt:

```text
77
```

The program accepted both and printed the flag:

```text
YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{cHeEsY80eed518}
```

```text
picoCTF{cHeEsY80eed518}
```
