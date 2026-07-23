
---

title: "The same code printed 10 on Linux and 0 on a Mac"
date: 2026-07-20
draft: false
summary: "This post's about a Dray compiler bug regarding how variadic arguments work on different platforms."
tags: ["open-source", "c", "raylib", "compiler", "rust", "dray"]
---

## Context and the problem

I've been working on a programming language called [Dray](https://github.com/bichanna/dray). It's reference counted, got no garbage collector, and the backend emits C which then gets handed to whatever C compiler you have lying around. Compiling to C is a lovely cheat. But you also inherit sharp edges C has...

Last week I got arrays and slices working and finally wanted a demo program that actually *printed* something. Up until then all my demo programs encoded the answer in the process exit code, like some kind of a caveman. Dray could call C functions by then, so this looked like a five minute job.

```
printf :: extern "printf" proc(fmt: *int8, value: int32) -> int32;

main :: proc() -> int32 {
    nums: [4]int32 = { 10, 20, 4, 3 };
    for n, [i] in nums {
        printf("nums[%d] = ", i);
        printf("%d\n", n);
    }
    return 0;
}
```

I compiled it on my Linux pc, ran it, got the numbers, committed it, went to bed pleased with myself.

```
nums[0] = 10
nums[1] = 20
nums[2] = 4
nums[3] = 3
```

Then the day after, I ran it on my mac.

```
nums[0] = 0
nums[0] = 0
nums[0] = 0
nums[0] = 0
```

Every value's zero, including the loop index, which was supposed to be counting. Not a crash, not garbage bytes, just zeros, which is the worst possible failure mode because zero looks like an answer.

## Wrong guesses

My first instinct was that the new `for n, [i] in nums` syntax was broken. New code is usually guilty until proven innocent, so I dumped the generated C, expecting something embarrassing. But the for loop was completely unremarkable:

```c
for (int32_t __dray_idx_1 = 0; __dray_idx_1 < 4; __dray_idx_1 += 1) {
  int32_t n = nums[__dray_idx_1];
  printf("nums[%d] = ", __dray_idx_1);
  printf("%d\n", n);
}
```

Then I assumed the array literal was zeroing itself. It wasn't. Then the slice `len` field, because I'd just changed how those get emitted. Was not it, either.

## Culprit

The thing that cracked it was hand-editing that generated C file to `#include <stdio.h>` and delete the `extern` declaration my compiler emitted. It worked. Which means the bug wasn't in any of the code I'd been staring at. It was in the one line I'd never looked at thrice.

Dray was emitting this:

```c
extern int32_t printf(int8_t *fmt, int32_t value);
```

The real one is `int printf(const char *restrict, ...)`.

Mine says two arguments and then nothing. It wasn't a mistake in the sense of a typo. It was the only thing my compiler could produce because Dray's `extern` syntax had no way to say "and then more arguments, however many the caller feels like."

## Two different answers to the same question

Every platform decides separately how variadic arguments get passed, and the popular answers don't agree.

x86-64 SysV passes integer arguments in rdi, rsi, rdx, rcx, r8, r9 whether or not the function is variadic. A variadic callee spills those registers into a save area on entry and `va_arg` reads through it. So my caller put the value in rsi because that's where a fixed second argument goes, and `printf` went looking in a buffer that gets populated from rsi.

```
  x86-64 SysV

  caller (thinks: fixed)              callee (is: variadic)

    rdi  <-  "%d\n"        ------->     reads rdi
    rsi  <-  42            ------->     spills rsi into save area
                                        va_arg reads save area  ->  42
```

Nothing goes wrong because the two conventions overlap here luckily. But that is not the same as being correct.

Apple's arm64 diverges a bit. Apple intentionally [deviates from AAPCS64](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms#Update-code-that-passes-arguments-to-variadic-functions) for variadic functions specifically, and puts the variadic arguments on the stack. Fixed arguments still go in x0 through x7. The two conventions have no overlap at all, so a caller compiled against the wrong prototype misses entirely:

```
  Apple arm64

  caller (thinks: fixed)              callee (is: variadic)

    x0   <-  "%d\n"        ------->     reads x0                ->  ok
    x1   <-  42                 X       reads [sp]
                                        (x1 never looked at)
    stack:   untouched                                          ->  0
```

`printf` behaved perfectly. It read its second argument from the place its own calling convention says the second argument lives, found whatever was on the stack, formatted it as `%d`, and printed it.

It turned out I wasn't the first one to make this mistake. Rust's Cranelift backend essentially had the same [shortcut](https://github.com/rust-lang/rustc_codegen_cranelift/issues/1451), declaring imported variadic functions as fixed-arity because varargs weren't implemented yet, and it notes the trick holds on essentially every target except arm64 macOS. So this is a known hack with a known exception, and I reinvented both halves without knowing either.

## `gcc` told me

The whole time, on the machine where everything worked, `gcc` was printing:

```
warning: conflicting types for built-in function 'printf';
expected 'int(const char *, ...)' [-Wbuiltin-declaration-mismatch]
```

I read it. I decided it was about `int8_t*` versus `const char*`, and thought "same bytes, don't matter." And I moved on.

The pointer spelling really is cosmetic. That's what made it a good cover, but the same warning was also telling me I'd declared a variadic function as non-variadic, which is a disagreement about where the arguments physically are, and gcc has no way to fix that on my behalf.

When you generate code, the compiler you generate for is the only thing in the pipeline that reads your output carefully. Ignoring its warnings is like ignoring your one reviewer.

## The fix

Dray `extern` declarations take an ellipsis now:

```
printf :: extern "printf" proc(fmt: *int8, ...) -> int32;
```

This emits `extern int32_t printf(int8_t *fmt, ...);` and the two sides agree again. Call checking treats a variadic as "at least these arguments", so extras are allowed and leaving off the format string is still an error.

That C-style **heterogeneous** variadics is only allowed for `extern`, though. A standard Dray function only allows **homogeneous** variadics, like this: `[...]string`.

Anyway, the Dray compiler's hosted [here](https://github.com/bichanna/dray), if you're interested.
