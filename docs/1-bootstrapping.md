# 1. Bootstrapping

## Objectives

In this section, you will learn:

*   The sequence of events from power-on to OS kernel execution.
*   The role of BIOS/UEFI in the boot process.
*   How to create a bootable image.
*   The basics of real mode and protected mode entry.

## Key Concepts

*   **Power-On Self-Test (POST):** Initial hardware checks performed by the BIOS/UEFI.
*   **BIOS/UEFI:** Firmware responsible for initializing hardware and loading the bootloader.
*   **Bootloader:** A small program that loads the operating system kernel into memory.
*   **Master Boot Record (MBR) / GUID Partition Table (GPT):** Disk partitioning schemes.
*   **Real Mode:** The initial operating mode of x86 processors, with 1MB addressable memory.
*   **Protected Mode:** An advanced operating mode providing access to all memory, multitasking, and memory protection.

## Step-by-Step Implementation

This section will guide you through creating a bootable sector, writing a simple bootloader, and the initial steps towards transitioning to protected mode.

### 1. Setting up your Development Environment

You will need the following tools:

*   **NASM (Netwide Assembler):** For assembling assembly code.
*   **QEMU:** An emulator to run your operating system without real hardware.
*   **GNU GCC (Optional, for later kernel development):** A C compiler.

On a Linux system, you can typically install them using:
```bash
sudo apt update
sudo apt install nasm qemu-system-x86 gcc
```

### 2. Creating a Bootable Sector

A bootable sector is the first 512 bytes of a storage device (like a hard drive or floppy disk) that the BIOS/UEFI loads into memory at address `0x7C00`. It must end with the magic number `0xAA55`.

Let's create a simple boot sector that prints a message to the screen.

Create a file named `boot_sector.asm`:

```assembly
; boot_sector.asm
ORG 0x7C00              ; Tell assembler that our code will be loaded at 0x7C00

BITS 16                 ; We are in 16-bit real mode

start:
    jmp short main      ; Jump to the main part of our bootloader

message:
    db "Hello from Osmygod Bootloader!", 0 ; Our message, null-terminated

main:
    ; Set up segment registers
    mov ax, 0x0000
    mov ds, ax
    mov es, ax
    mov ss, ax
    mov sp, 0x7C00      ; Stack grows downwards from 0x7C00

    ; Print message to screen using BIOS interrupt 0x10
    mov si, message     ; Load address of message into SI
    mov ah, 0x0E        ; BIOS teletype function
    mov bh, 0x00        ; Page number
    mov bl, 0x07        ; Text attribute (light grey on black)

.print_loop:
    lodsb               ; Load byte from [si] into AL, increment SI
    cmp al, 0           ; Check if null terminator
    je .end_print       ; If yes, end loop
    int 0x10            ; Call BIOS to print character
    jmp .print_loop     ; Loop back

.end_print:
    ; Hang the system (infinite loop)
    jmp $

times 510 - ($ - $) db 0 ; Fill remaining bytes with zeros
dw 0xAA55                 ; Boot signature (magic number)
```

### 3. Compiling and Creating a Bootable Image

Now, let's assemble this code and create a disk image.

1.  **Assemble the boot sector:**
    ```bash
    nasm -f bin boot_sector.asm -o boot_sector.bin
    ```
    This command tells NASM to output a raw binary file (`-f bin`) named `boot_sector.bin`.

2.  **Create a disk image:**
    ```bash
    dd if=/dev/zero of=osmygod.img bs=512 count=2880
    dd if=boot_sector.bin of=osmygod.img bs=512 count=1 conv=notrunc
    ```
    *   The first `dd` command creates an empty floppy disk image (`osmygod.img`) of 1.44MB (2880 sectors * 512 bytes/sector).
    *   The second `dd` command writes your `boot_sector.bin` to the first sector of `osmygod.img` without truncating the rest of the image (`conv=notrunc`).

### 4. Initial Steps Towards Protected Mode (Conceptual)

The transition from Real Mode (16-bit) to Protected Mode (32-bit) is a critical step for any modern OS. It involves several key actions:

1.  **Disable Interrupts:** To prevent unexpected behavior during the mode switch.
2.  **Enable A20 Line:** To allow access to more than 1MB of memory.
3.  **Load Global Descriptor Table (GDT):** The GDT defines memory segments for Protected Mode. (Detailed GDT setup will be covered in the "CPU Modes" section).
4.  **Set the Protected Mode Enable (PE) bit in CR0:** This is the final step that switches the CPU to Protected Mode.
5.  **Far Jump:** A jump to flush the CPU's instruction pipeline and load segment registers with Protected Mode selectors.

Here's a simplified conceptual snippet for the transition (actual GDT setup is omitted for brevity here):

```assembly
; ... (after bootloader code) ...

; Disable interrupts
cli

; Enable A20 line (various methods, often involves keyboard controller)
; For simplicity, we'll assume it's enabled or use a common method:
; mov al, 0xD1
; out 0x64, al
; mov al, 0xDF
; out 0x60, al

; Load GDT (lgdt instruction) - GDT definition would be elsewhere
; lgdt [gdt_descriptor]

; Set PE bit in CR0 to enter Protected Mode
mov eax, cr0
or eax, 0x1
mov cr0, eax

; Far jump to flush pipeline and load segment registers
; This jump will be to a 32-bit code segment selector
; jmp CODE_SEG_SELECTOR:protected_mode_start

; protected_mode_start:
;   BITS 32
;   ; Now in 32-bit protected mode
;   ; Reload segment registers with 32-bit selectors
;   ; ...
```
*(The full implementation of GDT and the complete Protected Mode transition will be covered in the next section, "2. CPU Modes".)*

## Verification/Testing

You can test your bootable image using the QEMU emulator.

1.  **Run QEMU with your image:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img
    ```
    *   `qemu-system-x86_64`: Specifies the x86-64 architecture emulator.
    *   `-fda osmygod.img`: Tells QEMU to use `osmygod.img` as a floppy disk image.

2.  **Expected Output:**
    If your `boot_sector.asm` is correctly assembled and written to the image, QEMU should open a new window and display:

    ```
    Hello from Osmygod Bootloader!
    ```

    The system will then halt in an infinite loop as programmed. If you see this message, your bootloader has successfully loaded and executed!

## Further Reading/Resources

*   [OSDev Wiki - Boot Sequence](https://wiki.osdev.org/Boot_Sequence)
*   [OSDev Wiki - Protected Mode](https://wiki.osdev.org/Protected_Mode)