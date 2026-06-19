# ARMssembly0

`Category: Reverse Engineering` · `Source: picoCTF` · `Difficulty: Medium`

> What integer does this program print?
> Flag format: `picoCTF{XXXXXXXX}` (hex, lowercase, no 0x, and 32 bits).
> Use arguments `a` and `b`: `2593949075` and `2233560849`.

---

## First look

The challenge gives a single ARM64 assembly file, `chall.S`. The task is to figure out what the program prints for the two
arguments without executing it. So this is a pure reading exercise: follow the registers and
the stack, and reproduce by hand what the CPU would do.

The program has two functions, `main` and `func1`. `main` is pretty simple: it reads the two
command line arguments, converts them to integers, calls `func1`, and prints the return value.
Everything interesting happens in `func1`.

---

## Reading main

```asm
ldr	x0, [x29, 32]      ; argv
add	x0, x0, 8          ; argv + 1  -> argv[1]
ldr	x0, [x0]
bl	atoi               ; atoi(argv[1])
mov	w19, w0            ; w19 = a

ldr	x0, [x29, 32]
add	x0, x0, 16         ; argv + 2  -> argv[2]
ldr	x0, [x0]
bl	atoi               ; atoi(argv[2])
mov	w1, w0             ; w1 = b

mov	w0, w19            ; w0 = a
bl	func1              ; func1(a, b)
```

So the first argument goes into `w0` and the second into `w1`, in that order. With the values
from the instructions that is `func1(2593949075, 2233560849)`, and `main` just prints whatever it
returns with the `"Result: %ld\n"` format string in  `.LC0`.

---

## Reading func1

```asm
func1:
	sub	sp, sp, #16
	str	w0, [sp, 12]   ; save a
	str	w1, [sp, 8]    ; save b
	ldr	w1, [sp, 12]   ; w1 = a
	ldr	w0, [sp, 8]    ; w0 = b
	cmp	w1, w0         ; compare a, b
	bls	.L2            ; if a <= b (unsigned), go to .L2
	ldr	w0, [sp, 12]   ; else w0 = a
	b	.L3
.L2:
	ldr	w0, [sp, 8]    ; w0 = b
.L3:
	add	sp, sp, 16
	ret
```

It compares the two arguments and returns the bigger one. The branch is `bls`, "branch if lower
or same", so the comparison is unsigned.
If `a <= b` it returns `b`, otherwise it returns `a`.
Both arguments are above 2^31, so a signed read would make them negative and flip the result. 

---

## Computing the result

```
a = 2593949075
b = 2233560849
a > b, so func1 returns a = 2593949075
```

All that is left is to write `2593949075` as a 32-bit hexadecimal value. It fits in 32 bits, and
in hex it is `0x9A9C8593`. The flag wants it lowercase with no prefix, so the answer is the eight
digits `9a9c8593`.

```
picoCTF{9a9c8593}
```
