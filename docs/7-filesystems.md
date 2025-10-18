# 7. Filesystems

## Objectives

In this section, you will learn:

*   The fundamental concepts of how data is organized and stored on persistent media.
*   Basic disk structures (sectors, tracks, cylinders).
*   Implementing a simple in-memory or RAM-disk based filesystem.
*   Concepts of file allocation (FAT, inodes).
*   Basic file operations (create, read, write, delete).

## Key Concepts

*   **Filesystem:** A method and data structure that an operating system uses to control how data is stored and retrieved.
*   **Disk Sector:** The smallest physical storage unit on a disk.
*   **File Allocation Table (FAT):** A simple filesystem structure that uses a table to keep track of where files are stored on the disk.
*   **Inode:** A data structure in Unix-like filesystems that stores information about a file or directory (metadata).
*   **Directory Entry:** An entry in a directory that maps a filename to its corresponding file data or metadata.
*   **Block Device:** A device that moves data in fixed-size blocks (e.g., hard drives, SSDs).

## Step-by-Step Implementation

This section will guide you through designing and implementing a very basic in-memory (RAM-disk based) filesystem, demonstrating how to create files, write data to them, and read data back.

### 1. RAM Disk Simulation

To simplify, we'll simulate a disk using a large array in memory. This avoids complex hardware interaction with actual disk controllers for now.

```c
// In a header file, e.g., filesystem.h

#ifndef FILESYSTEM_H
#define FILESYSTEM_H

#include <stdint.h>
#include <stdbool.h>

#define BLOCK_SIZE          512     // Size of each data block (like a sector)
#define TOTAL_BLOCKS        1024    // Total blocks in our RAM disk (512KB total)
#define DISK_SIZE           (TOTAL_BLOCKS * BLOCK_SIZE)

#define MAX_FILES           32      // Maximum number of files in the root directory
#define FILENAME_MAX_LEN    16      // Maximum length of a filename

#define FAT_EOF             0xFFFFFFFF // End of file marker in FAT
#define FAT_FREE            0x00000000 // Free block marker in FAT

// Our simulated RAM disk
extern uint8_t ram_disk[DISK_SIZE];

// Superblock structure (stores overall filesystem info)
typedef struct {
    uint32_t total_blocks;
    uint32_t free_blocks;
    uint32_t fat_start_block;       // Start block of FAT
    uint32_t fat_size_blocks;       // Size of FAT in blocks
    uint32_t root_dir_start_block;  // Start block of root directory
    uint32_t root_dir_size_blocks;  // Size of root directory in blocks
    uint33_t data_start_block;      // Start block of data region
} superblock_t;

// Directory Entry structure
typedef struct {
    char filename[FILENAME_MAX_LEN + 1]; // +1 for null terminator
    uint32_t size;                      // Size of the file in bytes
    uint32_t first_block;               // First data block of the file
    bool in_use;                        // Is this directory entry in use?
} dir_entry_t;

// Global filesystem structures
extern superblock_t superblock;
extern uint32_t *fat_table; // Pointer to the FAT in ram_disk
extern dir_entry_t *root_directory; // Pointer to the root directory in ram_disk

// Function prototypes
void fs_init();
int fs_create(const char* filename);
int fs_write(const char* filename, const uint8_t* data, uint32_t size);
int fs_read(const char* filename, uint8_t* buffer, uint32_t size);
int fs_delete(const char* filename);
void fs_list_files();

#endif // FILESYSTEM_H
```

```c
// In a C source file, e.g., filesystem.c

#include "filesystem.h"
#include "screen.h" // For printing
#include <string.h> // For memset, strcmp, strcpy

uint8_t ram_disk[DISK_SIZE];
superblock_t superblock;
uint32_t *fat_table;
dir_entry_t *root_directory;

// Helper to get a pointer to a block in the RAM disk
static uint8_t* get_block_ptr(uint32_t block_num) {
    if (block_num >= TOTAL_BLOCKS) return NULL;
    return &ram_disk[block_num * BLOCK_SIZE];
}

// Helper to find a free block in FAT
static uint32_t alloc_block() {
    for (uint32_t i = superblock.data_start_block; i < TOTAL_BLOCKS; i++) {
        if (fat_table[i] == FAT_FREE) {
            fat_table[i] = FAT_EOF; // Mark as used and end of chain for now
            superblock.free_blocks--;
            return i;
        }
    }
    return 0; // No free block
}

// Helper to free a block in FAT
static void free_block(uint32_t block_num) {
    if (block_num >= superblock.data_start_block && block_num < TOTAL_BLOCKS) {
        if (fat_table[block_num] != FAT_FREE) {
            fat_table[block_num] = FAT_FREE;
            superblock.free_blocks++;
        }
    }
}

// Helper to find a file by name
static int find_file_entry(const char* filename) {
    for (int i = 0; i < MAX_FILES; i++) {
        if (root_directory[i].in_use && strcmp(root_directory[i].filename, filename) == 0) {
            return i;
        }
    }
    return -1; // File not found
}

void fs_init() {
    print_string("Initializing filesystem...\n");
    memset(ram_disk, 0, DISK_SIZE); // Clear the entire RAM disk

    // Setup superblock
    superblock.total_blocks = TOTAL_BLOCKS;
    superblock.free_blocks = TOTAL_BLOCKS;

    // FAT starts immediately after superblock (block 0)
    superblock.fat_start_block = 1; // Superblock takes block 0
    superblock.fat_size_blocks = (sizeof(uint32_t) * TOTAL_BLOCKS + BLOCK_SIZE - 1) / BLOCK_SIZE;
    fat_table = (uint32_t*)get_block_ptr(superblock.fat_start_block);

    // Root directory starts after FAT
    superblock.root_dir_start_block = superblock.fat_start_block + superblock.fat_size_blocks;
    superblock.root_dir_size_blocks = (sizeof(dir_entry_t) * MAX_FILES + BLOCK_SIZE - 1) / BLOCK_SIZE;
    root_directory = (dir_entry_t*)get_block_ptr(superblock.root_dir_start_block);

    // Data blocks start after root directory
    superblock.data_start_block = superblock.root_dir_start_block + superblock.root_dir_size_blocks;

    // Mark system blocks as used in FAT
    for (uint32_t i = 0; i < superblock.data_start_block; i++) {
        fat_table[i] = FAT_EOF; // Mark as reserved/used
        superblock.free_blocks--;
    }

    // Initialize directory entries
    for (int i = 0; i < MAX_FILES; i++) {
        root_directory[i].in_use = false;
    }

    print_string("Filesystem initialized. Free blocks: ");
    print_dec(superblock.free_blocks);
    print_newline();
}

int fs_create(const char* filename) {
    if (strlen(filename) > FILENAME_MAX_LEN) {
        print_string("Error: Filename too long.\n");
        return -1;
    }
    if (find_file_entry(filename) != -1) {
        print_string("Error: File already exists.\n");
        return -1;
    }

    // Find a free directory entry
    int entry_idx = -1;
    for (int i = 0; i < MAX_FILES; i++) {
        if (!root_directory[i].in_use) {
            entry_idx = i;
            break;
        }
    }
    if (entry_idx == -1) {
        print_string("Error: No free directory entries.\n");
        return -1;
    }

    // Allocate first data block
    uint32_t first_block = alloc_block();
    if (first_block == 0) {
        print_string("Error: No free data blocks.\n");
        return -1;
    }

    // Setup directory entry
    dir_entry_t *entry = &root_directory[entry_idx];
    strcpy(entry->filename, filename);
    entry->size = 0;
    entry->first_block = first_block;
    entry->in_use = true;

    print_string("File created: ");
    print_string(filename);
    print_newline();
    return 0;
}

int fs_write(const char* filename, const uint8_t* data, uint32_t size) {
    int entry_idx = find_file_entry(filename);
    if (entry_idx == -1) {
        print_string("Error: File not found for writing.\n");
        return -1;
    }

    dir_entry_t *entry = &root_directory[entry_idx];

    // Free existing blocks if any
    uint32_t current_block = entry->first_block;
    while (current_block != FAT_EOF) {
        uint32_t next_block = fat_table[current_block];
        free_block(current_block);
        current_block = next_block;
    }

    // Re-allocate first block
    entry->first_block = alloc_block();
    if (entry->first_block == 0) {
        print_string("Error: No free blocks for writing.\n");
        return -1;
    }
    current_block = entry->first_block;
    entry->size = size;

    uint32_t bytes_written = 0;
    while (bytes_written < size) {
        uint32_t bytes_to_write_in_block = (size - bytes_written > BLOCK_SIZE) ? BLOCK_SIZE : (size - bytes_written);
        memcpy(get_block_ptr(current_block), data + bytes_written, bytes_to_write_in_block);
        bytes_written += bytes_to_write_in_block;

        if (bytes_written < size) { // Need more blocks
            uint32_t new_block = alloc_block();
            if (new_block == 0) {
                print_string("Error: Out of disk space during write.\n");
                return -1; // Ran out of space
            }
            fat_table[current_block] = new_block;
            current_block = new_block;
        } else {
            fat_table[current_block] = FAT_EOF; // End of file
        }
    }

    print_string("Wrote "); print_dec(size); print_string(" bytes to "); print_string(filename); print_newline();
    return 0;
}

int fs_read(const char* filename, uint8_t* buffer, uint32_t size) {
    int entry_idx = find_file_entry(filename);
    if (entry_idx == -1) {
        print_string("Error: File not found for reading.\n");
        return -1;
    }

    dir_entry_t *entry = &root_directory[entry_idx];
    if (size > entry->size) {
        size = entry->size; // Don't read past actual file size
    }

    uint32_t current_block = entry->first_block;
    uint32_t bytes_read = 0;

    while (bytes_read < size && current_block != FAT_EOF) {
        uint32_t bytes_to_read_in_block = (size - bytes_read > BLOCK_SIZE) ? BLOCK_SIZE : (size - bytes_read);
        memcpy(buffer + bytes_read, get_block_ptr(current_block), bytes_to_read_in_block);
        bytes_read += bytes_to_read_in_block;
        current_block = fat_table[current_block];
    }

    print_string("Read "); print_dec(bytes_read); print_string(" bytes from "); print_string(filename); print_newline();
    return bytes_read;
}

int fs_delete(const char* filename) {
    int entry_idx = find_file_entry(filename);
    if (entry_idx == -1) {
        print_string("Error: File not found for deletion.\n");
        return -1;
    }

    dir_entry_t *entry = &root_directory[entry_idx];

    // Free all data blocks associated with the file
    uint32_t current_block = entry->first_block;
    while (current_block != FAT_EOF) {
        uint32_t next_block = fat_table[current_block];
        free_block(current_block);
        current_block = next_block;
    }

    // Clear the directory entry
    memset(entry, 0, sizeof(dir_entry_t));
    entry->in_use = false;

    print_string("File deleted: ");
    print_string(filename);
    print_newline();
    return 0;
}

void fs_list_files() {
    print_string("\n--- Files on Disk ---\n");
    bool found_files = false;
    for (int i = 0; i < MAX_FILES; i++) {
        if (root_directory[i].in_use) {
            print_string("  - ");
            print_string(root_directory[i].filename);
            print_string(" (Size: ");
            print_dec(root_directory[i].size);
            print_string(" bytes, First Block: ");
            print_dec(root_directory[i].first_block);
            print_string(")\n");
            found_files = true;
        }
    }
    if (!found_files) {
        print_string("  (No files found)\n");
    }
    print_string("---------------------\n");
}
```

## Verification/Testing

To verify your filesystem implementation:

1.  **Update your Kernel (`kernel_main.c`):**
    *   Include `filesystem.h`.
    *   Call `fs_init()` at the beginning.
    *   Perform a sequence of file operations.

    ```c
    // In kernel_main.c

    #include "filesystem.h"
    #include "screen.h"
    // ... other includes

    void kernel_main() {
        // ... (previous setup like paging, IDT, PIC, serial_init, multitasking)

        clear_screen();
        print_string("Osmygod Kernel Booted!\n");

        fs_init();
        fs_list_files();

        // Test file creation and writing
        const char* file1_name = "hello.txt";
        const uint8_t file1_data[] = "Hello, Osmygod filesystem!";
        fs_create(file1_name);
        fs_write(file1_name, file1_data, sizeof(file1_data) - 1); // -1 for null terminator

        const char* file2_name = "longfile.bin";
        uint8_t file2_data[BLOCK_SIZE * 3]; // Data spanning multiple blocks
        for (int i = 0; i < sizeof(file2_data); i++) file2_data[i] = (uint8_t)i;
        fs_create(file2_name);
        fs_write(file2_name, file2_data, sizeof(file2_data));

        fs_list_files();

        // Test file reading
        uint8_t read_buffer[100];
        memset(read_buffer, 0, sizeof(read_buffer));
        int bytes_read = fs_read(file1_name, read_buffer, sizeof(read_buffer));
        if (bytes_read > 0) {
            print_string("Content of "); print_string(file1_name); print_string(": ");
            for (int i = 0; i < bytes_read; i++) print_char((char)read_buffer[i], cursor_col, cursor_row, default_color);
            print_newline();
        }

        // Test file deletion
        fs_delete(file1_name);
        fs_list_files();

        // ... (enable interrupts and start scheduler)

        while(1);
    }
    ```

2.  **Build your Kernel:**
    Ensure your Makefile includes `filesystem.c` in the compilation and linking process.

3.  **Run in QEMU:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img
    ```

4.  **Expected Output:**
    You should see messages indicating:
    *   Filesystem initialization.
    *   File creation (`hello.txt`, `longfile.bin`).
    *   File write operations.
    *   A list of files, showing `hello.txt` and `longfile.bin`.
    *   The content of `hello.txt` printed to the screen.
    *   File deletion of `hello.txt`.
    *   A final file list showing only `longfile.bin`.

    This sequence of outputs will confirm that your basic in-memory filesystem can create, write, read, and delete files, and manage its simulated disk space.

## Further Reading/Resources

*   [OSDev Wiki - Filesystems](https://wiki.osdev.org/Filesystems)
*   [OSDev Wiki - FAT](https://wiki.osdev.org/FAT)