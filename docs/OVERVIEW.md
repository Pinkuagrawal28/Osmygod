# Osmygod - Practice Series for Building Your Own Operating System

Welcome to the Osmygod Practice Series! This comprehensive guide is designed for aspiring system programmers, computer science enthusiasts, and anyone curious about the inner workings of an operating system. Through a hands-on, step-by-step approach, you will embark on a journey to build a foundational operating system from scratch.

## Why Build an OS?

Building an operating system is one of the most challenging yet rewarding endeavors in computer science. It offers unparalleled insights into:

*   **Low-Level Hardware Interaction:** Understand how software directly interfaces with CPU, memory, and peripherals.
*   **Core Computer Architecture:** Gain a deep appreciation for how different components of a computer system work together.
*   **Fundamental OS Concepts:** Grasp essential concepts like memory management, process scheduling, interrupt handling, and file systems by implementing them yourself.
*   **Problem-Solving Skills:** Develop advanced debugging and problem-solving abilities in a resource-constrained environment.

This series aims to demystify the complexities of operating systems, transforming abstract theories into concrete, working code.

## Target Audience

This series is ideal for:

*   Students and professionals with a basic understanding of C/C++ and assembly language.
*   Individuals familiar with computer architecture concepts.
*   Anyone eager to dive deep into system-level programming and kernel development.

No prior experience in OS development is required, but a strong willingness to learn and experiment is crucial.

## Osmygod - Detailed Overview

This document provides a more in-depth overview of the Osmygod practice series, expanding on the concepts introduced in the main `README.md`.

## Project Goals

The primary goal of the Osmygod series is to provide a practical, step-by-step guide for individuals interested in understanding and building an operating system from scratch. We aim to:

- **Demystify OS Concepts:** Break down complex operating system principles into understandable and implementable components.
- **Hands-on Learning:** Encourage active participation through coding exercises and practical examples.
- **Foundational Knowledge:** Equip learners with a solid understanding of low-level programming, computer architecture, and system-level interactions.
- **Community Collaboration:** Foster a collaborative environment where learners can share knowledge, ask questions, and contribute to the project.

## Learning Path

The series is designed to be followed sequentially, with each part building upon the knowledge and code developed in previous sections. It's recommended to complete each part before moving on to the next to ensure a comprehensive understanding.

### Core Topics Covered:

1.  [**Bootstrapping**](1-bootstrapping.md): Understanding the power-on self-test (POST), BIOS/UEFI, and the initial stages of loading an operating system.
2.  [**CPU Modes**](2-cpu-modes.md): Exploring real mode, protected mode, and long mode, and how to transition between them.
3.  [**Memory Management**](3-memory-management.md): Implementing techniques for managing system memory, including paging, segmentation, and virtual memory.
4.  [**Interrupts and Exceptions**](4-interrupts-exceptions.md): Handling hardware interrupts (e.g., keyboard, timer) and CPU exceptions (e.g., page faults, division by zero).
5.  [**Input/Output (I/O)**](5-input-output.md): Interacting with hardware devices through ports and memory-mapped I/O.
6.  [**Process and Thread Management**](6-process-thread-management.md): Creating, scheduling, and managing multiple processes and threads.
7.  [**Filesystems**](7-filesystems.md): Developing a basic understanding of how data is stored and retrieved on persistent storage.
8.  [**System Calls**](8-system-calls.md): Implementing an interface for user-space applications to request services from the kernel.

## How to Use These Guides

Each part of the series will have its own dedicated markdown file within the `docs/` directory. These files will contain:

- **Objectives:** What you will learn and achieve in that particular part.
- **Key Concepts:** Explanations of the theoretical foundations.
- **Step-by-Step Implementation:** Detailed instructions and code snippets to guide you through the coding process.
- **Verification/Testing:** How to test your code and ensure it's working as expected.
- **Further Reading/Resources:** Links to external resources for deeper understanding.

It's highly recommended to actively code along with the guides, experiment with the examples, and try to understand *why* certain things are done the way they are. Don't just copy and paste; strive for comprehension.

## Getting Help and Contributing

If you encounter any issues, have questions, or want to contribute, please refer to the `CONTRIBUTING.md` (to be created) and `README.md` files for guidelines on how to engage with the project community. Your feedback and contributions are invaluable in making this series better for everyone.
