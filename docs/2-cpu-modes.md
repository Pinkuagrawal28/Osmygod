# 2. CPU Modes

## Objectives

In this section, you will learn:

*   The characteristics and limitations of Real Mode.
*   The features and advantages of Protected Mode.
*   How to transition from Real Mode to Protected Mode.
*   The basics of Long Mode (64-bit) and its significance.

## Key Concepts

*   **Real Mode:** 16-bit mode, 1MB addressable memory, direct hardware access.
*   **Protected Mode:** 32-bit mode, up to 4GB addressable memory, memory protection, multitasking support, segmentation, paging.
*   **Global Descriptor Table (GDT):** A data structure that defines memory segments in protected mode.
*   **Control Registers (CR0, CR4):** Registers used to control CPU operating modes and features.
*   **Long Mode:** 64-bit mode, enabling access to vast amounts of memory and 64-bit instructions.

## Step-by-Step Implementation

This section will detail the process of setting up the Global Descriptor Table (GDT), enabling the A20 line, and finally switching the CPU to 32-bit protected mode. We will also briefly touch upon the initial steps for entering long mode.

### 1. Real Mode Limitations Revisited

As discussed, Real Mode operates in 16-bit, can only address 1MB of memory, and lacks memory protection. To build a modern operating system, we must transition to Protected Mode.

### 2. Enabling the A20 Line

The A20 line is the 21st address line of the CPU. In Real Mode, it's typically disabled to maintain compatibility with older 8086 processors (which only had 20 address lines). To access more than 1MB of memory in Protected Mode, the A20 line must be enabled.

A common way to enable the A20 line is through the keyboard controller. This involves sending commands to port `0x64` and `0x60`.

```assembly
; Function to enable A20 line
enable_a20:
    ; Wait for keyboard controller to be ready (input buffer empty)
    call wait_kbc_input
    mov al, 0xD1        ; Command: Write to output port
    out 0x64, al

    ; Wait for keyboard controller to be ready (input buffer empty)
    call wait_kbc_input
    mov al, 0xDF        ; Data: Enable A20 (bit 1 of output port)
    out 0x60, al

    ret

wait_kbc_input:
    in al, 0x64         ; Read status register
    test al, 2          ; Test if input buffer is full (bit 1)
    jnz wait_kbc_input  ; If full, wait
    ret
```

### 3. Global Descriptor Table (GDT) Setup

The GDT is a crucial data structure in Protected Mode. It defines memory segments, including their base address, limit, and access rights (e.g., executable, writable, privilege level).

Each entry in the GDT is an 8-byte segment descriptor. We will define a minimal GDT with:

*   A **Null Descriptor** (required, all zeros).
*   A **Code Segment Descriptor** (for our 32-bit kernel code).
*   A **Data Segment Descriptor** (for our 32-bit kernel data).

```assembly
; GDT definition
gdt_start:
    ; Null Descriptor (required)
    dq 0

; Code Segment Descriptor
; Base = 0, Limit = 0xFFFFF (4GB), Present, DPL=0, Executable, Read/Write, 32-bit
code_segment:
    dw 0xFFFF           ; Segment Limit (low)
    dw 0x0000           ; Base Address (low)
    db 0x00             ; Base Address (middle)
    db 10011010b        ; Access Byte: Present, DPL=0, Code, Executable, Read/Write
    db 11001111b        ; Flags (Granularity=1, 32-bit) | Segment Limit (high)
    db 0x00             ; Base Address (high)

; Data Segment Descriptor
; Base = 0, Limit = 0xFFFFF (4GB), Present, DPL=0, Writable, 32-bit
data_segment:
    dw 0xFFFF           ; Segment Limit (low)
    dw 0x0000           ; Base Address (low)
    db 0x00             ; Base Address (middle)
    db 10010010b        ; Access Byte: Present, DPL=0, Data, Writable
    db 11001111b        ; Flags (Granularity=1, 32-bit) | Segment Limit (high)
    db 0x00             ; Base Address (high)

gdt_end:

; GDT Descriptor (for LGDT instruction)
gdt_descriptor:
    dw gdt_end - gdt_start - 1  ; Limit (size of GDT - 1)
    dd gdt_start                ; Base Address of GDT

; Define segment selectors for convenience
CODE_SEG_SELECTOR equ code_segment - gdt_start
DATA_SEG_SELECTOR equ data_segment - gdt_start
```

### 4. Transition to Protected Mode

With the A20 line enabled and the GDT defined, we can now switch to Protected Mode.

```assembly
; --- Protected Mode Transition --- 

; 1. Disable Interrupts
cli

; 2. Enable A20 Line (call the function defined above)
call enable_a20

; 3. Load GDT
lgdt [gdt_descriptor]

; 4. Set the Protected Mode Enable (PE) bit in CR0
mov eax, cr0
or eax, 0x1
mov cr0, eax

; 5. Far Jump to flush pipeline and load segment registers with 32-bit selectors
;    The 'CODE_SEG_SELECTOR' is the offset of our code segment descriptor in the GDT.
;    The target address 'protected_mode_start' is where execution will continue in 32-bit mode.
jmp CODE_SEG_SELECTOR:protected_mode_start

BITS 32 ; Inform assembler that subsequent code is 32-bit

protected_mode_start:
    ; Now in 32-bit Protected Mode!
    ; Reload all data segment registers with the 32-bit data segment selector
    mov ax, DATA_SEG_SELECTOR
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    mov esp, 0x90000    ; Set up a stack for 32-bit mode (e.g., at 576KB)

    ; Print a message to the screen in Protected Mode
    ; Direct VGA text mode memory access (0xB8000)
    mov dword [0xB8000], 0x07410742 ; 'AB' in white on black
    mov dword [0xB8004], 0x07430744 ; 'CD'

    ; Hang the system
    jmp $
```

### 5. Entering Long Mode (64-bit) - Brief Overview

Transitioning to Long Mode (64-bit) from Protected Mode is a more complex process, typically involving:

*   **Enabling PAE (Physical Address Extension):** Setting the PAE bit in `CR4`.
*   **Setting up PML4 (Page Map Level 4) Table:** The top-level page table for 64-bit paging.
*   **Setting up PDPT (Page Directory Pointer Table), PD (Page Directory), and PT (Page Table):** To map virtual addresses to physical addresses.
*   **Enabling Long Mode:** Setting the LME bit in the EFER (Extended Feature Enable Register) MSR (Model Specific Register).
*   **Enabling Paging:** Setting the PG bit in `CR0`.
*   **Far Jump:** A jump to a 64-bit code segment.

Detailed implementation of Long Mode will be covered in a more advanced section, likely after a thorough understanding of memory management and paging in 32-bit Protected Mode.

## Verification/Testing

To verify the CPU mode transition, you will integrate the GDT setup and Protected Mode transition code into your `boot_sector.asm`.

1.  **Update `boot_sector.asm`:**
    Replace the `jmp # 2. CPU Modes

## Objectives

In this section, you will learn:

*   The characteristics and limitations of Real Mode.
*   The features and advantages of Protected Mode.
*   How to transition from Real Mode to Protected Mode.
*   The basics of Long Mode (64-bit) and its significance.

## Key Concepts

*   **Real Mode:** 16-bit mode, 1MB addressable memory, direct hardware access.
*   **Protected Mode:** 32-bit mode, up to 4GB addressable memory, memory protection, multitasking support, segmentation, paging.
*   **Global Descriptor Table (GDT):** A data structure that defines memory segments in protected mode.
*   **Control Registers (CR0, CR4):** Registers used to control CPU operating modes and features.
*   **Long Mode:** 64-bit mode, enabling access to vast amounts of memory and 64-bit instructions.

 at the end of your `main` routine in `boot_sector.asm` with the Protected Mode transition code provided above. Ensure you include the `enable_a20` function, GDT definition, and the `gdt_descriptor`.

2.  **Assemble and Create Disk Image:**
    ```bash
    nasm -f bin boot_sector.asm -o boot_sector.bin
    dd if=/dev/zero of=osmygod.img bs=512 count=2880
    dd if=boot_sector.bin of=osmygod.img bs=512 count=1 conv=notrunc
    ```

3.  **Run in QEMU:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img
    ```

4.  **Expected Output:**
    After running in QEMU, you should see the initial "Hello from Osmygod Bootloader!" message (from Real Mode), followed by the characters "ABCD" (or similar, depending on your VGA memory writes) in the top-left corner of the QEMU window. This indicates that your system successfully transitioned to 32-bit Protected Mode and was able to write directly to VGA memory, which is not possible in Real Mode using BIOS interrupts for text output. If you see this, your Protected Mode transition is successful!

## Further Reading/Resources

*   [OSDev Wiki - CPU Modes](https://wiki.osdev.org/CPU_Modes)
*   [OSDev Wiki - Global Descriptor Table](https://wiki.osdev.org/Global_Descriptor_Table)