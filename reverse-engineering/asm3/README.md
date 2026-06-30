# asm3

`Category: Reverse Engineering` | `Source: picoCTF` | `Difficulty: Hard`

> What does `asm3(0xdaba8a6c,0xfa8a55d0,0xd7771dae)` return?
> Submit the flag as a hexadecimal value starting with `0x`.

## First look

The source is very short, but it uses smaller pieces of registers that I have not often seen:

```asm
asm3:
	<+0>:	endbr32
	<+4>:	push   ebp
	<+5>:	mov    ebp,esp
	<+7>:	xor    eax,eax
	<+9>:	mov    ah,BYTE PTR [ebp+0x9]
	<+12>:	shl    ax,0x10
	<+16>:	sub    al,BYTE PTR [ebp+0xd]
	<+19>:	add    ah,BYTE PTR [ebp+0xf]
	<+22>:	xor    ax,WORD PTR [ebp+0x10]
	<+26>:	nop
	<+27>:	pop    ebp
	<+28>:	ret
```

The function receives three arguments:

```text
asm3(0xdaba8a6c, 0xfa8a55d0, 0xd7771dae)
```

The return value will be in `eax`, but the code mostly works on `ax`, `ah`, and `al`.

## Learning about the registers

`eax` is a 32 bit register. The lower half of it is called `ax`, and `ax` is split again into
two one byte registers:

```text
eax = 32 bits
ax  = lower 16 bits of eax
ah  = high byte of ax
al  = low byte of ax
```

In hexadecimal, one byte is written with two hex digits:

```text
ah = 0x8a   one byte
ax = 0x8a00 two bytes
```

The source also uses `BYTE PTR` and `WORD PTR`. In this context:

```text
BYTE = 1 byte
WORD = 2 bytes
```

Since this is 32 bit x86, the arguments are on the stack:

```text
[ebp+0x8]   first argument
[ebp+0xc]   second argument
[ebp+0x10]  third argument
```

x86 is little endian, so the bytes are stored in reverse order:

```text
0xdaba8a6c  becomes  6c 8a ba da
0xfa8a55d0  becomes  d0 55 8a fa
0xd7771dae  becomes  ae 1d 77 d7
```

So the useful stack values are:

```text
[ebp+0x9]       = 0x8a
[ebp+0xd]       = 0x55
[ebp+0xf]       = 0xfa
WORD [ebp+0x10] = 0x1dae
```

## Process

The first instruction clears `eax`:

```asm
xor eax,eax
```

So we start with:

```text
eax = 0x00000000
ax  = 0x0000
```

Then the byte at `[ebp+0x9]` goes into `ah`:

```asm
mov ah,BYTE PTR [ebp+0x9]
```

```text
ah = 0x8a
al = 0x00
ax = 0x8a00
```

Then `ax` is shifted left by `0x10`, which is 16. Since `ax` is only 16 bits wide, shifting it
left by 16 clears it:

```asm
shl ax,0x10
```

```text
ax = 0x0000
```

Now the code subtracts one byte from `al`:

```asm
sub al,BYTE PTR [ebp+0xd]
```

```text
al = 0x00 - 0x55 = 0xab
ax = 0x00ab
```

This wraps because `al` is only one byte. It cannot store a negative value, so the subtraction
stays inside one byte:

```text
0x00 - 0x55 = -0x55
0x100 - 0x55 = 0xab
```

Then it adds another byte into `ah`:

```asm
add ah,BYTE PTR [ebp+0xf]
```

```text
ah = 0x00 + 0xfa = 0xfa
ax = 0xfaab
```

The last operation is an xor with the first word of the third argument:

```asm
xor ax,WORD PTR [ebp+0x10]
```

```text
0xfaab ^ 0x1dae = 0xe705
```

## Flag

The challenge asks for the return value as a hexadecimal value starting with `0x`.

```text
0xe705
```
