# asm4

`Category: Reverse Engineering` | `Source: picoCTF` | `Difficulty: Hard`

> What will `asm4("picoCTF_574ff")` return?
> Submit the flag as a hexadecimal value starting with `0x`.

## First look

This one looks longer than the previous `asm` challenges, but the input is a string, so
`[ebp+0x8]` should be read as the address of that string (it is also given to us by the hint). Whenever the code loads `[ebp+0x8]`,
adds an index to it, and then reads `BYTE PTR [eax]`, it is reading one character from the
string.

```asm
mov    edx,DWORD PTR [ebp-0xc]
mov    eax,DWORD PTR [ebp+0x8]
add    eax,edx
movzx  eax,BYTE PTR [eax]
```

This is basically:

```c
s[index]
```

## Finding the length

The first loop starts with `[ebp-0xc] = 0`, then keeps checking characters until it reaches a
null byte:

```asm
mov    DWORD PTR [ebp-0xc],0x0
...
add    DWORD PTR [ebp-0xc],0x1
...
movzx  eax,BYTE PTR [eax]
test   al,al
jne    0x11c8 <asm4+27>
```

So `[ebp-0xc]` is the string length. For the input:

```text
picoCTF_574ff
```

the length is:

```text
13
```

## Process

After that, the function sets:

```asm
mov    DWORD PTR [ebp-0x10],0x240
mov    DWORD PTR [ebp-0x8],0x1
```

I treated them as:

```c
acc = 0x240;
i = 1;
```

The loop runs while:

```asm
i < length - 1
```

So for a string of length `13`, it goes from `i = 1` to `i = 11`.

Inside the loop, the function first computes:

```c
s[i] - s[i - 1]
```

Then it computes:

```c
s[i + 1] - s[i]
```

Both values are added to the accumulator. So one loop is the same as:

```c
acc += (s[i] - s[i - 1]) + (s[i + 1] - s[i]);
```

The middle `s[i]` cancels out:

```c
acc += s[i + 1] - s[i - 1];
```

At first I expanded a few iterations by hand, then noticed the pattern. Across the whole loop,
almost everything in the middle cancels, and only the first two and last two characters matter.

The input is:

```text
picoCTF_574ff
```

The first two characters are:

```text
p = 112
i = 105
```

The last two characters are:

```text
f = 102
f = 102
```

So the final value is:

```text
0x240 + 102 + 102 - 112 - 105
= 576 + 204 - 217
= 563
= 0x233
```

## Flag

The challenge asks for the hexadecimal return value:

```text
0x233
```
