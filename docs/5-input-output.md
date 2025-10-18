# 5. Input/Output (I/O)

## Objectives

In this section, you will learn:

*   How the CPU communicates with peripheral devices.
*   The concepts of port-mapped I/O and memory-mapped I/O.
*   Interacting with the keyboard and screen.
*   Basic serial port communication.

## Key Concepts

*   **Port-Mapped I/O (PMIO):** A method where I/O devices are accessed via special I/O instructions (IN/OUT).
*   **Memory-Mapped I/O (MMIO):** A method where I/O devices are accessed by reading from or writing to specific memory addresses.
*   **I/O Ports:** Addresses used for communication with devices in PMIO.
*   **VGA Text Mode:** A simple way to display text on the screen by writing directly to video memory.
*   **Keyboard Controller:** The hardware component that handles keyboard input.
*   **Serial Port:** A common interface for basic communication with external devices or for debugging output.

## Step-by-Step Implementation

This section will guide you through writing to the VGA text buffer to display characters on the screen, reading keyboard scan codes, and implementing basic serial port communication for debugging purposes.

### 1. VGA Text Mode Output (Memory-Mapped I/O)

VGA text mode is a simple way to display text by directly writing to video memory. The text buffer is located at physical address `0xB8000`. Each character on the screen occupies two bytes: one for the ASCII character and one for its attribute (color).

*   **Video Memory Address:** `0xB8000`
*   **Screen Dimensions:** Typically 80 columns x 25 rows.
*   **Character Cell:** 2 bytes per character (ASCII + Attribute).

```c
// In a header file, e.g., screen.h

#ifndef SCREEN_H
#define SCREEN_H

#include <stdint.h>

#define VGA_ADDRESS 0xB8000
#define MAX_COLS    80
#define MAX_ROWS    25

// Color codes
#define VGA_COLOR_BLACK         0x0
#define VGA_COLOR_BLUE          0x1
#define VGA_COLOR_GREEN         0x2
#define VGA_COLOR_CYAN          0x3
#define VGA_COLOR_RED           0x4
#define VGA_COLOR_MAGENTA       0x5
#define VGA_COLOR_BROWN         0x6
#define VGA_COLOR_LIGHT_GREY    0x7
#define VGA_COLOR_DARK_GREY     0x8
#define VGA_COLOR_LIGHT_BLUE    0x9
#define VGA_COLOR_LIGHT_GREEN   0xA
#define VGA_COLOR_LIGHT_CYAN    0xB
#define VGA_COLOR_LIGHT_RED     00xC
#define VGA_COLOR_LIGHT_MAGENTA 0xD
#define VGA_COLOR_LIGHT_BROWN   0xE
#define VGA_COLOR_WHITE         0xF

// Function prototypes
void clear_screen();
void print_char(char c, int col, int row, char color);
void print_string(const char* str);
void print_dec(uint32_t n);
void print_hex(uint32_t n);
void print_newline();

#endif // SCREEN_H
```

```c
// In a C source file, e.g., screen.c

#include "screen.h"

static uint16_t *vga_buffer = (uint16_t *)VGA_ADDRESS;
static int cursor_col = 0;
static int cursor_row = 0;
static char default_color = (VGA_COLOR_BLACK << 4) | VGA_COLOR_LIGHT_GREY; // Black background, light grey foreground

// Function to update the hardware cursor position
static void update_cursor(int col, int row) {
    uint16_t pos = (row * MAX_COLS) + col;

    // Send commands to VGA controller ports
    outb(0x3D4, 0x0F); // Cursor Low Register
    outb(0x3D5, (uint8_t)(pos & 0xFF));
    outb(0x3D4, 0x0E); // Cursor High Register
    outb(0x3D5, (uint8_t)((pos >> 8) & 0xFF));
}

// Function to scroll the screen up by one line
static void scroll_screen() {
    // Move all lines up by one
    for (int i = 1; i < MAX_ROWS; i++) {
        for (int j = 0; j < MAX_COLS; j++) {
            vga_buffer[(i - 1) * MAX_COLS + j] = vga_buffer[i * MAX_COLS + j];
        }
    }
    // Clear the last line
    for (int j = 0; j < MAX_COLS; j++) {
        vga_buffer[(MAX_ROWS - 1) * MAX_COLS + j] = (default_color << 8) | ' ';
    }
    cursor_row = MAX_ROWS - 1;
}

void clear_screen() {
    for (int i = 0; i < MAX_ROWS * MAX_COLS; i++) {
        vga_buffer[i] = (default_color << 8) | ' ';
    }
    cursor_col = 0;
    cursor_row = 0;
    update_cursor(cursor_col, cursor_row);
}

void print_char(char c, int col, int row, char color) {
    if (col >= MAX_COLS || row >= MAX_ROWS) return; // Out of bounds
    vga_buffer[row * MAX_COLS + col] = (color << 8) | c;
}

void print_string(const char* str) {
    while (*str != '\0') {
        if (*str == '\n') {
            print_newline();
        } else {
            if (cursor_col >= MAX_COLS) {
                print_newline();
            }
            vga_buffer[cursor_row * MAX_COLS + cursor_col] = (default_color << 8) | *str;
            cursor_col++;
        }
        str++;
    }
    update_cursor(cursor_col, cursor_row);
}

void print_newline() {
    cursor_col = 0;
    cursor_row++;
    if (cursor_row >= MAX_ROWS) {
        scroll_screen();
    }
    update_cursor(cursor_col, cursor_row);
}

// Helper for printing numbers (decimal)
void print_dec(uint32_t n) {
    if (n == 0) {
        print_string("0");
        return;
    }
    char buf[12]; // Max 10 digits for 32-bit + null terminator
    int i = 0;
    while (n > 0) {
        buf[i++] = (n % 10) + '0';
        n /= 10;
    }
    buf[i] = '\0';
    // Reverse the string
    for (int j = 0; j < i / 2; j++) {
        char temp = buf[j];
        buf[j] = buf[i - 1 - j];
        buf[i - 1 - j] = temp;
    }
    print_string(buf);
}

// Helper for printing numbers (hexadecimal)
void print_hex(uint32_t n) {
    print_string("0x");
    char *hex = "0123456789ABCDEF";
    char buf[9]; // 8 hex digits + null terminator
    buf[8] = '\0';
    for (int i = 7; i >= 0; i--) {
        buf[i] = hex[n % 16];
        n /= 16;
    }
    print_string(buf);
}
```

### 2. PS/2 Keyboard Input (Port-Mapped I/O)

The PS/2 keyboard controller communicates with the CPU via I/O ports `0x60` (data port) and `0x64` (status/command port). When a key is pressed or released, the keyboard sends a scan code to port `0x60`, which then triggers an IRQ1 (interrupt 33 after PIC remapping).

```c
// In your isr_handlers.c (from previous section)

// ... (inside isr_handler function)

    if (regs.int_no == 33) { // Keyboard IRQ1
        uint8_t scancode = inb(0x60); // Read scancode from keyboard data port
        print_string("Keyboard Scancode: ");
        print_hex(scancode);
        print_newline();

        // Basic example: if 'A' is pressed (scancode 0x1E for make code)
        if (scancode == 0x1E) {
            print_string("You pressed 'A'!\n");
        }
        // You would typically have a scancode to ASCII conversion table here
    }

// ... (rest of isr_handler)
```

### 3. Basic Serial Port Communication (Port-Mapped I/O)

The serial port (COM1, typically at I/O port `0x3F8`) is invaluable for debugging an OS, as it allows you to send output to a host machine even if your VGA output isn't working or is complex. It uses a UART (Universal Asynchronous Receiver/Transmitter).

```c
// In a header file, e.g., serial.h

#ifndef SERIAL_H
#define SERIAL_H

#include <stdint.h>

#define COM1_PORT 0x3F8

void serial_init();
int serial_received();
char serial_read();
int serial_is_transmit_empty();
void serial_write(char a);
void serial_print_string(const char* str);

#endif // SERIAL_H
```

```c
// In a C source file, e.g., serial.c

#include "serial.h"

// Helper functions for I/O (from pic.c or a common io.h)
static inline void outb(uint16_t port, uint8_t val) {
    asm volatile("outb %0, %1" : : "a"(val), "Nd"(port));
}

static inline uint8_t inb(uint16_t port) {
    uint8_t ret;
    asm volatile("inb %1, %0" : "=a"(ret) : "Nd"(port));
    return ret;
}

void serial_init() {
    outb(COM1_PORT + 1, 0x00);    // Disable interrupts
    outb(COM1_PORT + 3, 0x80);    // Enable DLAB (set baud rate divisor)
    outb(COM1_PORT + 0, 0x03);    // Set divisor to 3 (lo byte) -> 38400 baud
    outb(COM1_PORT + 1, 0x00);    //                  (hi byte)
    outb(COM1_PORT + 3, 0x03);    // Disable DLAB, 8 bits, no parity, 1 stop bit
    outb(COM1_PORT + 2, 0xC7);    // Enable FIFO, clear, 14-byte threshold
    outb(COM1_PORT + 4, 0x0B);    // IRQs enabled, RTS/DSR set
}

int serial_received() {
    return inb(COM1_PORT + 5) & 1; // Check if data is available
}

char serial_read() {
    while (serial_received() == 0);
    return inb(COM1_PORT);
}

int serial_is_transmit_empty() {
    return inb(COM1_PORT + 5) & 0x20; // Check if transmit buffer is empty
}

void serial_write(char a) {
    while (serial_is_transmit_empty() == 0);
    outb(COM1_PORT, a);
}

void serial_print_string(const char* str) {
    while (*str != '\0') {
        serial_write(*str);
        str++;
    }
}
```

## Verification/Testing

To verify your I/O implementations:

1.  **Update your Kernel (`kernel_main.c`):**
    *   Include `screen.h` and `serial.h`.
    *   Call `clear_screen()` at the beginning.
    *   Call `serial_init()`.
    *   Use `print_string`, `print_dec`, `print_hex`, `print_newline` to display various messages on the VGA screen.
    *   Use `serial_print_string` to send debugging messages over the serial port.

    ```c
    // In kernel_main.c

    #include "screen.h"
    #include "serial.h"
    // ... other includes

    void kernel_main() {
        // ... (previous setup like paging, IDT, PIC)

        clear_screen();
        print_string("Osmygod Kernel Booted!\n");
        print_string("Welcome to the I/O section.\n");
        print_string("Decimal test: "); print_dec(12345); print_newline();
        print_string("Hex test: "); print_hex(0xDEADBEEF); print_newline();

        serial_init();
        serial_print_string("Serial port initialized. Hello from serial!\n");

        // ... (enable interrupts)

        while(1);
    }
    ```

2.  **Build your Kernel:**
    Ensure your Makefile includes `screen.c` and `serial.c` in the compilation and linking process.

3.  **Run in QEMU with Serial Redirection:**
    ```bash
    qemu-system-x86_64 -fda osmygod.img -serial stdio
    ```
    The `-serial stdio` option redirects the first serial port (COM1) to your terminal's standard I/O, so you'll see serial output directly in the terminal where QEMU is running.

4.  **Expected Output:**
    *   **QEMU Window:** You should see the messages "Osmygod Kernel Booted!", "Welcome to the I/O section.", "Decimal test: 12345", and "Hex test: 0xDEADBEEF" displayed on the screen. When you press keys, you should see the keyboard scan codes printed.
    *   **Terminal Output (from QEMU):** You should see "Serial port initialized. Hello from serial!\n" printed in the terminal where you launched QEMU.

If you observe these outputs, your I/O functions are working correctly!

## Further Reading/Resources

*   [OSDev Wiki - I/O Ports](https://wiki.osdev.org/I/O_Ports)
*   [OSDev Wiki - VGA Text Mode](https://wiki.osdev.org/VGA_Text_Mode)
*   [OSDev Wiki - PS/2 Keyboard](https://wiki.osdev.org/PS/2_Keyboard)