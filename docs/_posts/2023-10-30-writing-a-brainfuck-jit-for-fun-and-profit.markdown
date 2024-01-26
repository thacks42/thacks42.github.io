---
layout: post
title:  "Writing a Brainfuck JIT for fun and profit. Part 1"
---

## Did you know that X is Turing complete???

When first learning about programming languages one thing that comes up a lot is complexity: No matter if you learn python or C++, there are so many details to discover,
so many different ways in which a goal can be accomplished, so many rules and exceptions to them, but also a huge amount of potential to solve whichever problem you might stumble across.

After you spend some time programming, there comes a point at which you might look at a problem and realize "I know how to solve this with a program",
which invites us to ask a deceptively subtle question: "Are all programming languages created equal in their abilities to solve problems?"
Or in other words: are there a certain classes of problems which some languages can solve and others can't? And if such classes exist, is there a "superclass" which encompasses all of them?
As it turns out, there is! We call the ability of formal systems (such as programming languages) to solve all programmatically solvable problems "Turing completeness".

Turing completeness is something people tend to find in the [weirdest places](https://en.wikipedia.org/wiki/Turing_completeness#Unintentional_Turing_completeness), which is not too surprising
once you consider how few requirements a system has to fulfill to get there. While the [formal definitions](https://en.wikipedia.org/wiki/Turing_completeness#Formal_definitions) can be somewhat laborious,
an informal definition, such as "a turing complete system can compute anything any other computing system can compute (given enough time)" is usually sufficient.

In other words: The fact that Magic: The Gathering is turing complete means that, given enough time (and probably also table space) you can "run" a "program" in it to calculate the 42nd digit of pi 
(Or whatever else it is you want to compute). This usually implies that you have to follow some specific rules as to how to "input" data into your "machine" and how to interpret the board state as "output",
but the rule-set itself is sufficient to perform your computations.

The part people usually tend to gloss over is the "given sufficient time", after all this is not for practical applications, it's just for the fun of giving theoretical CS people something to talk about
on parties, or allow them to give "interesting" talks at IT conferences.

## Inventing languages no one needs

And this is usually also the mental framework from which to view minimally turing complete languages like Brainfuck: Their main contribution to society is memes
("hey, you should rewrite your ray-tracer in brainfuck"), pedantry ("technically, you *could* rewrite your ray-tracer in brainfuck") or weird hobby projects for people with questionable hobbies (such as this one).
However it can also be an interesting example to show just how seemingly limited a system might appear and still be fully Turing complete.
The language itself is easy enough to explain. It consists of exactly eight symbols: `+` `-` `<` `>` `[` `]` `,` `.` and the environment that runs the code consist of a large array of byte sized[^1] "cells",
and a pointer that points to the "current" cell. The eight symbols then have the following behavior assigned to them:
```
+   increment the value of the current cell
-   decrement the value of the current cell
<   move the pointer to the left (i.e. to the previous cell)
>   move the pointer to the right (i.e. to the next cell)
[   if the current cell is zero, find the matching ] and continue execution after it
]   if the current cell is non-zero, find the matchin [ and continue execution after it
,   read one character from the terminal input and write it into the current cell
.   output the value of the current cell interpreted as an ascii character to the terminal
```
All characters other than those eight are considered comments and can show up all over the program without any special escaping.

To get an idea for how brainfuck programs tend to look like, we can consider a few examples:

```
print "!"

+++++++++++++++++++++++++++++++++       increment the current cell 33 times
.                                       and output its value: ASCII 33 is an exclamation mark so this will print "!"
```
```
set the current cell to zero:

[           if the current cell is not zero
    -       decrement the current cell
]           repeat until the current cell is zero.
```
```
add two cells:

[           if the current cell is not zero:
    -       decrement the current cell
    >       move to the next cell
    +       increment the cell
    <       move back to the first cell
]           repeat until the current cell is zero

overall this has the effect of adding the value of the current cell to the next cell and setting the current cell to zero (destructive add)
```

It becomes immediately clear that any such program performing a non-trivial task (such as finding prime numbers or even just printing short sections of text) will end up with hundreds or even thousands
of symbols. And due to the fact that there is almost no inherent structure other than loops, reading brainfuck code is closer to reading tea-leaves,
than it is to comprehending any kind of conventional source code. This is only worsened by the fact that quite a lot of brainfuck programs out there tend to be presented in
"[block form](https://raw.githubusercontent.com/frerich/brainfuck/master/samples/mandelbrot.bf)" without any comments, indentation or other visual help.

## Writing an interpreter

On the other hand writing a brainfuck interpreter is quite simple, even in C or C++ it only takes a few lines. This comes down to the simplicity of parsing the input:
we can just process one character at a time and the only side information we have to track is the location of open-braces \[ which requires no more than a stack.
So all we need is a big array, a pointer, a stack and a big switch statement:

```cpp
void bf_interpreter(const std::string& bf_code){
    std::vector<uint8_t> memory(30000); // the array of cells
    uint8_t* ptr = &memory[0]; // pointer to the first cell
    
    std::stack<size_t> open_braces;
    
    for(size_t i = 0; i < bf_code.size(); i++;){
        switch(bf_code[i]){
            case '+':
                *ptr++;
                break;
            case '-':
                *ptr--;
                break;
            case '>':
                ptr++;
                break;
            case '<':
                ptr--;
                break;
            ...
        }
    }
}
```

## Why not write a compiler... in assembly?!

In fact doing this is simple enough that it can reasonably written in even less pleasant languages, which is also where the initial inspiration for this project came from:
I was asked for some input on how to make a brainfuck interpreter written in x86_64 assembly run faster (apparently that's what they teach kinds in school these days, I'm jealous).
Given that they were already writing assembly code, I suggested a (maybe just-in-time) compiler as opposed to an interpreter.
The idea being that you first iterate over the whole bf source code once and for each symbol you generate a small number of assembly instructions, usually no more than one or two, 
and once you're finished you jump to the first instruction emitted and execute them as a "normal" program. This eliminates the overhead of running the interpreter loop, 
which means that you have far fewer jumps, comparisons etc [^2]. 
This is another great advantage of dealing with such a simple language: even though it makes the program size explode and writing code in it a huge pain, implementation of its behavior is so simple,
that for most of the symbols there is a one-to-one correspondence to assembly instructions. For this example we will assume that the `rsi` register holds our pointer:

```x86asm
inc rsi             ; increments the pointer, equivalent to `>`
dec rsi             ; decrements the pointer, equivalent to `<`
inc byte [rsi]      ; increments the value pointed at by the pointer, equivalent to `+`
dec byte [rsi]      ; decrements the value pointed at by the pointer, equivalent to `-`
```

Loops require slightly more work:

```x86asm
;code for [
cmp byte[rsi], 0    ; compare the current cell to zero
je  closing_brace   ; if they are equal, jump to the matching closing brace
```

```x86asm
;code for ]
cmp byte[rsi], 0    ;compare the current cell to zero
jne open_brace      ;if they are not equal, jump to the matching open brace
```

For input and output we can just make a call to the C library functions `getchar` and `putchar` (or our own implementation of them):

```x86asm
movsx edi, byte[rsi]    ; copy and zero extend the value pointed at by the pointer into the "edi" register
call putchar            ; call putchar, which expects the input to be in the edi register
```
```x86asm
call getchar            ; call getchar, returns the character value in the `ax` register
mov [rsi], ax           ; copy the value into the current cell
```

The keen-eyed reader may already have noticed that the `closing_brace` and `open_brace` labels are a bit hand-wavy as they have to jump to the *matching* brace.
In fact this whole approach of emitting assembly is not really a functional approach if we don't want to resort to external help, because to convert from assembly to machine code
we'd still have to run an assembler. But since the amount of machine code instructions we need to produce boils down to a pretty small set, we can actually just emit those directly.
To do so we just need to know the encodings of all the instructions we need, which can easily done by running them all through an assembler and looking at the output using objdump:

```x86asm
   0:    48 ff c6                 inc    rsi
   3:    48 ff ce                 dec    rsi
   6:    fe 06                    inc    byte [rsi]
   8:    fe 0e                    dec    byte [rsi]
   a:    80 3e 00                 cmp    byte [rsi],0x0
   d:    74 00                    je     <some_location>
   f:    75 00                    jne    <some_location>
  11:    0f be 3e                 movsx  edi, byte [rsi]
  14:    e8 ff ff ff ff           call   <some_location>
  19:    66 89 06                 mov    byte [rsi],ax
```

So if we want to emit the instruction that increments our pointer, we just write `0x48 0xff 0xc6` (`inc rsi`) and we're good. For the jumps generated for the `[` and `]` we need to keep track of the 
location of the forward jumps, which we can do using the stack, and then whenever we find the matching backwards jump, we have to calculate the relative difference
(jumps in x86 usually take relative offsets) and use that for our backwards jump offset. We then also have to go back to the forward jump and patch in the relative offset in there as well.
The calls to putchar and getchar also need a relative address. Once we take care of those, we're done with our first [compiler](https://gist.github.com/thacks42/3514902a28e2af376320b14c0dac8dd4).


[^1]: the actual size of the array varies from implementation to implementation with 30000 cells being the value given in the language specification. Variations also include changing the sizes of the cells to more than 8 bits.
[^2]: one important thing to be aware of here is that programs usually spend most of their time in loops, and while an interpreter has to do all of its work for each iteration of the loop, a compiler only has to parse the loop once, and emit assembly instructions for it, and never deal with it again.
