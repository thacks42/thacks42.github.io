---
layout: post
title:  "Writing a Brainfuck JIT for fun and profit. Part 2"
---

## **Real** compilers optimize

Just like you're not a *real* programmer unless you used a [teletype](https://en.wikipedia.org/wiki/Computer_terminal)
to write [common lisp](https://en.wikipedia.org/wiki/Common_Lisp), you're not a *real* compiler unless you optimize or so I've heard.
After all with brainfuck there are some really low hanging fruit: it doesn't take much to look at a sequence like
```
++++++++++++
```
and wonder: instead of compiling this to twelve increment instructions, why not just use an addition? So this is exactly what we will do.
There are four common patterns we want to look for and optimize: 

* Multiple pointer increments into a single pointer addition
* Multiple pointer decrements into a single pointer subtraction
* Multiple cell increments into a single cell addition
* Multiple cell decrements into a single cell subtraction


These are usually also some of the most visually recognizable patterns in [large blocks](https://raw.githubusercontent.com/frerich/brainfuck/master/samples/mandelbrot.bf) of brainfuck code.
The result may then look something like this:

```x86asm
parse_pointer_increment:
    mov r8d, 0                  ; set the our character counter to zero
    
    .folding_loop:              ; fold multiple `>` into one addition
        inc r8                  ; increment the counter
        mov dl, byte [rsi + r8] ; grab the next character in the input stream
        cmp dl, '>'             ; check for another `>`
        je .folding_loop        ; if there is another one, repeat
    
    cmp r8d, 1                  ; check how many `>` we found
    jne .add_multiple           ; more than one -> use add instead of increment
    
    mov byte [rdi],     0x48    ; single `>`, just emit a `inc rsi` (0x48 0xff 0xc6)
    mov byte [rdi + 1], 0xff    
    mov byte [rdi + 2], 0xc6
    add rdi, 3                  ; and increment our ouput pointer by the number of bytes we wrote
    jmp parse_loop              ; continue parsing
    
.add_multiple:
    mov byte [rdi],     0x48    ; found multiple, so we emit a `add rsi, immediate` instruction (0x48 0x83 0xc6 XX)
    mov byte [rdi + 1], 0x83
    mov byte [rdi + 2], 0xc6
    mov byte [rdi + 3], r8b     ; emit the immediate
    add rdi, 4                  ; increment the output pointer by the number of bytes we wrote
    lea rsi, [rsi + r8 - 1]     ; increment the source pointer by the number of repeated chars we read
    jmp parse_loop              ; continue parsing
```

So whenever we find a `>` we "peek" ahead in the input stream to count how many `>` characters follow. If there are none, we bail out of our optimization attempt and do what we did before.
Otherwise we collect them into a single pointer addition.

We can then perform similar optimizations for all of `<`, `+` and `-`.
But there's another somewhat common pattern in brainfuck code: `[-]`, which decrements a cell until it is zero.
This should also be easy enough to optimize into a single write to that cell:

```x86asm
parse_open_brace:
    cmp byte [rsi + 1], '-'     ; check if the next symbol is a `-`
    jne .normal_brace           ; if not, just parse normally
    cmp byte [rsi + 2], ']'     ; if it was, check if the one after that is a `]`
    jne .normal_brace           ; if not, just parse normally
    
    mov byte [rdi],     0xc6    ; we're dealing with a `[-]`, so just emit a `mov byte [rsi], 0`  (0xc6, 0x06, 0x00)
    mov byte [rdi + 1], 0x06
    mov byte [rdi + 2], 0x00
    add rdi, 3                  ; increment the ouput pointer by 3
    add rsi, 2                  ; increment the source pointer by the number of characters we read
    jmp parse_loop              ; continue parsing
    
    .normal_brace:              ; handle normal loop
        ...                     ; like before...
        jmp parse_loop          ; continue parsing
```

## Are we gcc yet?

Now this was easy enough, so what's next? How about addition of two neighboring cells? As discussed in the previous post `[->+<]` will add the value of the current cell to the next one,
and leave the current cell with a value of zero. This *sounds* like a great place to optimize but what about something like `[->>+<<]`? I.e. an addition not to the next cell but the one after that?
Or `[->>>+<<<]`?
In general loops are where programs tend to spend most of their time, which makes them an important target for optimization. And loops that don't need to be loops,
because the actions they perform (like addition of one cell to another) could be done by one or two instruction instead, almost scream to be optimized.

If we keep writing our optimizations the way we have so far, by looking at one or two additional symbols in the input stream,
then each of these possible patterns will require its own section of code to recognize and replace. And if we manage to write some generic code for all patterns of this kind,
then some slightly different pattern like `[->+>+<<]` (addition of one cell to two other cells) will take the same amount of *almost* duplicate code.
While I guess duplicated code is par for the course when writing assembly, there is a more fundamental issue: we're still writing a single-pass compiler.

Trying to do *all* optimizations at once requires far more code than performing one optimization at a time and then feeding the result of that optimization pass into another optimization pass and so on.
That way we could first compress all the increments and decrements into a single addition, feed the result of that into something to optimize cell-to-cell additions, etc.
However this would require us to either invent additional symbols, e.g. for pointer-constant subtraction or cell-cell addition, or we invent a more useful intermediate representation
similar to what "proper" compilers do. If done correctly this will allow us to write simple optimization passes which mesh seamlessly and produce quite surprising results.

So starting over it is. And since we're starting over anyway now might be a good point to switch to a slightly less verbose language. X86 assembly is fun and all,
but I faintly remember compiler people going on about "parse trees" and dynamically allocating tree structures in assembly and then traversing them sounds like I'd be spending my next few weeks
staring at core dumps in gdb, so C++ it is. (Don't worry there will still be more than enough segfaults)

