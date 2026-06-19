# ARMssembly2

`Category: Reverse Engineering` · `Source: picoCTF` · `Difficulty: Hard`

> What integer does this program print?
> Flag format: `picoCTF{XXXXXXXX}` (hex, lowercase, no 0x, and 32 bits).
> Use argument `a`: `2401941830`.

---

## First look

It's the same family as ARMssembly0 and 1, one ARM64 source, here there's only one argument. `main` runs `atoi` on
`argv[1]`, passes it to `func1`, and prints the return value with `"Result: %ld\n"`. So again
everything is in `func1`, and I need the integer it produces for `a = 2401941830`.

---

## Reading func1

```asm
func1:
	str	w0, [sp, 12]   ; arg
	str	wzr, [sp, 24]  ; acc = 0
	str	wzr, [sp, 28]  ; i   = 0
	b	.L2
.L3:
	ldr	w0, [sp, 24]
	add	w0, w0, 3      ; acc += 3
	str	w0, [sp, 24]
	ldr	w0, [sp, 28]
	add	w0, w0, 1      ; i += 1
	str	w0, [sp, 28]
.L2:
	ldr	w1, [sp, 28]   ; i
	ldr	w0, [sp, 12]   ; arg
	cmp	w1, w0
	bcc	.L3            ; loop while i < arg (unsigned)
	ldr	w0, [sp, 24]   ; return acc
	ret
```

We can notice that there is a loop, two slots in the stack start at zero: a counter `i` that goes up by 1 each pass, and an
accumulator `acc` that goes up by 3. The condition `bcc` ("branch if carry clear", same instruction as blo) keeps looping while `i < arg`, so the body runs `arg` times and the accumulator
ends at `3 * arg`.
But `a * 3` would be `7205825490`, which needs 33 bits to be stored, and everything happens in `w` registers, which are documented to be 32 bits wide. So the result
is `3 * arg` with the top bit dropped.

---

## Getting the flag

```
3 * 2401941830 = 7205825490
```

That is larger than 2^32 (4294967296), so it wraps:

```
7205825490 - 4294967296 = 2910858194  =  0xAD802BD2
```

Lowercase with no prefix:

```
picoCTF{ad802bd2}
```
