# ARMssembly4

`Category: Reverse Engineering` · `Source: picoCTF` · `Difficulty: Hard`

> What integer does this program print?
> Flag format: `picoCTF{XXXXXXXX}` (hex, lowercase, no 0x, and 32 bits).
> Use argument `a`: `1854822502`.

---

## First look

Same family as the earlier ARMssembly challenges, one ARM64 file. `main` runs `atoi` on `argv[1]`,
passes it to `func1`, and prints the return with `"Result: %ld\n"`. This time, we have 8 functions that branch to eachother in different ways. So I
try building the call graph, then follow my one input `a` through it.
Everything is in `w` registers (32-bit), and `a = 1854822502` is below 2^31 so there's no sign or
overflow trouble.

---

## Reading the functions

**func1** is the entry point:

```asm
func1:
	cmp	w0, 100
	bls	.L2            ; n <= 100  -> func3(n)
	add	w0, w0, 100    ; else n > 100
	bl	func2          ; -> func2(n + 100)
.L2:
	bl	func3          ; -> func3(n)
```

`func1(n) = if n > 100: func2(n + 100) else func3(n)`.

**func2**:

```asm
func2:
	cmp	w0, 499
	bhi	.L5            ; n > 499  -> func5(n + 13)
	sub	w0, w0, #86    ; else n <= 499
	bl	func4          ; -> func4(n - 86)
.L5:
	add	w0, w0, 13
	bl	func5          ; -> func5(n + 13)
```

`func2(n) = if n > 499: func5(n + 13) else func4(n - 86)`.

**func3** just forwards to `func7`:

```asm
func3:
	bl	func7          ; return func7(n)
```

**func4** is a trap:

```asm
func4:
	str	w0, [x29, 28]  ; save n
	mov	w0, 17
	bl	func1          ; func1(17), result saved...
	str	w0, [x29, 44]
	ldr	w0, [x29, 28]  ; ...then thrown away, reload n
	ret                    ; return n
```

It calls `func1(17)` but it never uses the result, then returns the original `n`. So `func4(n) = n`.
After following the `func1(17)` call, it resolves to `func7(17) = 7` and gets discarded.

**func5** forwards to `func8`, and **func8** adds 2:

```asm
func5:
	bl	func8          ; return func8(n)
func8:
	add	w0, w0, 2      ; return n + 2
```

So `func5(n) = n + 2`.

**func7** returns `n` if it's big, else 7:

```asm
func7:
	cmp	w0, 100
	bls	.L18           ; n <= 100 -> return 7
	...return n
.L18:
	mov	w0, 7
```

`func7(n) = if n > 100: n else 7`.

**func6** is never called by anyone (`main` only calls `func1`, and nothing calls `func6`).

---

## The whole program

Putting everything together, with a big input like `a` every comparison takes the "bigger" branch, so the
path is just `func1 -> func2 -> func5 -> func8`, and `func3`/`func4`/`func7` are never reached. The
net effect is adding `100 + 13 + 2 = 115`.

---

## Computing the result

```
func1(1854822502) : 1854822502 > 100  -> func2(1854822502 + 100) = func2(1854822602)
func2(1854822602) : 1854822602 > 499  -> func5(1854822602 + 13)  = func5(1854822615)
func5(1854822615) = func8(1854822615)  = 1854822615 + 2           = 1854822617
```

So the result is `a + 115`:

```
1854822502 + 115 = 1854822617 = 0x6E8E58D9
```

With the formatting rules:

```
picoCTF{6e8e58d9}
```
