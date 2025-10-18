# 3. Memory Management

## Objectives

In this section, you will learn:

*   Different strategies for managing system memory.
*   The concepts of segmentation and paging.
*   How to set up page tables and enable paging.
*   Implementing a basic physical memory allocator.
*   Understanding virtual memory.

## Key Concepts

*   **Physical Memory:** The actual RAM installed in the system.
*   **Virtual Memory:** An abstraction that provides each process with its own isolated address space.
*   **Segmentation:** A memory management technique that divides memory into logical segments.
*   **Paging:** A memory management technique that divides memory into fixed-size blocks (pages) and maps virtual addresses to physical addresses.
*   **Page Table:** Data structures used by the CPU to translate virtual addresses to physical addresses.
*   **Page Directory:** The top-level page table in a multi-level paging scheme.
*   **Physical Memory Allocator (PMA):** A component that manages the allocation and deallocation of physical memory frames.

## Step-by-Step Implementation

This section will guide you through setting up a basic page directory and page tables, enabling paging, and implementing a simple bitmap-based physical memory allocator.

### 1. Introduction to Paging

Paging is a memory management scheme that allows the operating system to provide each process with its own isolated virtual address space. This offers several advantages:

*   **Memory Protection:** Prevents one process from corrupting another's memory.
*   **Virtual Memory:** Allows processes to use more memory than physically available by swapping pages to disk.
*   **Memory Isolation:** Each process sees a contiguous, private address space, simplifying programming.
*   **Physical Memory Management:** The OS can allocate physical memory frames non-contiguously, making efficient use of RAM.

In 32-bit protected mode, paging typically uses a two-level hierarchy: a Page Directory (PD) and Page Tables (PTs). Each entry in these tables is 4 bytes.

*   **Page Directory Entry (PDE):** Points to a Page Table.
*   **Page Table Entry (PTE):** Points to a 4KB physical memory frame.

### 2. Setting up a Page Directory and Page Tables

For simplicity, we will create an identity mapping for the first 4MB of memory. This means virtual addresses will directly map to their corresponding physical addresses (e.g., virtual address `0x1000` maps to physical address `0x1000`). This is common for the initial kernel setup.

We'll need a Page Directory and one Page Table to map the first 4MB (1024 entries * 4KB/entry = 4MB).

Let's assume we have a C environment set up after transitioning to protected mode. We'll need to define structures for our page directory and page table entries.

```c
// In a header file, e.g., memory.h

#ifndef MEMORY_H
#define MEMORY_H

#include <stdint.h>

// Page Directory Entry (PDE) and Page Table Entry (PTE) flags
#define PAGE_PRESENT    0x1     // Page is present in memory
#define PAGE_RW         0x2     // Read/Write (0=Read-only, 1=Read/Write)
#define PAGE_USER       0x4     // User/Supervisor (0=Supervisor, 1=User)

// A Page Directory is an array of 1024 PDEs
typedef uint32_t page_directory_entry_t;
page_directory_entry_t *page_directory = (page_directory_entry_t *)0x100000; // Example: Place PD at 1MB physical address

// A Page Table is an array of 1024 PTEs
typedef uint32_t page_table_entry_t;
page_table_entry_t *first_page_table = (page_table_entry_t *)0x101000; // Example: Place first PT at 1MB + 4KB physical address

// Function to initialize paging
void init_paging();

#endif // MEMORY_H
```

```c
// In a C source file, e.g., memory.c

#include "memory.h"

void init_paging() {
    // Clear the page directory and first page table
    for (int i = 0; i < 1024; i++) {
        page_directory[i] = 0;
        first_page_table[i] = 0;
    }

    // Map the first 4MB of physical memory to the first page table
    // Each entry maps a 4KB page
    for (uint32_t i = 0; i < 1024; i++) {
        // Map virtual address i * 4KB to physical address i * 4KB
        first_page_table[i] = (i * 0x1000) | PAGE_PRESENT | PAGE_RW; // Present, Read/Write
    }

    // Link the first page table to the first entry in the page directory
    // The address must be physical, and aligned to 4KB
    page_directory[0] = ((uint32_t)first_page_table) | PAGE_PRESENT | PAGE_RW; // Present, Read/Write

    // Now, enable paging (this part will be done in assembly)
}
```

### 3. Enabling Paging (Assembly)

After setting up the page directory and page tables in C, you need to enable paging using assembly instructions.

```assembly
; In your protected mode entry point (e.g., in boot_sector.asm or a separate kernel.asm)

extern init_paging ; Declare the C function

; ... (after protected_mode_start and setting up segment registers) ...

    call init_paging ; Call the C function to set up page tables

    ; Load the physical address of the Page Directory into CR3
    ; (Assuming page_directory is at 0x100000 physical address)
    mov eax, 0x100000
    mov cr3, eax

    ; Enable the Paging (PG) bit in CR0
    mov eax, cr0
    or eax, 0x80000000 ; Set the 31st bit (PG bit)
    mov cr0, eax

    ; Paging is now enabled!

    ; Continue with kernel execution...

    ; Example: Write to a virtual address that is identity mapped
    mov dword [0xB8000], 0x07410742 ; Still works as 0xB8000 is mapped

    jmp $
```

### 4. Physical Memory Allocator (PMA)

A Physical Memory Allocator is responsible for managing the physical RAM frames (typically 4KB in size). A simple bitmap allocator can be used.

First, you need to know the total amount of available physical memory. For simplicity in this example, we'll assume a fixed amount, but in a real OS, you'd get this information from the BIOS/UEFI (e.g., using `int 0x15, eax=0xE820`).

```c
// In memory.h

#define MAX_PHYSICAL_FRAMES (1024 * 1024) // Example: 4GB / 4KB = 1M frames

// Bitmap for physical memory frames. Each bit represents a 4KB frame.
// (MAX_PHYSICAL_FRAMES / 8) bytes needed for the bitmap.
uint8_t physical_memory_bitmap[MAX_PHYSICAL_FRAMES / 8];

void pma_init(uint32_t total_memory_bytes);
uint32_t pma_alloc_frame(); // Returns physical address of allocated frame
void pma_free_frame(uint32_t frame_address);
```

```c
// In memory.c

#include "memory.h"
#include <string.h> // For memset

// Global variables (initialized to 0 by default in .bss)
// uint8_t physical_memory_bitmap[MAX_PHYSICAL_FRAMES / 8];
uint32_t total_physical_frames = 0;

void pma_init(uint32_t total_memory_bytes) {
    total_physical_frames = total_memory_bytes / 0x1000; // 4KB frames
    // Mark all frames as free initially
    memset(physical_memory_bitmap, 0, sizeof(physical_memory_bitmap));

    // Mark the first few frames as used (e.g., for kernel, GDT, page tables)
    // This is a simplified example; a real OS would parse memory map
    for (uint32_t i = 0; i < (0x102000 / 0x1000); i++) { // Mark up to 1MB + 8KB as used
        // Set bit i in the bitmap
        physical_memory_bitmap[i / 8] |= (1 << (i % 8));
    }
}

uint32_t pma_alloc_frame() {
    for (uint32_t i = 0; i < total_physical_frames; i++) {
        if (!(physical_memory_bitmap[i / 8] & (1 << (i % 8)))) { // If frame is free
            physical_memory_bitmap[i / 8] |= (1 << (i % 8)); // Mark as used
            return i * 0x1000; // Return physical address
        }
    }
    return 0; // No free frame found
}

void pma_free_frame(uint32_t frame_address) {
    uint32_t frame_index = frame_address / 0x1000;
    if (frame_index < total_physical_frames) {
        physical_memory_bitmap[frame_index / 8] &= ~(1 << (frame_index % 8)); // Mark as free
    }
}
```

## Verification/Testing

To verify your memory management implementation:

1.  **Integrate Paging and PMA into your Kernel:**
    *   Modify your `boot_sector.asm` (or a new `kernel.asm` that `boot_sector.asm` loads) to call `init_paging` and then enable paging as shown in the assembly snippet.
    *   In your C kernel entry point (e.g., `kernel_main`), call `pma_init` with a suitable total memory size (e.g., `16 * 1024 * 1024` for 16MB).

2.  **Test Paging:**
    *   After enabling paging, try writing to a virtual address that is identity-mapped (e.g., `0xB8000` for VGA text mode). It should still work.
    *   *(Advanced: To test page faults, you would need interrupt handling set up, which is covered in the next section. For now, focus on successful mapping.)*

3.  **Test PMA:**
    *   In your `kernel_main` function, allocate a few frames using `pma_alloc_frame()`.
    *   Print the returned physical addresses to the screen (you'll need a basic print function for this, which will be covered in I/O).
    *   Free some frames using `pma_free_frame()`.
    *   Allocate again and observe if the freed frames are reused.

    ```c
    // Example in kernel_main (after pma_init)
    void kernel_main() {
        // ... (paging setup)

        pma_init(16 * 1024 * 1024); // Initialize PMA for 16MB

        uint32_t frame1 = pma_alloc_frame();
        uint32_t frame2 = pma_alloc_frame();
        uint32_t frame3 = pma_alloc_frame();

        // (You'll need a print function here to see the addresses)
        // print_string("Frame 1: "); print_hex(frame1); print_newline();
        // print_string("Frame 2: "); print_hex(frame2); print_newline();
        // print_string("Frame 3: "); print_hex(frame3); print_newline();

        pma_free_frame(frame2);

        uint32_t frame4 = pma_alloc_frame(); // Should ideally reuse frame2's address
        // print_string("Frame 4: "); print_hex(frame4); print_newline();

        while(1);
    }
    ```

4.  **Compile and Run in QEMU:**
    You will need a basic build system (Makefile) to compile your C code and link it with your assembly bootloader. For now, assume you have a way to compile `memory.c` into an object file and link it with your `boot_sector.bin` (or a separate kernel binary loaded by the boot sector).

    ```bash
    # Example (simplified, actual Makefile would be more complex)
    # nasm -f elf32 boot_entry.asm -o boot_entry.o
    # gcc -m32 -c memory.c -o memory.o -ffreestanding -O2 -Wall -Wextra
    # ld -m elf_i386 -T linker.ld -o kernel.bin boot_entry.o memory.o
    # dd if=kernel.bin of=osmygod.img bs=512 seek=1 conv=notrunc # Write kernel after boot sector
    # qemu-system-x86_64 -fda osmygod.img
    ```

5.  **Expected Output:**
    If successful, your system should boot into protected mode with paging enabled, and your PMA tests should show allocated and potentially reused physical memory frame addresses (once you have a printing mechanism). The system should not crash due to memory access violations within the mapped regions.

## Further Reading/Resources

*   [OSDev Wiki - Paging](https://wiki.osdev.org/Paging)
*   [OSDev Wiki - Memory Management](https://wiki.osdev.org/Memory_Management)