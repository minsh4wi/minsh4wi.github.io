---
title: "2 - GhostStream"
date: 2026-10-24 18:05:00 +0300
categories: [CTF, Flare-On 13]
tags: [reverse-engineering, flare-on, ctf]
description: Write-up for the GhostStream challenge from Flare-On 13.
---

We are given a file named `GhostStream.wim`, which uses the Windows Imaging Format (WIM).

## The mistake I made at first

The first thing I did was open it in HxD.

![](</assets/img/flare-on-13/Pasted image 20260929100556.png>)

I found the magic bytes `4D 5A` at offset `0xD0`, so I removed the bytes before them and extracted the EXE file!

After analyzing the program, I found that it uses a file named `PolyjuicePotion.txt`. I created an empty file with that name and added values to its ADS. From the analysis, I figured out that it uses strings related to the movie `Harry Potter`. I may have guessed the right values, but the spelling was wrong, so I spent a lot of time on this challenge (._.)

## WIM file format

![](</assets/img/flare-on-13/Pasted image 20260929105624.png>)

![](</assets/img/flare-on-13/Pasted image 20260929105852.png>)

We can use 7-Zip to extract the files. After extracting the contents, we get two files: `GhostStream.exe` and `PolyjuicePotion.txt`.

## Reversing the program

When analyzing the main function of `GhostStream.exe`, I found a pattern the program uses to import functions. It passes encoded data to the function at address `0000000140001130`. I named this function `loads_Processes`.

![](</assets/img/flare-on-13/Pasted image 20260929110834.png>)

After debugging, I found that `ProcName` changes to `OpenClipboard`. It is then passed to `GetProcAddress` and executed.

I traced this pattern in the other locations and gave those functions names, too.

When this function executes, copying and pasting no longer work, so it is better to NOP it.

![](</assets/img/flare-on-13/Pasted image 20260929111426.png>)

It is also better to NOP the sleep function at address `0000000140002130`.

I know these are simple things, but they may be useful for beginner REs.

At address `00000001400021E0`, we can see an anti-debugging technique.

![](</assets/img/flare-on-13/Pasted image 20260929112332.png>)

The value `0x11` stands for `ThreadHideFromDebugger`, so we should skip this call or NOP it while debugging. We can also use ScyllaHide.

> **Tip for beginner REs:** If you find that an address in x64dbg does not match the address in IDA, load the file into `PE-bear`, go to `Optional Header` > `DLL Characteristics`, and change the value to `8120`.

The function at address `0000000140001860` takes one argument: `lpThreadParameter`.

![](</assets/img/flare-on-13/Pasted image 20260929114155.png>)

When analyzing and debugging this function, I found that only one block is important to us.

![](</assets/img/flare-on-13/Pasted image 20260929114605.png>)

The value of `dword_140006668` is hardcoded to `0`. The challenge says, `you aren't getting your nutrient paste until your debugger forces a confession out of it.` So, we need to change its value to `1` or change `jne` to `je` to enter the `if` statement.

Now we know that the file has an alternate data stream (ADS).

![](</assets/img/flare-on-13/Pasted image 20260929120225.png>)

We can extract it using this command:

```cmd
more < PolyjuicePotion.txt:LordVoldemort
```

The value is `AlbusDumbledore`.

At `LABEL_39`, the program creates a pipe and connects to it.

![](</assets/img/flare-on-13/Pasted image 20260929121120.png>)

Then, at address `0000000140001400`, it loads a resource named `Partner_CTF.exe`. We can extract it using Resource Hacker, or find it in the temp directory after executing this function, because the program calls:

```cpp
GetTempPathW(0x104u, Buffer);
```

Finally, it sends the buffer to the other program.

![](</assets/img/flare-on-13/Pasted image 20260929122324.png>)

After execution, the program prints `[*] The Polyjuice Potion is still effective, preventing the true identity from being revealed.` This happens because it checks `Buffer[0]` against `A`. Our stream value is `AlbusDumbledore`, so the first character is `A`. If we change it to another character, this message will not appear. However, the message `[FAILED] Try again with better debugging`, which comes from `Partner_CTF.exe`, is still printed. Maybe we need to change the value to another character from the movie.

After more debugging and analysis, I found a function we had not analyzed yet: `sub_140001B20`. It contains a hint from the text in `PolyjuicePotion.txt`: `7 days`.

![](</assets/img/flare-on-13/Pasted image 20260929123412.png>)

While debugging, I found that `WindowLongPtrW + 0x18` equals `0x2760`, which is `10080` in decimal. This is the number of minutes in 7 days.

So, we need to move the instruction pointer past this block to skip it, then check whether the buffer changes to a word that does not start with `A`.

![](</assets/img/flare-on-13/Pasted image 20260929124256.png>)

When we run until the `WriteFile` function, the value of `Buffer` is `HarryPotter`.

![](</assets/img/flare-on-13/Pasted image 20260929135547.png>)

This may be the value the program expects. We can add it to the ADS with this command:

```cmd
echo HarryPotter> PolyjuicePotion.txt:LordVoldemort
```

However, when I execute the program, it still prints `[FAILED] Try again with better debugging`. Maybe there is something else we need to edit in the other file.

## Reversing the Partner_CTF program

When analyzing `Partner_CTF`, I found that it creates a pipe connection to the other program and uses `ReadFile` to read the value from the buffer. Since we know the buffer value is `HarryPotter`, we can pass that value in the buffer and set `NumberOfBytesRead` (the value pointed to by the fourth argument, `&NumberOfBytesRead`) to `0xB` to pass the debugger check.

![](</assets/img/flare-on-13/flare13_2-GhostStream.gif>)

In the following code, the program changes the memory permissions of `strlen` so that it can write shellcode to it, and then restores the permissions.

![](</assets/img/flare-on-13/Pasted image 20260929145850.png>)

The string `Str`, which contains ``Bxs]bv`^u)TpcnolVjN``, will be decoded into `Piertotum Locomotor`. However, `Str_Variable`, which contains `Wingardium Leviosa`, will be encoded.

We want the first string to be readable, but not the second one, so we only need to execute this function on the first string.

Further down, the return value of `strlen` for the second string is compared.

![](</assets/img/flare-on-13/Pasted image 20260929153823.png>)

So, it should return the actual length of `Wingardium Leviosa`, which is `0x12`.

Now, set a breakpoint at address `000000014000167B`, where the flag is checked.

![](</assets/img/flare-on-13/flare13_2-GhostStream-2 1.gif>)

![](</assets/img/flare-on-13/Pasted image 20260929152202.png>)

![](</assets/img/flare-on-13/Pasted image 20260929152346.png>)

**flag:**

```text
y0u_kn0w_n07h1n6_j0n_5n0w@flare-on.com
```
