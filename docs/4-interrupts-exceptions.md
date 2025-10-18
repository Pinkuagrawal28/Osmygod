# 4. Interrupts and Exceptions

## Objectives

In this section, you will learn:

*   The difference between interrupts and exceptions.
*   How to set up the Interrupt Descriptor Table (IDT).
*   Handling hardware interrupts (e.g., keyboard, timer).
*   Handling CPU exceptions (e.g., page fault, general protection fault).
*   The role of the Programmable Interrupt Controller (PIC).

## Key Concepts

*   **Interrupt:** An event that diverts the CPU from its current execution to handle a specific task, typically from hardware.
*   **Exception:** An event that occurs synchronously with program execution, usually due to an error or an unusual condition.
*   **Interrupt Descriptor Table (IDT):** A data structure that maps interrupt and exception vectors to their corresponding handler routines.
*   **Interrupt Service Routine (ISR):** The function that is executed when an interrupt or exception occurs.
*   **Programmable Interrupt Controller (PIC):** A hardware component that manages interrupt requests from various devices and forwards them to the CPU.
*   **Interrupt Vector:** A number that identifies a specific interrupt or exception.

## Step-by-Step Implementation

This section will cover setting up the Interrupt Descriptor Table (IDT), writing basic Interrupt Service Routines (ISRs) for common exceptions and hardware interrupts, and configuring the Programmable Interrupt Controller (PIC).

### 1. Interrupt Descriptor Table (IDT) Structure

The IDT is a table that maps interrupt and exception vectors (numbers 0-255) to their corresponding handler routines. Each entry in the IDT is an 8-byte gate descriptor.

```c
// In a header file, e.g., idt.h

#ifndef IDT_H
#define IDT_H

#include <stdint.h>

// Structure for an IDT entry
struct idt_entry {
    uint16_t base_low;      // The lower 16 bits of the address of the ISR
    uint16_t selector;      // Our kernel code segment selector
    uint8_t  zero;          // This must always be zero
    uint8_t  flags;         // Type and attributes, e.g., 0x8E for 32-bit interrupt gate
    uint16_t base_high;     // The upper 16 bits of the address of the ISR
} __attribute__((packed));

// Structure for the IDT pointer (for lidt instruction)
struct idt_ptr {
    uint16_t limit;         // Size of the IDT - 1
    uint32_t base;          // Address of the first IDT entry
} __attribute__((packed));

// Declare the IDT array
struct idt_entry idt[256];
struct idt_ptr idt_ptr_reg;

// Function to set an IDT gate
void idt_set_gate(uint8_t num, uint32_t base, uint16_t selector, uint8_t flags);

// Function to initialize the IDT
void idt_init();

#endif // IDT_H
```

```c
// In a C source file, e.g., idt.c

#include "idt.h"
#include "screen.h" // Assuming you have a basic print function

// External assembly ISR stubs (defined in assembly)
extern void isr0();
extern void isr1();
// ... declare all 32 exception ISRs
extern void isr32(); // Timer
extern void isr33(); // Keyboard
// ... declare other hardware ISRs

void idt_set_gate(uint8_t num, uint32_t base, uint16_t selector, uint8_t flags) {
    idt[num].base_low = (base & 0xFFFF);
    idt[num].base_high = (base >> 16) & 0xFFFF;
    idt[num].selector = selector;
    idt[num].zero = 0;
    idt[num].flags = flags;
}

void idt_init() {
    idt_ptr_reg.limit = (sizeof(struct idt_entry) * 256) - 1;
    idt_ptr_reg.base = (uint32_t)&idt;

    // Clear the IDT
    for (int i = 0; i < 256; i++) {
        idt_set_gate(i, 0, 0, 0);
    }

    // Set gates for exceptions (0-31)
    idt_set_gate(0, (uint32_t)isr0, 0x08, 0x8E); // Example: Divide by Zero
    idt_set_gate(1, (uint32_t)isr1, 0x08, 0x8E); // Debug
    // ... set other exception gates

    // Set gates for hardware interrupts (remapped PIC IRQs 0-15 to 32-47)
    idt_set_gate(32, (uint32_t)isr32, 0x08, 0x8E); // Timer
    idt_set_gate(33, (uint32_t)isr33, 0x08, 0x8E); // Keyboard
    // ... set other hardware interrupt gates

    // Load the IDT
    asm volatile("lidt %0" : : "m"(idt_ptr_reg));

    // Enable interrupts (after PIC remapping)
    // asm volatile("sti");
}
```

### 2. Assembly Stubs for ISRs

When an interrupt or exception occurs, the CPU pushes some information onto the stack. Our assembly stubs are responsible for saving the rest of the CPU's state (general-purpose registers), pushing the interrupt number, and then calling a generic C handler. After the C handler returns, the stub restores the CPU state and returns from the interrupt.

```assembly
; In an assembly file, e.g., isr_stubs.asm

BITS 32

%macro ISR_NOERRCODE 1
    global isr%1
isr%1:
    cli
    push byte 0             ; Push a dummy error code (for consistency with error code ISRs)
    push byte %1            ; Push the interrupt number
    jmp isr_common_stub
%endmacro

%macro ISR_ERRCODE 1
    global isr%1
isr%1:
    cli
    push byte %1            ; Push the interrupt number
    jmp isr_common_stub
%endmacro

; Exceptions with no error code
ISR_NOERRCODE 0  ; Divide by Zero
ISR_NOERRCODE 1  ; Debug
; ... (up to 7)
ISR_ERRCODE 8    ; Double Fault (has error code)
; ... (up to 17)
ISR_NOERRCODE 18 ; Machine Check
ISR_ERRCODE 19   ; SIMD Floating-Point Exception
; ... (up to 31)

; Hardware Interrupts (IRQs 0-15, remapped to 32-47)
ISR_NOERRCODE 32 ; IRQ0 - Timer
ISR_NOERRCODE 33 ; IRQ1 - Keyboard
; ... (up to 47)

extern isr_handler

isr_common_stub:
    pusha                   ; Push all general-purpose registers (EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI)
    mov ax, ds              ; Save current data segment
    push eax
    mov ax, 0x10            ; Load kernel data segment selector (0x10 is typically the data segment selector)
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    call isr_handler        ; Call the C interrupt handler
    pop eax                 ; Restore data segment
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    popa                    ; Pop all general-purpose registers
    add esp, 8              ; Clean up pushed interrupt number and error code
    sti                     ; Re-enable interrupts
    iret                    ; Return from interrupt
```

### 3. Programmable Interrupt Controller (PIC) Configuration

The 8259 PIC manages hardware interrupts. By default, IRQs 0-7 map to interrupt vectors 0x08-0x0F, which conflict with CPU exceptions. We need to remap them to a higher range (e.g., 0x20-0x2F, or 32-47).

```c
// In a C source file, e.g., pic.c

#include <stdint.h>

// PIC ports
#define PIC1_COMMAND    0x20
#define PIC1_DATA       0x21
#define PIC2_COMMAND    0xA0
#define PIC2_DATA       0xA1

// End-of-Interrupt (EOI) command
#define PIC_EOI         0x20

// Function to write a byte to an I/O port
static inline void outb(uint16_t port, uint8_t val) {
    asm volatile("outb %0, %1" : : "a"(val), "Nd"(port));
}

// Function to read a byte from an I/O port
static inline uint8_t inb(uint16_t port) {
    uint8_t ret;
    asm volatile("inb %1, %0" : "=a"(ret) : "Nd"(port));
    return ret;
}

void pic_remap(int offset1, int offset2) {
    uint8_t a1, a2;

    a1 = inb(PIC1_DATA);
    a2 = inb(PIC2_DATA);

    outb(PIC1_COMMAND, 0x11); // Start initialization sequence
    outb(PIC2_COMMAND, 0x11);

    outb(PIC1_DATA, offset1); // ICW2: Master PIC vector offset
    outb(PIC2_DATA, offset2); // ICW2: Slave PIC vector offset

    outb(PIC1_DATA, 0x04);    // ICW3: Tell Master PIC that there is a slave PIC at IRQ2
    outb(PIC2_DATA, 0x02);    // ICW3: Tell Slave PIC its cascade identity

    outb(PIC1_DATA, 0x01);    // ICW4: 8086 mode
    outb(PIC2_DATA, 0x01);

    outb(PIC1_DATA, a1);      // Restore saved masks
    outb(PIC2_DATA, a2);
}

void pic_send_eoi(uint8_t irq) {
    if (irq >= 8) {
        outb(PIC2_COMMAND, PIC_EOI);
    }
    outb(PIC1_COMMAND, PIC_EOI);
}

void irq_set_mask(uint8_t irq_line) {
    uint16_t port;
    uint8_t value;

    if(irq_line < 8) {
        port = PIC1_DATA;
    } else {
        port = PIC2_DATA;
        irq_line -= 8;
    }
    value = inb(port) | (1 << irq_line);
    outb(port, value);
}

void irq_clear_mask(uint8_t irq_line) {
    uint16_t port;
    uint8_t value;

    if(irq_line < 8) {
        port = PIC1_DATA;
    } else {
        port = PIC2_DATA;
        irq_line -= 8;
    }
    value = inb(port) & ~(1 << irq_line);
    outb(port, value);
}
```

### 4. Writing Basic ISRs (C Handlers)

These are the C functions that your assembly stubs will call. They receive the interrupt number and an error code (if applicable).

```c
// In a C source file, e.g., isr_handlers.c

#include "screen.h" // For printing
#include "pic.h"    // For PIC_send_eoi

// Structure to hold CPU state pushed by assembly stub
struct registers {
    uint32_t edi, esi, ebp, esp, ebx, edx, ecx, eax; // Pushed by pusha
    uint32_t ds, es, fs, gs;                         // Pushed manually
    uint32_t int_no, err_code;                       // Pushed by stub
    uint32_t eip, cs, eflags, useresp, ss;           // Pushed by CPU
};

// Generic C interrupt handler
void isr_handler(struct registers regs) {
    print_string("Received interrupt: ");
    print_dec(regs.int_no);
    print_string(" Error code: ");
    print_dec(regs.err_code);
    print_newline();

    if (regs.int_no >= 32 && regs.int_no <= 47) { // Hardware IRQ
        pic_send_eoi(regs.int_no - 32);
    }

    // Specific handlers for exceptions
    if (regs.int_no == 0) {
        print_string("Divide by Zero Exception!\n");
        asm volatile("hlt"); // Halt the CPU
    } else if (regs.int_no == 14) { // Page Fault
        uint32_t cr2;
        asm volatile("mov %%cr2, %0" : "=r"(cr2));
        print_string("Page Fault at address: ");
        print_hex(cr2);
        print_string(" Error code: ");
        print_hex(regs.err_code);
        print_newline();
        asm volatile("hlt");
    } else if (regs.int_no == 13) { // General Protection Fault
        print_string("General Protection Fault!\n");
        asm volatile("hlt");
    }

    // Specific handlers for hardware interrupts
    if (regs.int_no == 32) { // Timer IRQ0
        // print_string("Tick!\n"); // Too chatty, maybe increment a counter
    }
    if (regs.int_no == 33) { // Keyboard IRQ1
        uint8_t scancode = inb(0x60); // Read scancode from keyboard data port
        print_string("Keyboard Scancode: ");
        print_hex(scancode);
        print_newline();
    }
}
```

### 5. Initializing Interrupts in your Kernel

In your `kernel_main` function (or equivalent C entry point):

```c
// In kernel_main.c

#include "idt.h"
#include "pic.h"
#include "screen.h" // For print_string, print_dec, print_hex, print_newline

void kernel_main() {
    // ... (previous setup like paging)

    print_string("Initializing IDT...\n");
    idt_init();

    print_string("Remapping PIC...\n");
    pic_remap(0x20, 0x28); // Remap IRQs to 0x20-0x2F

    print_string("Clearing IRQ masks...\n");
    irq_clear_mask(0); // Enable Timer IRQ
    irq_clear_mask(1); // Enable Keyboard IRQ

    print_string("Enabling interrupts...\n");
    asm volatile("sti"); // Enable interrupts globally

    print_string("System ready!\n");

    // Test exceptions
    // print_string("Testing divide by zero...\n");
    // int a = 1 / 0; // This should trigger a divide by zero exception

    // Test page fault (if paging is enabled and you try to access unmapped memory)
    // uint32_t *ptr = (uint32_t *)0xC0000000; // Example unmapped address
    // *ptr = 0xDEADBEEF; // This should trigger a page fault

    while(1);
}
```

## Verification/Testing

To verify your interrupt and exception handling:

1.  **Build your Kernel:**
    You will need a Makefile to compile your C files (`idt.c`, `pic.c`, `isr_handlers.c`, `kernel_main.c`, `screen.c` for printing) and assemble your assembly stubs (`isr_stubs.asm`). Link them together with your bootloader.

    ```makefile
    # Simplified Makefile example
    CC = gcc
    AS = nasm
    LD = ld

    CFLAGS = -m32 -c -ffreestanding -O2 -Wall -Wextra -I.
    ASFLAGS = -f elf32
    LDFLAGS = -m elf_i386 -T linker.ld

    OBJS = boot_entry.o idt.o pic.o isr_handlers.o kernel_main.o screen.o isr_stubs.o

    all: osmygod.img

osmygod.img: kernel.bin
	dd if=/dev/zero of=osmygod.img bs=512 count=2880
	dd if=boot_sector.bin of=osmygod.img bs=512 count=1 conv=notrunc
	dd if=kernel.bin of=osmygod.img bs=512 seek=1 conv=notrunc

kernel.bin: $(OBJS)
	$(LD) $(LDFLAGS) -o kernel.bin $(OBJS)

boot_entry.o: boot_entry.asm
	$(AS) $(ASFLAGS) boot_entry.asm -o boot_entry.o

isr_stubs.o: isr_stubs.asm
	$(AS) $(ASFLAGS) isr_stubs.asm -o isr_stubs.o

%.o: %.c
	$(CC) $(CFLAGS) $< -o $@

clean:
	rm -f *.o *.bin *.img
    ```
    *(Note: `boot_entry.asm` would be your assembly file that transitions to protected mode and calls `kernel_main`. `linker.ld` is a linker script that defines memory layout.)*

2.  **Run in QEMU:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img
    ```

3.  **Expected Output:**
    *   **Initialization Messages:** You should see "Initializing IDT...", "Remapping PIC...", "Clearing IRQ masks...", "Enabling interrupts...", "System ready!" printed to the screen.
    *   **Timer Interrupts:** If the timer IRQ is unmasked, you might see periodic "Tick!" messages (if you uncommented it) or observe a global counter incrementing.
    *   **Keyboard Interrupts:** When you press keys on your keyboard, you should see "Keyboard Scancode: [hex_value]" printed for each key press/release.
    *   **Divide by Zero Exception:** If you uncomment `int a = 1 / 0;`, the system should print "Received interrupt: 0 Error code: 0\nDivide by Zero Exception!\n" and then halt.
    *   **Page Fault Exception:** If you uncomment the page fault test and access an unmapped address, the system should print "Received interrupt: 14 Error code: [hex_value]\nPage Fault at address: [hex_address]\n" and then halt.

If you observe these behaviors, your interrupt and exception handling is working correctly!

## Further Reading/Resources

*   [OSDev Wiki - Interrupts](https://wiki.osdev.org/Interrupts)
*   [OSDev Wiki - Exceptions](https://wiki.osdev.org/Exceptions)
*   [OSDev Wiki - PIC](https://wiki.osdev.org/PIC)