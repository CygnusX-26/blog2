---
layout: post
title: "Swimming to Escape - csctf2024"
date: 2024-08-30 16:00:00
categories: writeups
tags: misc esolang
excerpt: Teach the fish how to escape
mathjax: false
---
* content
{:toc}
496 points - 11 solves

**Author**: aa.crypto

### Challenge Description 
Teach the fish how to escape

We're given a python server that looks like this:
```python
from fish import Interpreter, StopExecution

FLAG = open('flag.txt').read().strip()
whitelist = r' +-*,%><^v~:&!?=()01\'/|_#l$@r{};"'

if __name__ == '__main__':

    while True:
        print('Please input your code:')
        code = input()
        assert len(code) <= 26
        assert all([c in whitelist for c in code])

        code = code + 'n;;;\n' + ';' * 30
        interpreter = Interpreter(code)
        interpreter._stack = [ord(c) for c in FLAG]

        count = 0
        while True:
            try:
                instr = interpreter.move()
                print(instr)
                print(interpreter._stack)
                count += 1
            except:
                break

            if count >= 5000:
                print('Too many moves!!')
                break
        print()
```

And we are given a parser for the [Fish Esolang](http://esolangs.org/wiki/Fish)

Fish is a [stack based esolang](https://esolangs.org/wiki/Stack)

We can see from the provided server, that the flag is placed one char at a time onto the stack.

The server also appends `n;;;\n` followed by 30 `;`s to whatever we submit.

Also there are a list of whitelisted characters we can use, and we have a payload limit of 26 characters.

My first thoughts are to do some research into how the Fish esolang works.

I created a list of some important characters that we are allowed to use.

```
> - make instruction pointer move forwards
< - make instruction pointer move backwards
! - trampoline (skip following instruction)
? - conditional trampoline (pop a value off the stack and execute the next instruction if the popped value is non-zero)
0 - push zero to the stack
1 - push one to the stack
+ - pop top two values and add together
- - pop top two values and sub 
= - pop top two values and push 1 if equal and zero otherwise
( - pop top two values if second value is greater than first push 1, zero otherwise
) - same as greater than, but less than
" and ' - read in until a closing quote, push each char onto the stack that is read
: - duplicate top stack value
~ - remove top value on the stack
$ - swap top two values on the stack
@ - swap top three values on the stack
r - reverse the stack
l - push length of stack onto the stack
& - push or pop value of *the* register
; - end execution
n - pop top of stack and output char as a number (we dont have access to this one)
```

Ok so `n;;;\n` followed by 30 `;` is appended to our input. This prints whatever is on the top of the stack and ends execution.

The reason for the 30 `;`s on the next line is because fish is a 2d language and the instruction pointer can move up, down, left and right. These `;` end the program which effectively prevents us from moving down.

My first thoughts is that this challenge is really easy! Just print each char of the flag off the stack one at a time.

![initial_thoughts](../../../../images/csctf2024/initial.png)

But we find that 

![initial_thoughts](../../../../images/csctf2024/first_fail.png)

We reached the character limit, meaning the flag is longer than 26 characters.

Then I remembered that we can use the `l` function to print out the size of the stack.

![initial_thoughts](../../../../images/csctf2024/stacksize.png)

Ok the flag is 140 characters long, which means this method defintely wont work, even if we reverse the stack we can only get a maximum of 51 characters.

My next idea was to write some kind of for loop that would extract the nth char down on the stack, and we can just print out each character of the flag one at a time.

What made this really tricky was that they banned the `.` instruction which lets us jump to previous instructions, and also the `[]` instructions which let us pop many values off the stack at a time.

We still have the `>` and `<` instructions which tell the program to move forwards and backwards, but the problem now is we need to write some code that is a for loop and works forwards and backwards.

We can also note that there is a `!` instruction which skips the next instruction and the `space` instruction which is a [NOP](https://en.wikipedia.org/wiki/NOP_(code)).

Using these two instructions we can create a "expensive" but simple way to make moving backwards easier.

`!  ‎` (! followed by a space) allows us to skip the NOP moving forwards, and skip whatever instruction before the `!` moving backwards. This way, moving forwards these two instructions do nothing, but moving backwards we can skip some instructions which might mess up our code execution.

Now lets write some pseudocode to jump to the nth character on the stack.

```
PUSH n
MOVE FORWARD
PUSH 1
SKIP NEXT
NOP
SUBTRACT THE TOP TWO ON THE STACK
SKIP NEXT
NOP
SWAP THE TOP TWO STACK VALUES
SKIP NEXT
NOP
POP TOP OF STACK
DUPLICATE TOP OF STACK
DUPLICATE TOP OF STACK
SKIP NEXT
NOP
SKIP NEXT IF ZERO IS ONTOP OF STACK
MOVE BACKWARDS
POP TOP OF STACK
POP TOP OF STACK
```

The pseudocode pushes our number onto the stack, subtracts one from it, and checks if the result is non zero. If it is non zero, it goes back upwards, skipping any instructions that would break the ordering of elements on the stack, and we repeat again.

If the value reaches zero, it exits the loop and pops off some remaining stack data.

Lets convert this to `Fish` code. For example to get the 33rd element on the stack. ("!" has [ascii value 33](https://www.cs.cmu.edu/~pattis/15-1XX/common/handouts/ascii.html))

```
"!">1! -! $! ~::! ?<~~
```

Now we just have to find a way to get all numbers between zero and 139 using only the characters we have in our whitelist. Surprisingly after writing a script we can get every single number except 0 and 115 (which we can figure out manually later).

The script below prints out combos of chars that generate numbers

```python
s = r' +-*,%><^v~:&!?=()01\'/|_#l$@r{};'
l2 = []
for i in s:
    l2.append(i)
l2.sort()
s = l2
l = ["-1"] * 140

for i in range(len(s)):
    for j in range(i):
        if l[ord(s[i]) - ord(s[j])] == "-1":
            l[ord(s[i]) - ord(s[j])] = s[i] + s[j] + "-"
for i in range(len(s)):
    for j in range(len(s)):
        if ord(s[i]) + ord(s[j]) >= 140:
            break
        if l[ord(s[i]) + ord(s[j])] == "-1":
            l[ord(s[i]) + ord(s[j])] = s[i] + s[j] + "+"
for i in range(len(s)):
    if l[ord(s[i])] == "-1":
        l[ord(s[i])] = s[i]
```

You can view a list of the generated values in the form `num: <char><char><operator>` [here](../../../static/csctf2024/values.txt)

Now all we have to do is append these values to the front of our fish script, and we can print out every character of the flag in reverse except one.

The solve script prints out `BRUH` instead of the missing character.

```python
from pwn import *

conn = remote("swimming-to-escape.challs.csc.tf", 1337)
# context.log_level = 'debug'
s = r' +-*,%><^v~:&!?=()01\'/|_#l$@r{};'
l2 = []
for i in s:
    l2.append(i)
l2.sort()
s = l2
l = ["-1"] * 140

for i in range(len(s)):
    for j in range(i):
        if l[ord(s[i]) - ord(s[j])] == "-1":
            l[ord(s[i]) - ord(s[j])] = s[i] + s[j] + "-"
for i in range(len(s)):
    for j in range(len(s)):
        if ord(s[i]) + ord(s[j]) >= 140:
            break
        if l[ord(s[i]) + ord(s[j])] == "-1":
            l[ord(s[i]) + ord(s[j])] = s[i] + s[j] + "+"
for i in range(len(s)):
    if l[ord(s[i])] == "-1":
        l[ord(s[i])] = s[i]
        
for i in range(len(l)):
    if len(l[i]) == 2:
        l[i] = None
        continue
    if len(l[i]) == 3:
        if '"' in l[i]:
            l[i] = f"'{l[i][0]}{l[i][1]}'{l[i][2]}"
        else:
            l[i] = f'"{l[i][0]}{l[i][1]}"{l[i][2]}'
        continue
    elif len(l[i]) == 1:
        l[i] = f'"{l[i][0]}"'

for i in range(len(l)):
    if l[i] is not None:
        conn.sendlineafter(b'Please input your code:\n', f'''{l[i]}>1! -! $! ~::! ?<~~'''.encode())
        print(chr(int(conn.recvline().strip().decode())), end='')
    else:
        print("BRUH", end="")



conn.interactive()
```

The script prints out

`CSCTF{do_y0u_l1k3_f15h??BRUH_h3r3_c0m3S_r4Nd0M_cH4rs_a0893a103c0c4cb6595a25703de54f69e4615e99637b450cddbd1cb9332bd400_F15h_Sw1m5_4nd_35cAp3s!!}`

It seems obvious that character is a `?`
so now we get the flag :)!

CSCTF{do_y0u_l1k3_f15h???_h3r3_c0m3S_r4Nd0M_cH4rs_a0893a103c0c4cb6595a25703de54f69e4615e99637b450cddbd1cb9332bd400_F15h_Sw1m5_4nd_35cAp3s!!}

### My thoughts

This was a super fun challenge props to the author `aa.crypto`. It was my first time actually writing code with an esolang and this challenge took a lot of effort to solve X) (there were many failed solve scripts/strategies before the one in this writeup). The 26 character limit was definitely the most challenging part. Thanks for reading see you next time ^_^




