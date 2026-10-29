---
title: "Linked Lists in Assembly with x64dbg"
date: 2026-10-24 17:06:00 +0300
categories: [Reverse Engineering, Data Structures]
tags: [reverse-engineering, assembly, data-structures, x64dbg]
description: "Notes and examples for recognizing and analyzing linked lists in assembly with x64dbg."
---

## Analyzing a Single Node

```c
#include <stdio.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

int main(void) {
    Node node;

    node.value = 10;
    node.next = NULL;

    return 0;
}
```

The assembly does not necessarily make the structure obvious. The compiler simply writes values to memory, much as it would for ordinary local variables.

One clue is that the second member may appear at `[rbp+10h]` instead of `[rbp+0Ch]`. The compiler inserts four bytes of padding after the `int` member so the 8-byte pointer is aligned on an 8-byte boundary. These offsets depend on how the compiler lays out the local variable in the stack frame.

However, this alone is **not enough** to conclude that the object is a linked list—or even a structure. From the assembly alone, we may only be able to infer that a 4-byte value is followed by an 8-byte value.

> **Note:** A linked list is usually identified from a broader pattern: pointer dereferences, traversal logic, and termination when a pointer becomes `NULL`—not from one instruction or one object.
{: .prompt-info }

![Single node in x64dbg](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260709195835.png)

When suitable debugging symbols or type information (such as a PDB) are available, IDA may be able to recover the structure definition and display member names and offsets.

![Node structure in IDA](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260709202918.png)

![Node structure details in IDA](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260709203056.png)

## Analyzing Multiple Nodes

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

int main(void) {
    Node *head = malloc(sizeof(Node));
    Node *second = malloc(sizeof(Node));
    Node *third = malloc(sizeof(Node));

    if (head == NULL || second == NULL || third == NULL) {
        free(head);
        free(second);
        free(third);
        return 1;
    }

    head->value = 10;
    head->next = second;

    second->value = 20;
    second->next = third;

    third->value = 30;
    third->next = NULL;

    Node *cur = head;

    while (cur != NULL) {
        printf("%d\n", cur->value);
        cur = cur->next;
    }

    free(head);
    free(second);
    free(third);

    return 0;
}
```

![Multiple nodes in the debugger](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260709205207.png)

![Traversing the linked list](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260709224052.png)

![Linked-list traversal in assembly](/assets/img/reverse-engineering/data-structures/Pasted%20image%2020260709224637.png)

## Conclusion

Typical patterns that may indicate linked-list traversal include:

- Reading a node's first field, for example, `[reg]`.
- Reading a pointer field, for example, `[reg+8]` on a typical x64 layout where the pointer follows a 4-byte integer and 4 bytes of padding.
- Repeatedly dereferencing the next-node pointer.
- A loop that ends when the pointer becomes `NULL`.

These clues should be considered together with the surrounding control flow and memory accesses.
