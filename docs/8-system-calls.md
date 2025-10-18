# 8. System Calls

## Objectives

In this section, you will learn:

*   The purpose and mechanism of system calls.
*   How user-space applications interact with the kernel.
*   Implementing a basic system call interface.
*   Creating simple system calls (e.g., `write`, `exit`).

## Key Concepts

*   **System Call:** A programmatic way in which a computer program requests a service from the kernel of the operating system.
*   **User Mode:** The CPU operating mode where applications run, with limited privileges.
*   **Kernel Mode:** The CPU operating mode where the operating system kernel runs, with full privileges.
*   **Interrupt/Software Interrupt:** A common mechanism used to trigger a system call, transitioning from user mode to kernel mode.
*   **System Call Table:** A table that maps system call numbers to their corresponding kernel functions.
*   **Parameters Passing:** Methods for passing arguments from user-space to kernel-space during a system call.

## Step-by-Step Implementation

This section will guide you through setting up a software interrupt handler for system calls, defining a system call table, and implementing a few basic system calls like `sys_write` (to print to the console) and `sys_exit`.

### 1. System Call Mechanism (Software Interrupt)

System calls provide a controlled interface for user-mode applications to request services from the kernel. A common mechanism for this transition from user mode to kernel mode is a software interrupt. We will use `int 0x80` as our system call interrupt vector.

First, we need to register an Interrupt Service Routine (ISR) for `int 0x80` in our IDT.

```c
// In idt.c (from 4-interrupts-exceptions.md)

// ... (inside idt_init function)

    // Set gate for system call interrupt (0x80)
    idt_set_gate(0x80, (uint32_t)isr128, 0x08, 0x8E); // isr128 is the assembly stub for int 0x80

// ... (rest of idt_init)
```

And in `isr_stubs.asm`, you would define `isr128`:

```assembly
; In isr_stubs.asm

ISR_NOERRCODE 128 ; For int 0x80 (system call)
```

### 2. System Call Numbers and Table

Each system call will have a unique number. The kernel will use this number to look up the corresponding function in a system call table.

```c
// In a header file, e.g., syscall.h

#ifndef SYSCALL_H
#define SYSCALL_H

#include <stdint.h>
#include <stddef.h> // For size_t

// System call numbers
enum syscall_numbers {
    SYS_EXIT,
    SYS_WRITE,
    // Add more system calls here
    NUM_SYSCALLS
};

// Function pointer type for system calls
typedef int (*syscall_handler_t)(uint32_t, uint32_t, uint32_t, uint32_t, uint32_t);

// System call table
extern syscall_handler_t syscall_table[NUM_SYSCALLS];

// Function to initialize system calls
void init_syscalls();

// Kernel-side system call implementations
int sys_exit(uint32_t status);
int sys_write(uint32_t fd, uint32_t buf, uint32_t count);

#endif // SYSCALL_H
```

```c
// In a C source file, e.g., syscall.c

#include "syscall.h"
#include "screen.h"
#include "task.h" // For task management (e.g., terminating current task)

syscall_handler_t syscall_table[NUM_SYSCALLS];

// Kernel-side implementation of sys_exit
int sys_exit(uint32_t status) {
    print_string("Syscall: sys_exit called with status ");
    print_dec(status);
    print_newline();
    // In a real OS, this would terminate the current process/thread.
    // For now, we'll just halt the system or the current task.
    if (current_task) {
        current_task->state = TASK_TERMINATED; // Mark current task as terminated
        schedule(); // Schedule another task
    }
    asm volatile("hlt"); // If no other tasks, halt
    return 0; // Should not be reached
}

// Kernel-side implementation of sys_write
int sys_write(uint32_t fd, uint32_t buf_ptr, uint32_t count) {
    // For simplicity, we only support writing to stdout (fd=1) to VGA console
    if (fd == 1) {
        const char* str = (const char*)buf_ptr;
        for (uint32_t i = 0; i < count; i++) {
            print_char(str[i], cursor_col, cursor_row, default_color);
            cursor_col++;
            if (cursor_col >= MAX_COLS) {
                print_newline();
            }
        }
        update_cursor(cursor_col, cursor_row);
        return count;
    }
    print_string("Syscall: sys_write - Unsupported file descriptor.\n");
    return -1; // Error
}

void init_syscalls() {
    syscall_table[SYS_EXIT] = (syscall_handler_t)sys_exit;
    syscall_table[SYS_WRITE] = (syscall_handler_t)sys_write;
    // Initialize other system calls here
}
```

### 3. Generic System Call Handler (in `isr_handlers.c`)

Our generic `isr_handler` needs to be modified to dispatch system calls when `int 0x80` occurs. The system call number will be passed in `EAX`, and arguments in `EBX`, `ECX`, `EDX`, `ESI`, `EDI`.

```c
// In isr_handlers.c (from 4-interrupts-exceptions.md)

#include "syscall.h" // Include syscall.h
// ... other includes

// ... (inside isr_handler function)

    if (regs.int_no == 0x80) { // System Call Interrupt
        // Get syscall number from EAX
        uint32_t syscall_num = regs.eax;

        if (syscall_num < NUM_SYSCALLS && syscall_table[syscall_num] != 0) {
            // Call the appropriate system call handler
            // Pass arguments from registers (EBX, ECX, EDX, ESI, EDI)
            int ret = syscall_table[syscall_num](regs.ebx, regs.ecx, regs.edx, regs.esi, regs.edi);
            regs.eax = ret; // Store return value in EAX for user-mode
        } else {
            print_string("Syscall: Invalid system call number: ");
            print_dec(syscall_num);
            print_newline();
            regs.eax = -1; // Return error
        }
    } else if (regs.int_no >= 32 && regs.int_no <= 47) { // Hardware IRQ
        // ... (existing IRQ handling)
    }

// ... (rest of isr_handler)
```

### 4. User-Mode Application (Conceptual and Wrapper)

To test system calls, we need a way for a "user-mode" task to invoke them. This involves setting up registers and triggering `int 0x80`.

```c
// In a header file, e.g., user_app.h

#ifndef USER_APP_H
#define USER_APP_H

#include <stdint.h>
#include <stddef.h> // For size_t

// User-mode wrapper for sys_write
static inline int write(int fd, const char* buf, size_t count) {
    int ret;
    asm volatile("int $0x80"
                 : "=a"(ret)                  // Output: EAX (return value)
                 : "a"(SYS_WRITE),             // Input: EAX (syscall number)
                   "b"((uint33_t)fd),          // Input: EBX (arg1)
                   "c"((uint33_t)buf),         // Input: ECX (arg2)
                   "d"((uint33_t)count)        // Input: EDX (arg3)
                 : "memory");                  // Clobbers memory
    return ret;
}

// User-mode wrapper for sys_exit
static inline void exit(int status) {
    asm volatile("int $0x80"
                 :                            // No output
                 : "a"(SYS_EXIT),              // Input: EAX (syscall number)
                   "b"((uint33_t)status)       // Input: EBX (arg1)
                 : "memory");
    while(1); // Should not be reached if exit works
}

#endif // USER_APP_H
```

### 5. Integrating into `kernel_main`

```c
// In kernel_main.c

#include "syscall.h"
#include "user_app.h" // Include user_app.h
// ... other includes

// Example user-mode task function
void user_task_a() {
    write(1, "Hello from user task A!\n", 25);
    write(1, "User task A exiting...\n", 23);
    exit(0);
}

void kernel_main() {
    // ... (previous setup like paging, IDT, PIC, serial_init, multitasking)

    clear_screen();
    print_string("Osmygod Kernel Booted!\n");

    init_syscalls(); // Initialize system call table

    init_multitasking();

    // Create a user-mode task
    create_task(user_task_a, 4096); // Create task A with 4KB stack

    print_string("Enabling interrupts and starting scheduler...\n");
    asm volatile("sti"); // Enable interrupts globally

    while(1) {
        // Idle loop for the kernel task
        // print_string("Idle kernel task...\n");
        // for (int i = 0; i < 10000000; i++);
    }
}
```

## Verification/Testing

To verify your system call implementation:

1.  **Build your Kernel:**
    Ensure your Makefile includes `syscall.c` in the compilation and linking process. You will also need to ensure `isr_stubs.asm` has the `ISR_NOERRCODE 128` definition.

2.  **Run in QEMU:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img
    ```

3.  **Expected Output:**
    You should see the following messages printed to the QEMU console:
    *   "Hello from user task A!"
    *   "User task A exiting..."
    *   "Syscall: sys_exit called with status 0"

    If `user_task_a` is the only active task, the system might then halt after `sys_exit` is called. This demonstrates that your user-mode task successfully invoked kernel services (writing to screen and exiting) via system calls.


## Further Reading/Resources

*   [OSDev Wiki - System Calls](https://wiki.osdev.org/System_Calls)
*   [OSDev Wiki - Software Interrupts](https://wiki.osdev.org/Software_Interrupts)