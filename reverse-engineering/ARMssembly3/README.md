# ARMssembly3

`Category: Reverse Engineering` · `Source: picoCTF` · `Difficulty: Hard`

> What integer does this program print?
> Flag format: `picoCTF{XXXXXXXX}` (hex, lowercase, no 0x, and 32 bits).
> Use argument `a`: `4101707659`.

---

## First look

Same family as the earlier ARMssembly challenges, one ARM64 file. This time there are two
helpers, `func1` and `func2`, plus `main`. `main` runs `atoi` on `argv[1]`, passes it to `func1`,
and prints the return with `"Result: %ld\n"`. So everything is in `func1` (which itself calls
`func2`), and I need what it returns for `a = 4101707659`.

---

## Reading func2

```asm
func2:
	str	w0, [sp, 12]   ; save x
	ldr	w0, [sp, 12]
	add	w0, w0, 3      ; x + 3
	ret                    ; return x + 3
```

Nothing to it, `func2(x) = x + 3`.

---

## Reading func1

```asm
func1:
	str	w0, [x29, 28]  ; n   = arg
	str	wzr, [x29, 44] ; result = 0
	b	.L2
.L4:
	ldr	w0, [x29, 28]
	and	w0, w0, 1      ; n & 1  (is n odd?)
	cmp	w0, 0
	beq	.L3            ; if even, skip the add
	ldr	w0, [x29, 44]
	bl	func2          ; result = func2(result) = result + 3
	str	w0, [x29, 44]
.L3:
	ldr	w0, [x29, 28]
	lsr	w0, w0, 1      ; n = n >> 1  (drop the lowest bit)
	str	w0, [x29, 28]
.L2:
	ldr	w0, [x29, 28]
	cmp	w0, 0
	bne	.L4            ; loop while n != 0
	ldr	w0, [x29, 44]  ; return result
	ret
```

Two local variables: `n` starts at the argument, `result` starts at 0. Each pass:

- `.L4` looks at the lowest bit (`n & 1`). If `n` is odd it adds 3 to `result` through `func2`, if it's even it jumps to `.L3`.
- `.L3` shifts `n` right by 1, basically divide n by 2 and rounding it down.
- `.L2` compares `n` with 0, and loops back to `.L4` if that's the case.

So everytime n is odd, we add 3 to result and divide by 2 rounding down, everytime n is even, we just divide by 2.

That's also
`result = 3 * (number of 1-bits in n)`.

---

## Computing the result

```
a = 4101707659 = 0xF47B178B
  = 1111 0100 0111 1011 0001 0111 1000 1011
```

Counting the ones gives 19 set bits, so:

```
3 * 19 = 57 = 0x39
```

With the formatting rules:

```
picoCTF{00000039}
```
