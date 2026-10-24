---
title: "4 - ToxicMiner"
date: 2026-10-24 18:03:00 +0300
categories: [CTF, Flare-On 13]
tags: [reverse-engineering, flare-on, ctf]
description: Write-up for the ToxicMiner challenge from Flare-On 13.
---
## My initial analysis

When I looked at the strings in the file, I thought it statically used the `OpenSSL` library. I also found many constants from different encryption algorithms. When I used the Findcrypt plugin, it showed many indicators of encryption algorithms, but the functions using these constants still needed to be renamed.

![](</assets/img/flare-on-13/Pasted image 20261003170255.png>)

So, I tried [IDA FLIRT](https://docs.hex-rays.com/core/flirt). The strings showed that the program uses [OpenSSL 1.0.1e (11 Feb 2013)](https://openssl-library.org/source/old/1.0.1/), so I downloaded it and created a signature file. I followed the instructions in this [article](https://itspazaz.com/2023-09-03-openssl-1.0-x64-flirt/), then went to **View → Subviews → Signatures** and added the signature. It identified only 67 functions.

![](</assets/img/flare-on-13/Pasted image 20261003171530.png>)

You can learn more about FLIRT in `The IDA Pro Book`, Chapter 12.

Then I thought that, since the binary had many functions similar to or implementing the same functionality as the `OpenSSL` library, I could write a Python script to compare the program with a compiled version of OpenSSL, such as a DLL.

I found a tool named [Diaphora](https://github.com/joxeankoret/diaphora/tree/master) and used it to compare the two files. It reported more than 35% similarity. The analysis had not finished yet, but the tool crashed.

I tried another approach. I went to the main function and found an important string: `"BTC donation address: 1FhDPLPpw18X4srecguG3MxJYe4a1JsZnd (tpruvot)\n\n"`.

I searched for this string and found that the program uses source code from [CPUMiner-Multi](https://github.com/tpruvot/cpuminer-multi). I compared our program with `cpu-miner.c` and found that they were very similar. Then I started analyzing our decompiled code alongside this source code.

![](</assets/img/flare-on-13/Pasted image 20261003173158.png>)

I also downloaded and compiled the project. Then I compared the important and interesting functions.

It is very similar to our program, and now I can name the functions.

![](</assets/img/flare-on-13/Pasted image 20261003182307.png>)

In the main function, our program sets some constants that force it to run in benchmark mode and use the algorithm where `opt_algo = 39`:

```
opt_benchmark = 1;
opt_algo = 39;
```

The function `miner_thread` at address `0000000140100920` is very similar to the source code. The important part for us is `switch ( opt_algo )` at address `00000001401012B0`. It jumps to the case for the selected algorithm. In our program, `opt_algo` is set to `39` in the main function.

In the source code, value `39` corresponds to `ALGO_SHA256D`. The program then calls `scanhash_sha256d` at address `0000000140134F70` in our binary.

This function contains an important detail that is not present in the other functions or the source code. It calls `GetVolumeInformationA`, which retrieves our machine's volume serial number and stores it in `VolumeSerialNumber`.

Then it checks whether the length of `rpc_user` is 64. We need to use the `--user` option and pass a 64-character string. This check is important because it sets `v9` to `1`.

![](</assets/img/flare-on-13/Pasted image 20261003214720.png>)

For example, I entered:

```
ToxicMiner.exe --user=11223344556677889900aabbccddeeff11223344556677889900aabbccddeeff
```

It stores every two hex digits as one byte.

![](</assets/img/flare-on-13/Pasted image 20261004105156.png>)

After this loop, `Buf2` contains the value of `rpc_user` as bytes.

![](</assets/img/flare-on-13/Pasted image 20261003230239.png>)

At address `0000000140135156`, this value is compared with `Buf1`.

![](</assets/img/flare-on-13/Pasted image 20261004115159.png>)

Then it runs the same function on `Buf1` and stores the result in `v29`.

It then compares this result with `Buf2`, which contains `rpc_user` as bytes.

So, we need to know two values to enter this loop: `rpc_user` and `VolumeSerialNumber`.

However, after `memcmp`, there is a block of code that does not depend on `rpc_user`. This is because `sub_140134CA0` does not depend on the value of `Buf2`; it overwrites `Buf2` and eventually returns the value stored there.

![](</assets/img/flare-on-13/Pasted image 20261004122456.png>)

After the `memcmp` instruction, the program seems to perform the operations needed to get the flag. `unk_1401BD3B0` contains encrypted data, which may be the encrypted flag. At the end, the program prints `Accepted: `.

![](</assets/img/flare-on-13/Pasted image 20261004121219.png>)

So, we need to brute-force `VolumeSerialNumber`. It is 4 bytes long, which means there are `2^32` possible values, from `0` to `0xFFFFFFFF`.

I heard about a technology that lets threads run on the GPU instead of the CPU, which could make our script much faster. It is called [CUDA](https://developer.nvidia.com/cuda/gpus). I'll try it later, but for now I'll use [OpenMP](https://www.openmp.org/). It runs on the CPU and supports cross-platform, shared-memory parallel programming in C, C++, and Fortran.

## Solver

When I first solved this challenge, I had trouble writing code that exactly matched `sub_140134CA0`, so I loaded the program itself and called this function directly.

**Script:**

```cpp
#include <windows.h>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <atomic>
#include <chrono>

uint8_t encrypted_flag[51] = {
    0xF4,0xF9,0x65,0x9F,0x31,0xF3,0xDA,0x5C,0xF1,0x37,0xA1,0xB8,0xCF,0x90,0xA1,0x58,
    0x1E,0xEB,0xE4,0x99,0xA7,0xC1,0x1B,0x15,0x02,0xAF,0x74,0x63,0xD3,0xB2,0x19,0x28,
    0x66,0xDE,0x3D,0xE9,0x71,0x63,0xE7,0x75,0xDF,0xE4,0xAB,0x61,0x6F,0xB9,0x0C,0x27,
    0x89,0xFC,0x87
};

using DeriveKey = uint64_t(__fastcall*)(uint32_t, uint32_t, uint8_t*);

std::atomic<bool> found{false};
uint32_t foundSerial;
char flag[52];

void rc4(uint8_t* key, const uint8_t* data, uint8_t* out, int len) {
    uint8_t s[256];
    for (int i = 0; i < 256; i++) s[i] = i;

    int j = 0;
    for (int i = 0; i < 256; i++) {
        j = (j + s[i] + key[i % 16]) & 255;
        uint8_t t = s[i];
        s[i] = s[j];
        s[j] = t;
    }

    int i = 0;
    j = 0;

    for (int k = 0; k < len; k++) {
        i = (i + 1) & 255;
        j = (j + s[i]) & 255;

        uint8_t t = s[i];
        s[i] = s[j];
        s[j] = t;

        out[k] = data[k] ^ s[(s[i] + s[j]) & 255];
    }
}

int main() {
    HMODULE mod = LoadLibraryExA(
        "ToxicMiner.exe", nullptr, DONT_RESOLVE_DLL_REFERENCES
    );

    if (!mod) {
        puts("Failed to load ToxicMiner.exe");
        return 1;
    }

    DWORD oldProtect;
    VirtualProtect(mod, 0x200000, PAGE_EXECUTE_READWRITE, &oldProtect);

    auto derive = reinterpret_cast<DeriveKey>(
        reinterpret_cast<uint8_t*>(mod) + 0x134CA0
    );

    auto start = std::chrono::steady_clock::now();

    #pragma omp parallel for schedule(dynamic, 65536)
    for (int64_t n = 0; n <= 0xFFFFFFFFLL; n++) {
        if (found.load(std::memory_order_relaxed)) continue;

        uint8_t key[32], plain[51];
        derive((uint32_t)n, 189800062, key);
        rc4(key, encrypted_flag, plain, 51);

        if (memcmp(plain + 38, "@flare-on.com", 13) == 0) {
            bool expected = false;

            if (found.compare_exchange_strong(expected, true)) {
                foundSerial = (uint32_t)n;
                memcpy(flag, plain, 51);
                flag[51] = '\0';
            }
        }
    }

    auto end = std::chrono::steady_clock::now();

    if (found) {
        printf("Serial: 0x%08X\n", foundSerial);
        printf("Flag: %s\n", flag);
    } else {
        puts("Flag not found");
    }

    printf("Time: %.2f sec\n",
        std::chrono::duration<double>(end - start).count());

    return 0;
}

```

**Compile and run:**
1. Open **x64 Native Tools Command Prompt for VS 2022**.
2. Compile the code:

```
cl /O2 /openmp solver.cpp
```

3. Run it:

   ```bat
   solver.exe
   ```

**flag:**

```
cpu_m3lt3d_4nd_all_i_g0t_w4s_th1s_fl4g@flare-on.com
```
