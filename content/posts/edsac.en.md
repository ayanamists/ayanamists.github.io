---
title: Programming in 1951 -- The Case of EDSAC
date: 2022-05-21
tags:
  - EDSAC
  - Bootstrapping
categories:
  - Computer History
math: true
---

{{<  admonition "warning" >}}
This article was translated from Chinese by ChatGPT and may contain some errors.
{{< /admonition >}}

## EDSAC

EDSAC was a computer designed and built by the Cambridge University Mathematical Laboratory. In 1949, the project team successfully used EDSAC to generate and print a table of squares[^1], symbolizing that EDSAC was officially put into use.

![x^2](https://pic.imgdb.cn/item/6263ebe6239250f7c5fbde45.jpg)

The basic computing model of EDSAC is very similar to modern computers, which can be summarized as:

1. It has 32 tanks, each tank stores 32 words, note that here the word is 18 bits, not 16 bits on the x86 platform. These 32 tanks are similar to the memory of modern computers (the following text refers to these 32 tanks as "memory").
2. It has 5 registers, two of which (instruction register, instruction address register) are used for control, and three are used for calculation.
3. Computation is represented as sequentially executing orders, each order occupies 1 word, one bit of which is not used, so each order occupies 17 bits. All instructions and data are placed in the aforementioned memory (in fact, in the design of EDSAC, there is no difference between instructions and data). Each instruction may read or write registers or memory.
4. EDSAC's input device is a five-hole punch tape, which also serves the function of inputting programs (what this means will be explained later). EDSAC's output device is a telegraph printer. It is also equipped with a telephone, which can be used for "human-computer interaction". The world's first electronic game implemented on EDSAC -- Tic Tac Toe (disputed) is interacted through the telephone.

## EDSAC's Program

As mentioned earlier, in the design of EDSAC, "computation" is "sequentially executing instructions", so where are these instructions stored? Like today's computers, these instructions are stored in "memory". But where does the program in "memory" come from?

The answer is, the program in "memory" is read in through punch tape. The punch tape used by EDSAC is a "five-hole tape", which looks like this:

![1](https://pic.imgdb.cn/item/62888b4109475431290607f6.jpg)

Each unit of the tape has 5 holes, if punched, it represents 1, if not punched, it represents 0. So theoretically, we can regard the tape as a list, each element of the list is a natural number from 0 to 31.

The tape does not represent machine code (18-bit instructions), but the direct encoding of "text programs". In the EDSAC era, programmers first needed to write the text form of the program on paper, and then punch the text form of the program on the tape through a keypunch.

So, what does the text form of the program look like? This program is similar to today's assembly language program, a program that outputs "HI" is as follows:

```text {linenos=table}
T64K
GK
ZF
O5θ
O6θ
O7θ
ZF
*F
HF
IF
EZ
PF
```

What is this mess? Indeed, this form of program is still very difficult to read. Let's take a look bit by bit.

First of all, each line of this program represents one of three situations:

1. Instruction
2. Pseudo instruction
3. Data

Pseudo instructions are similar to the pseudo instructions in today's assembly language, they are **to provide necessary information for Initial Orders**. Initialization instructions (initial orders) are the focus of our discussion below, and can be temporarily understood as a kind of assembler.

Let's annotate the program above:

```
T64K        Pseudo instruction, meaning to load the current program into the 64th word (the 0th word of the 2nd box)
GK          Pseudo instruction, set the relocation parameter (θ parameter) to the current position (next line, here is the position of ZF)
--------------------------------------------------------------
ZF          Halt                                              }            
O5θ         Set the printer's offset mode to "Letter shift"     }     
O6θ         Let the printer print the letter represented by the highest 5 bits at position 6 (H)             }
O7θ         Let the printer print the letter represented by the highest 5 bits at position 6 (I)             }      These lines are actually loaded into memory
ZF          Halt                                              }
*F          Data of the O5θ instruction index                    }
HF          Data of the O6θ instruction index                    }
IF          Data of the O7θ instruction index                    }   
--------------------------------------------------------------
EZ          Pseudo instruction, the function of using EZ and PF at the same time is to tell the initialization instruction to set the instruction to be executed at the position of the θ parameter
PF          Same as above
```

As you can see, to understand this program, we must understand the relationship between the tape and EDSAC, and what the Initial Orders are for.

## Initial Orders

Initialization orders are the most core piece of code in the EDSAC system, a program, from today's perspective, it can be said to be the operating system of EDSAC, a relocatable assembler and loader. There are two versions of the Initial Orders, called Initial Orders 1 and Initial Orders 2. Since the text form program introduced above is read by the Initial Orders, the text form programs read by different Initial Orders are also different.

In order to understand the Initial Orders, it is best to observe what the program on the tape is like from the perspective of the Initial Orders. First of all, the program on the tape is a direct translation of the text form program, and each unit on the tape corresponds to a letter or number. For example, the line `T64K` will produce four corresponding units on the tape. In this way, the program that the Initial Orders get is the encoding of the text form program on the tape. The Initial Orders need to translate it into 18-bit instructions and load it into memory.

The Initial Orders read the numbers on the tape with instructions starting with `I`. For example:

```text {linenos=table}
I2S
```

It means to read the data at the head of the tape to the lowest 5 bits of position 2. (The meaning of `S` will be explained below)

In this way, we will immediately have a question, there are 26 English letters, plus 10 numbers, and 4 Greek letters (θ ϕ Δ π), to give each of these symbols a unique code, at least 40 numbers are needed, won't the five-point tape (up to 32 numbers) run out?

In fact, when the puncher punches, P and 0, Q and 1, W and 2, E and 3, all the way to O and 9, all punch the same hole:

![2](https://pic.imgdb.cn/item/62889b4a094754312911aa46.jpg)

Won't this cause a problem? From the perspective of Initial Orders, how do I distinguish whether the number I read is P or 0?

The answer is that the initialization instruction does not need to distinguish. Each line of the text form program has a specific format, which can be represented by formal grammar:

$$
\mathsf{Order} \rightarrow \mathsf{Pre}\ \mathsf{N}^*\ \mathsf{Post}
$$

Where:

+ $\mathsf{Pre}$ represents the type of instruction, for example, `I` is read from the tape, and `O` is output from the printer.
+ $\mathsf{N}^*$ represents multiple (can be none, if none, this part is 0) 0-9 integers, representing the address.
+ $\mathsf{Post}$ is the suffix of each instruction. In the design of initialization instruction 1, it is used to distinguish between short instructions and long instructions, `S` is a short instruction, and `L` is a long instruction; in the design of Initial Orders 2, `F` represents a short instruction, `D` represents a long instruction, and `θ` represents that the address of this instruction should be added with the θ parameter.

These three parts actually correspond to the division of the 18 bits of the real machine code:

![3](https://pic.imgdb.cn/item/6288a103094754312915a9f4.jpg)

After a little thought, we will find that as long as we ensure that $\mathsf{Post}$ and $\mathsf{N}$ do not have the same encoding, there will be no ambiguity. And this is very easy to satisfy.

## Function call?

The computer in 1951 already has the ability to call functions, which is undoubtedly very surprising. What is slightly comforting is that this kind of function call cannot be nested.

How is such a complex feature as function calling implemented on EDSAC? In fact, programs using initialization instruction 1 cannot perform function calls, and the appearance of Initial Orders 2 is mainly to solve this problem.

What magic did Initial Orders 2 use? Let's recall the θ parameter mentioned earlier. If an instruction ends with θ, then its address should be added with the current value of the θ parameter when translating. This mechanism, called "relocation" technology today, gives each program relative independence. Each program can refer to **relative addresses** instead of absolute addresses, so that function calls can be implemented simply by copying.

Specifically, EDSAC has a set of standard libraries. Whenever a programmer wants to use a certain library function, he will note it in the text form of the program. When the operator operating the puncher punches to this position, he will use the tape copier to copy the library function to the corresponding position of this tape[^2]:

![4](https://pic.imgdb.cn/item/6288a515094754312918d654.jpg)

## Bootstrapping?

> There are three things in the world that are hard to believe: first, wisdom cannot accomplish everything; second, strength cannot lift everything; third, power cannot conquer everything. Therefore, even if one possesses the wisdom of Yao but lacks the help of the masses, great achievements cannot be accomplished; if one has the strength of Wu Huo but does not have the help of others, one cannot lift oneself; if one has the power of Ben and Yu but lacks tactics, one cannot win indefinitely. Therefore, there are situations that cannot be achieved, and tasks that cannot be completed. Wu Huo feels light when lifting a thousand jun, but heavy when lifting himself. This is not because he is heavier than a thousand jun, but because the situation is not convenient. Li Zhu finds it easy to travel a hundred steps, but difficult to see the eyebrows and eyelashes. This is not because a hundred steps are near and the eyebrows and eyelashes are far, but because the path is not possible. Therefore, a wise ruler does not exhaust Wu Huo because he cannot lift himself, nor does he trap Li Zhu because he cannot see himself. By taking advantage of possible situations and seeking easy paths, one can use less effort and establish achievements and reputation. There are times of fullness and emptiness, things have pros and cons, objects have life and death, if the ruler of the people shows joy and anger for these three, then the gentleman of gold and stone will leave the heart. The simplicity of the sages is profound. The ancient wise rulers observed others, but did not let others observe them. Understanding that Yao cannot achieve alone, **Wu Huo cannot lift himself**, Ben and Yu cannot win by themselves, and using tactics to observe the path of action is complete.

Wu Huo is a strong man, and a strong man can lift many things, but can he lift himself?

Intuitively, of course not. Formalized, we can say that the "lifting" relationship is anti-reflexive.

The "self-bootstrapping" of a compiler comes from this allusion. Suppose the compiler A of language L is written in language L, then compiling A with A is "self-bootstrapping".

The initialization instruction also has a "bootstrapping" problem. Without the initialization instruction, the program on the tape cannot be executed at all, but so far, the only method we have introduced for program input is from tape input. Without the initialization instruction, you can't read the tape; if you can't read the tape, you can't input the initialization instruction into EDSAC.

How to solve this problem? The paper[^3] mentions:

> These orders are known as the initial orders and are permanently wired on a set of uniselectors. When the starting button is pressed, these orders are automatically transferred to the store and they then cause the input tape to be read.

It can be seen that the initialization instruction is directly written into the memory of EDSAC through a device called uniselectors. This uniselectors is also called a stepping switch. Due to the author's lack of electronic knowledge, a complete introduction cannot be given temporarily.

## A small question

Having read this far, the reader may wonder how many lines, or how many instructions, such a great Initial Orders 2 will be. Try to estimate it yourself, then look up the relevant information, you may be very surprised.


[^1]: Currently located in the Science Museum in London, most of the pictures in this article are from the [official documentation](https://www.dcs.warwick.ac.uk/~edsac/Software/EdsacTG.pdf) of the EDSAC simulator.

[^2]: [This video](https://www.bilibili.com/video
