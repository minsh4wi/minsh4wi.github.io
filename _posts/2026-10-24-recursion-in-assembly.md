---
title: "Recursive Functions in Assembly (x64dbg)"
date: 2026-10-24 17:05:00 +0300
categories: [Reverse Engineering, Data Structures]
tags: [reverse-engineering, assembly, data-structures]
description: "A practical look at recursive functions, stack frames, and return values using x64dbg."
---

I wrote two simple C programs, compiled them, and traced their execution in x64dbg to understand how recursive functions work at the assembly level.

## Recursive Functions Without a Return Value (`void`)

```c
#include <stdio.h>

void func(int n)
{
    if (n == 0)
        return;

    printf("%d\n", n);
    func(n - 1);
}

int main(void)
{
    func(3);
    return 0;
}
```

The function prints the current value of `n`, decrements it by one, and recursively calls itself until `n` becomes zero. Since the function's return type is `void`, it does not return a meaningful value to its caller.

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708161138.png)

The `main` function calls `func` and passes it the value `3`.

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708161521.png)

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708162257.png)

After the `call` instruction, the caller does not use the value in `eax`. Although `eax` may contain a value left by previous instructions or function calls, the caller ignores it because a `void` function does not return a meaningful value.

When a function calls another function, the CPU pushes the return address onto the stack and the callee establishes its own stack frame when needed.

The same behavior occurs when a function calls itself recursively. Every recursive call has its own execution context and return address on the stack.

Once the base case is reached, the recursive calls begin to return. Each `ret` instruction takes the corresponding return address from the stack and transfers execution to the instruction immediately following the `call` in the previous stack frame. This process continues until all recursive calls have returned.

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708164415.png)

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708173037.png)

## Recursive Functions With a Return Value

```c
#include <stdio.h>

int func(int n)
{
    if (n == 0)
        return 1;

    return n * func(n - 1);
}

int main(void)
{
    printf("%d\n", func(3));
    return 0;
}
```

The function recursively calls itself until it reaches the base case (`n == 0`), where it returns `1`. The result is then used by each caller to calculate the final value.

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708175013.png)

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708180151.png)

Once the base case is reached, the recursive calls stop and the program begins to unwind the call stack. Each `ret` instruction returns execution to the instruction immediately following the corresponding `call`.

In this example, execution returns to address `0x00007FF613F218CE`. At this point, the caller uses the value returned by the previous recursive call to calculate its own result. This process repeats until all recursive calls have returned.

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708180621.png)

On Windows x64, an integer return value is placed in `eax`/`rax`. The caller uses that value to perform the multiplication before returning its own result to the previous caller.

![](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260708181433.png)
