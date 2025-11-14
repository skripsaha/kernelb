# BoxOS Kernel - Quick Reference Guide

## Executive Summary

**BoxOS** is a 16.5K LOC x86-64 OS kernel featuring a revolutionary event-driven architecture with zero-syscall design.

| Aspect | Details |
|--------|---------|
| **Architecture** | x86-64 (64-bit Intel/AMD) |
| **Build System** | GNU Make with NASM + GCC |
| **Total Files** | 73 (C, ASM, headers, scripts) |
| **Total LOC** | 16,565 lines of code |
| **Languages** | C + x86-64 Assembly |
| **Key Innovation** | Lock-free event-driven pipeline |
| **Boot Method** | 2-stage bootloader (MBR + Stage2) |

## File Organization Summary

```
src/
├── boot/               2 ASM files (927 LOC)     - 2-stage bootloader
├── kernel/
│   ├── arch/x86-64/   7 files (1400+ LOC)      - CPU architecture (GDT, IDT, PIC)
│   ├── core/          9 files (1500+ LOC)      - Memory & CPU management
│   ├── drivers/       6 files (2200+ LOC)      - Hardware (disk, keyboard, timer, video)
│   ├── entry/         2 files (1000+ LOC)      - Kernel entry & linker script
│   ├── eventdriven/   13 files (8000+ LOC)     - Event-driven system (CORE)
│   ├── main_box/      1 file (180 LOC)         - Kernel main
│   └── shell/         1 file (800 LOC)         - Interactive shell
└── lib/kernel/        4 files (1400+ LOC)      - Kernel libc (printf, string ops)
```

## Boot Process

1. **Stage1 (0x7C00)** - MBR bootloader loads Stage2
2. **Stage2 (0x8000)** - Detects memory (E820), enables A20, loads kernel, enters long mode
3. **Kernel (0x10000)** - 64-bit entry point, zeros BSS, calls kernel_main()
4. **kernel_main()** - Initializes all subsystems

## Initialization Sequence

```
kernel_main()
  1. serial_init()           - Early debug
  2. vga_init()              - Console
  3. enable_fpu()            - FPU
  4. mem_init()              - Heap allocator
  5. e820_set_entries()      - Memory map
  6. pmm_init()              - Physical memory manager
  7. vmm_init()              - Virtual memory manager
  8. gdt_init()              - Segment descriptors
  9. idt_init()              - Interrupt handlers
 10. pic_init()              - Interrupt controller
 11. cpu_detect_topology()   - CPU detection
 12. ata_init()              - ATA disk driver
 13. tagfs_init()            - Tag-based filesystem
 14. keyboard_init()         - PS/2 keyboard
 15. pit_init()              - PIT timer
 16. task_system_init()      - Task management
 17. eventdriven_system_init() - Event pipeline
 18. shell_run()             - Interactive shell
```

## Core Subsystems

### Memory Management
- **E820**: BIOS memory map parsing
- **PMM**: Physical page frame allocation (bitmap-based)
- **VMM**: Virtual memory with 4-level paging (42KB largest file)

### Interrupt/Exception Handling
- **IDT**: 32 exceptions + 16 IRQs
- **GDT**: Kernel/User segments, TSS
- **PIC**: 8259A interrupt controller

### Hardware Drivers
| Driver | Purpose | Size |
|--------|---------|------|
| ATA | IDE/PATA disk I/O | 16.5 KB |
| Keyboard | PS/2 input | 4.5 KB |
| VGA | Text mode display | 1 KB |
| Serial | COM port debug | 2 KB |
| PIT | System timer | 2 KB |

### Event-Driven System (UNIQUE)
- **Receiver**: Validates events, generates IDs
- **Center**: Determines routing path
- **Guide**: Orchestrates routing to decks
- **Decks**: Parallel processing (operations, storage, hardware, network)
- **Execution**: Assembles responses

## Key Entry Points

| Address | Mode | File | Function |
|---------|------|------|----------|
| 0x7C00 | 16-bit | stage1.asm | MBR bootloader |
| 0x8000 | 16-bit → 64-bit | stage2.asm | Advanced bootloader |
| 0x10000 | 64-bit | kernel_entry.asm | `_start()` - BSS init |
| - | - | main.c | `kernel_main()` |

## Assembly Files

| File | Lines | Purpose |
|------|-------|---------|
| stage1.asm | 114 | Load Stage2 from disk |
| stage2.asm | 813 | E820, paging, long mode |
| kernel_entry.asm | 426 | BSS zero, stack setup |
| isr.asm | 150 | Exception/IRQ handlers |
| context_switch.asm | 150 | Task context save/restore |

## Memory Layout

```
Physical:
0x0500          E820 map (2KB)
0x7C00          Stage1 (512B)
0x8000          Stage2 (4.6KB)
0x10000-0x34400 Kernel (160KB)
0x500000        Page tables (16KB)
0x510000        Stack

Virtual:
0x0000000000    User space (128TB)
0xFFFF800000    Kernel space (128TB)
```

## Event-Driven Architecture Flow

```
User Application
    ↓ (no syscall)
[Lock-free Ring Buffer: User→Kernel]
    ↓
RECEIVER (Core 4)
    ├─ Validate
    ├─ Generate ID
    └─ Add timestamp
    ↓
CENTER (Core 5)
    ├─ Determine route (prefixes)
    └─ Store in routing table
    ↓
GUIDE (Core 6)
    ├─ Read routing entries
    └─ Dispatch to decks
    ↓
[Parallel on different cores]
├─ Operations Deck (7)
├─ Storage Deck (8)
├─ Hardware Deck (9)
└─ Network Deck (10)
    ↓
EXECUTION DECK (Core 11)
    ├─ Assemble response
    └─ Write to response ring buffer
    ↓
[Lock-free Ring Buffer: Kernel→User]
    ↓
User Application (polls response)
```

## Build & Run Commands

```bash
# Build
make clean                          # Clean build directory
make all                           # Full build (img, iso, elf, vdi)

# Run
make run                           # QEMU with 512MB RAM
make debug                         # QEMU with GDB (port 1234)
make rununiversal                  # QEMU with 4 cores, 4GB RAM
make runlog                        # QEMU with detailed logging

# System Info
make info                          # Show build information
make install-deps                  # Install build dependencies
```

## Output Formats

- `build/boxos.img` - 10MB raw disk image
- `build/boxos_floppy.img` - 1.44MB floppy
- `build/boxos.iso` - Bootable ISO
- `build/boxos.elf` - ELF with debug symbols
- `build/boxos.vdi` - VirtualBox disk

## Shell Commands

| Command | Function |
|---------|----------|
| help | Show available commands |
| clear | Clear screen |
| echo | Print text |
| ls | List files |
| cat | Read file |
| mkdir | Create directory |
| touch | Create file |
| rm | Delete file |
| create | Create tagged file |
| eye | View file with tags |
| ps | List processes |
| info | System information |
| reboot | Restart system |

## Key Design Patterns

### Lock-Free Ring Buffers
- SPSC (Single Producer Single Consumer)
- Atomic operations only (XADD, CMPXCHG)
- Cache-line aligned (no false sharing)
- Power-of-2 sizing (256 slots)

### Prefix-Based Routing
- Each event has array of prefixes
- Prefix = next deck to process
- Deck clears prefix when done
- Guide detects completion

### Energy-Based Task Priority
- Priority 0-100
- Self-regulating resource allocation
- Adaptive learning from resource usage

## Performance Characteristics

Expected metrics vs traditional syscalls:

| Metric | Event-Driven | Traditional |
|--------|--------------|-------------|
| Latency | 50-100 ns | 1000-2000 ns |
| Throughput | 5M ops/sec | 500K ops/sec |
| Context switches | 0 | 2+ per call |
| CPU overhead | Zero | High |

## Largest Components

1. **tagfs.c** (59KB) - Tag-based filesystem
2. **vmm.c** (42KB) - Virtual memory manager
3. **klib.c** (37KB) - Kernel libc
4. **shell.c** (23.5KB) - Interactive shell
5. **task.c** (22.9KB) - Task system
6. **storage_deck.c** (21KB) - Filesystem processing
7. **stage2.asm** (20.3KB) - Bootloader

## Configuration Files

- **Makefile** - Build system (220 lines)
- **linker.ld** - Linker script (42 lines)
- **CFLAGS** - `-g -m64 -ffreestanding -nostdlib`
- **LDFLAGS** - Binary format, custom linker script
- **ASMFLAGS** - ELF64 output format

## Dependencies

Build-time:
- NASM (x86-64 assembler)
- GCC (C compiler)
- LD (GNU linker)
- Make

Runtime:
- QEMU (emulation)
- VirtualBox (optional)

## Architecture Highlights

- **Zero-syscall design**: Lock-free ring buffers replace syscalls
- **Multi-core aware**: Each pipeline stage on dedicated core
- **Asynchronous**: User space never blocks
- **Parallel processing**: Decks process simultaneously
- **Lock-free IPC**: Atomic operations only
- **Comprehensive**: Complete kernel from bootloader to shell

---

**Note**: This is a sophisticated research/educational kernel demonstrating advanced OS design principles. See `KERNEL_ARCHITECTURE_ANALYSIS.md` for detailed documentation.
