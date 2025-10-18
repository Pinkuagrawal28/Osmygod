# 6. Process and Thread Management

## Objectives

In this section, you will learn:

*   The difference between processes and threads.
*   How to implement a basic process scheduler.
*   Context switching between different tasks.
*   Managing process states (running, ready, blocked).
*   Basic inter-process communication (IPC) concepts.

## Key Concepts

*   **Process:** An instance of a computer program that is being executed. It has its own memory space, registers, and resources.
*   **Thread:** A single sequence of execution within a process. Threads within the same process share memory and resources.
*   **Process Control Block (PCB):** A data structure containing information about a process (e.g., PID, state, registers, memory).
*   **Scheduler:** The part of the operating system that decides which process or thread should run next.
*   **Context Switch:** The process of saving the state of one process/thread and restoring the state of another.
*   **Time Slicing:** A scheduling technique where each process is given a small amount of CPU time (quantum) in a round-robin fashion.

## Step-by-Step Implementation

This section will guide you through creating a simple task scheduler, implementing context switching using assembly, and managing a basic list of processes/threads.

### 1. Processes vs. Threads (Clarification)

*   **Process:** An independent execution environment with its own dedicated memory space, resources (file handles, etc.), and at least one thread of execution. Creating a new process is resource-intensive.
*   **Thread:** A single sequence of execution within a process. Threads within the same process share the same memory space and resources, making them lightweight to create and switch between.

For simplicity in this initial OS, we will focus on managing independent "tasks" that are essentially kernel-level threads, sharing the kernel's address space but having their own stacks and CPU contexts. True process isolation with separate address spaces will build upon the paging concepts from the memory management section.

### 2. Process Control Block (PCB) / Task Structure

We need a data structure to save the state of a task when it's not running and restore it when it is scheduled. This is often called a Process Control Block (PCB) or, for threads, a Thread Control Block (TCB). We'll use a `task_t` structure.

```c
// In a header file, e.g., task.h

#ifndef TASK_H
#define TASK_H

#include <stdint.h>

// Define the maximum number of tasks our simple scheduler can handle
#define MAX_TASKS 10

// Enum for task states
enum task_state {
    TASK_READY,
    TASK_RUNNING,
    TASK_BLOCKED, // For future use
    TASK_TERMINATED // For future use
};

// Structure to save CPU registers during a context switch
// This must match the order of registers pushed/popped by the assembly stub
struct cpu_state {
    uint32_t eax;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
    uint32_t esp; // This will be the stack pointer for the task
    uint32_t eflags;
    uint32_t eip; // Instruction pointer
};

// Task Control Block (TCB)
typedef struct task {
    struct cpu_state cpu_state; // Saved CPU state
    uint32_t stack_ptr;         // Current stack pointer
    uint32_t kernel_stack;      // Pointer to the task's kernel stack
    enum task_state state;      // Current state of the task
    uint32_t id;                // Unique task ID
    // Add more fields as needed (e.g., priority, sleep_time, etc.)
} task_t;

// Global task list and current task pointer
extern task_t tasks[MAX_TASKS];
extern task_t *current_task;
extern uint32_t next_task_id;

// Function prototypes
void init_multitasking();
void create_task(void (*entry_point)(), uint32_t stack_size);
void schedule();

#endif // TASK_H
```

```c
// In a C source file, e.g., task.c

#include "task.h"
#include "screen.h"
#include "memory.h" // For pma_alloc_frame

task_t tasks[MAX_TASKS];
task_t *current_task = 0;
uint32_t next_task_id = 0;

// Assembly function for context switch
extern void switch_task_asm(uint32_t *old_esp, uint32_t new_esp);

void init_multitasking() {
    for (int i = 0; i < MAX_TASKS; i++) {
        tasks[i].state = TASK_TERMINATED; // All tasks initially terminated
    }
    // Create an initial task for the kernel itself
    // This task will represent the main kernel loop
    current_task = &tasks[0];
    current_task->id = next_task_id++;
    current_task->state = TASK_RUNNING;
    // The kernel's stack is already set up, so we just use its current ESP
    asm volatile("mov %%esp, %0" : "=r"(current_task->stack_ptr));
}

void create_task(void (*entry_point)(), uint32_t stack_size) {
    // Find a free task slot
    int new_task_idx = -1;
    for (int i = 0; i < MAX_TASKS; i++) {
        if (tasks[i].state == TASK_TERMINATED) {
            new_task_idx = i;
            break;
        }
    }

    if (new_task_idx == -1) {
        print_string("Error: No free task slots!\n");
        return;
    }

    task_t *new_task = &tasks[new_task_idx];

    // Allocate a kernel stack for the new task
    // We'll use our physical memory allocator for simplicity, but a proper
    // virtual memory setup would be better.
    uint32_t stack_phys_addr = pma_alloc_frame(); // Allocate 4KB for stack
    if (stack_phys_addr == 0) {
        print_string("Error: Could not allocate stack for new task!\n");
        return;
    }
    // For identity mapping, virtual == physical
    new_task->kernel_stack = stack_phys_addr + 0x1000; // Stack grows downwards, point to top

    // Set up initial stack frame for the new task
    // Simulate a call to the entry_point function
    uint32_t *stack = (uint32_t *)new_task->kernel_stack;

    // Push dummy values for registers that switch_task_asm will pop
    *(--stack) = 0x202; // EFLAGS (interrupts enabled)
    *(--stack) = (uint32_t)entry_point; // EIP
    *(--stack) = 0; // EAX
    *(--stack) = 0; // EBX
    *(--stack) = 0; // ECX
    *(--stack) = 0; // EDX
    *(--stack) = 0; // ESI
    *(--stack) = 0; // EDI
    *(--stack) = 0; // EBP

    new_task->stack_ptr = (uint32_t)stack; // Save the new stack pointer
    new_task->id = next_task_id++;
    new_task->state = TASK_READY;

    print_string("Created task with ID: ");
    print_dec(new_task->id);
    print_newline();
}

void schedule() {
    if (!current_task) return; // Should not happen after init_multitasking

    int next_task_idx = -1;
    int current_idx = -1;

    // Find current task's index
    for (int i = 0; i < MAX_TASKS; i++) {
        if (&tasks[i] == current_task) {
            current_idx = i;
            break;
        }
    }

    // Find the next READY task in a round-robin fashion
    for (int i = 1; i <= MAX_TASKS; i++) {
        int idx = (current_idx + i) % MAX_TASKS;
        if (tasks[idx].state == TASK_READY || tasks[idx].state == TASK_RUNNING) {
            next_task_idx = idx;
            break;
        }
    }

    if (next_task_idx == -1) {
        // No other tasks are ready, just keep running current task or halt
        return;
    }

    task_t *next_task = &tasks[next_task_idx];

    if (next_task == current_task) {
        return; // No need to switch if next task is current task
    }

    // Update states
    if (current_task->state == TASK_RUNNING) {
        current_task->state = TASK_READY;
    }
    next_task->state = TASK_RUNNING;

    task_t *old_task = current_task;
    current_task = next_task;

    // Perform the context switch
    switch_task_asm(&old_task->stack_ptr, current_task->stack_ptr);
}
```

### 3. Context Switching (Assembly)

The `switch_task_asm` function is critical. It saves the current CPU state onto the old task's stack and loads the new CPU state from the new task's stack. This is typically done in assembly because it directly manipulates the stack pointer (`ESP`) and other registers.

```assembly
; In an assembly file, e.g., context_switch.asm

BITS 32

global switch_task_asm
switch_task_asm:
    ; Arguments: [ESP+4] = old_esp_ptr, [ESP+8] = new_esp

    ; Save current task's context onto its stack
    push ebp
    mov ebp, esp

    push eax
    push ebx
    push ecx
    push edx
    push esi
    push edi

    ; Save current ESP into old_esp_ptr
    mov eax, [ebp + 8] ; old_esp_ptr
    mov [eax], esp

    ; Load new ESP
    mov esp, [ebp + 12] ; new_esp

    ; Restore new task's context from its stack
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop eax

    pop ebp

    ret ; Return to the new task's EIP (which was popped as part of the stack setup)
```

### 4. Integrating with the Timer Interrupt

To enable preemptive multitasking, we need to call `schedule()` periodically from the timer interrupt handler.

```c
// In your isr_handlers.c (from previous section)

#include "task.h" // Include task.h
// ... other includes

// Global tick counter (optional)
uint32_t timer_ticks = 0;

// ... (inside isr_handler function)

    if (regs.int_no == 32) { // Timer IRQ0
        timer_ticks++;
        // print_string("Tick! "); print_dec(timer_ticks); print_newline();

        // Call the scheduler on each timer tick
        schedule();
    }

// ... (rest of isr_handler)
```

### 5. Initial Task Setup in `kernel_main`

```c
// In kernel_main.c

#include "task.h"
#include "screen.h"
// ... other includes

// Example task functions
void task_a() {
    while (1) {
        print_string("Task A running...\n");
        for (int i = 0; i < 10000000; i++); // Simple delay
    }
}

void task_b() {
    while (1) {
        print_string("Task B running...\n");
        for (int i = 0; i < 10000000; i++); // Simple delay
    }
}

void kernel_main() {
    // ... (previous setup like paging, IDT, PIC, serial_init)

    clear_screen();
    print_string("Osmygod Kernel Booted!\n");

    init_multitasking();

    create_task(task_a, 4096); // Create task A with 4KB stack
    create_task(task_b, 4096); // Create task B with 4KB stack

    print_string("Enabling interrupts and starting scheduler...\n");
    asm volatile("sti"); // Enable interrupts globally

    // The initial kernel task will now effectively become the idle task
    // or you can have it enter an infinite loop to keep the CPU busy
    while(1) {
        // print_string("Idle task running...\n");
        // asm volatile("hlt"); // Halt CPU until next interrupt
    }
}
```

## Verification/Testing

To verify your process and thread management:

1.  **Build your Kernel:**
    Ensure your Makefile includes `task.c` and `context_switch.asm` in the compilation and linking process. You will also need to link with `memory.o` for `pma_alloc_frame`.

2.  **Run in QEMU:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img
    ```

3.  **Expected Output:**
    You should see the output from "Task A running..." and "Task B running..." interleaved on the QEMU screen. This demonstrates that your scheduler is successfully switching between the tasks, providing a basic form of multitasking.

    The exact interleaving might vary depending on the timer frequency and the delays in your tasks, but the key is to see both tasks executing concurrently. If you only see output from one task, or the system crashes, there's an issue with your context switching or scheduler logic.


## Further Reading/Resources

*   [OSDev Wiki - Multitasking](https://wiki.osdev.org/Multitasking)
*   [OSDev Wiki - Processes](https://wiki.osdev.org/Processes)