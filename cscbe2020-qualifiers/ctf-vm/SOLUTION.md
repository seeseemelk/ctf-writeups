# Solution to CTF-VM
Let's first try to run CTF-VM and see what the output is.

```
0:seeseemelk@MobileDalek (master)$ wine CTF-VM.exe
Usage: CTF-WM <byteCodeFile> <vm-args>
Created by Zat: github.com/molatho
```

We see that we need to pass two arguments: a bytecode file and vm arguments.
Let's see what happens when we pass comrade.lel as the bytecode file.

```
0:seeseemelk@MobileDalek (master)$ wine CTF-VM.exe comrade.lel
 > Comrade Kharchenko, you need to confirm your identity by providing the code word.
```

We see that we need to provide a code word. Let's add something extra on the command line.

```
1:seeseemelk@MobileDalek (master)$ wine CTF-VM.exe comrade.lel foobar
 > Comrade Gagarin, that was not very convincing!
```

Apparently, we gave the incorrect password.
If we pass something else as the bytecode file, we get the following output:

```
0:seeseemelk@MobileDalek (master)$ wine CTF-VM.exe CTF-VM.exe
VM-Error: 1 at 1
Stack: [ ]
```

This output teaches us that the bytecode file like contains real bytecode for
some kind of virtual machine.

# Reversing the bytecode
There are two ways we could use to reverse the instruction set of this VM:
by reversing the VM, or by reversing the bytecode file.

I started of reversing the VM using radare2 and cutter, but didn't really any anywhere.
Next up I tried opening the bytecode file using a hex editor.

[images/bless-1.png]

If we assume that instruction execution starts at the very first byte, we see a
pattern emerge. If you don't see the pattern immediately, try group every five
bytes together, like this:

```
5E 20 00 00 00   ^ ...
5E 3E 00 00 00   ^>...
5E 20 00 00 00   ^ ...
5E 43 00 00 00   ^C...
5E 6F 00 00 00   ^o...
5E 6D 00 00 00   ^m...
5E 72 00 00 00   ^r...
5E 61 00 00 00   ^a...
5E 64 00 00 00   ^d...
5E 65 00 00 00   ^e...
5E 20 00 00 00   ^ ...
5B 67 07 00 00
```

Here we see that each line starts with `5E`, followed by an ASCII character,
followed by three null bytes.
The first byte is likely an opcode, a byte specifying what the virtual machine
should actually do.
This is later confirmed by the last line, which contains `5B 67 07 00 00`.
The four bytes after the opcode are most likely a 4-byte argument in little-endian.

So now that we know how an instruction is formed, we can start to reverse the
instruction set of this virtual machine.
We already know one:

```
5E XX -- -- --    Print ASCII character XX
```

The next opcode we encounter is `5B`. So what does this do?
If we assume that the argument is little endian, the argument would be equal
to `0x00000767`.
If we go to this address in the file, it goes exactly to the start of this
instruction: `5B 6B 08 00 00`.
If we go to address `0x0000086B`, we again see the start of instruction `47 03 00 00 00`.
As this argument of this instruction always point to the start of a new instruction,
we can relatively safely assume that this instruction is a jump instruction.
We can validate this by changing the very beginning of the file to this:

```
5E 43 00 00 00 (Print 'C')
5B 00 00 00 00 (Goto address 0)
```

When we run this bytecode file, this is our output:
```
0:seeseemelk@MobileDalek$ wine CTF-VM.exe comrade.lel
CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
```

So now we already now two instructions:
```
5E XX -- -- --    Print ASCII character XX
5B 78 56 34 12    Jump to address 0x123456789
```

With these two instructions we can easily try out new instructions by modifying
our bytecode file.
While trying several combinations to try and discover the purpose of opcode `5C`,
I accidentally wrote `5B C0 00 00 00`. When executed, I promptly got the following
output:
```
0:seeseemelk@MobileDalek$ wine CTF-VM.exe comrade.lel2 codeword
 elcome back. We trust you will keep our secret code safe: CSC{160799pr0bl3m5bu7b1n4ryr3v3r533n61n33r1n641n70n3}
```

While I had expected to VM to either crash or give some random output, this I
did not expect. Entering this code worked, and I did not attempt to further
reverse engineer the instruction set as I had other challenges to complete.
