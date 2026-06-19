# ARMssembly1

`Category: Reverse Engineering` · `Source: picoCTF` · `Difficulty: Medium`

> For what argument does this program print "win"?
> Flag format: `picoCTF{XXXXXXXX}` (hex, lowercase, no 0x, and 32 bits).
> Variables: `a = 86`, `b = 3`, `c = 3`.

---

## First look

Same kind of file as ARMssembly0: one ARM64 source, one command line argument. The
difference is the question. `main` prints either `"You win!"` or `"You Lose :("`, and I need
the argument that makes it win instead of the value it prints.

---

## Reading main

```asm
bl	atoi               ; arg = atoi(argv[1])
str	w0, [x29, 44]
ldr	w0, [x29, 44]
bl	func               ; func(arg)
cmp	w0, 0
bne	.L4                ; if func(arg) != 0, jump to the "You Lose" branch
```

`main` converts `argv[1]` with `atoi`, passes it to `func`, and compares the return value with
zero. `bne` takes the losing branch whenever the result is not zero, so I win when `func`
returns exactly `0`.

---

## Reading func

```asm
func:
	str	w0, [sp, 12]   ; arg
	mov	w0, 86
	str	w0, [sp, 16]   ; a = 86
	mov	w0, 3
	str	w0, [sp, 20]   ; b = 3
	mov	w0, 3
	str	w0, [sp, 24]   ; c = 3
	ldr	w0, [sp, 20]   ; w0 = b
	ldr	w1, [sp, 16]   ; w1 = a
	lsl	w0, w1, w0     ; a << b
	str	w0, [sp, 28]
	ldr	w1, [sp, 28]
	ldr	w0, [sp, 24]   ; w0 = c
	sdiv	w0, w1, w0     ; (a << b) / c
	str	w0, [sp, 28]
	ldr	w1, [sp, 28]
	ldr	w0, [sp, 12]   ; arg
	sub	w0, w1, w0     ; (a << b) / c - arg
	str	w0, [sp, 28]
	ldr	w0, [sp, 28]
	ret
```

`func` loads the three constants and the argument, then runs three operations in a row:

- `lsl w0, w1, w0` is `a << b`, a left shift, so `86 << 3 = 688`
- `sdiv` is signed integer division: `688 / 3 = 229` (truncated, remainder 1 is dropped)
- `sub` does `229 - arg`

So `func(arg) = 229 - arg`.

---

## Getting the flag

`func` returns `0` when `229 - arg == 0`, so the winning argument is `229`. Written as a 32-bit
hex value that is `0x000000e5`, and the flag wants it lowercase with no prefix:

```
picoCTF{000000e5}
```

> Small note: the source file picoCTF handed me had different placeholder constants (`88`, `4`,
> `3`), while the variables given with my instance were `86`, `3`, `3`. The variables are the
> ones that decide the flag, so I retraced with those and lined the file up to match.
