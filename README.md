# Atari Jaguar Game Development Skill

![AI Assisted](https://img.shields.io/badge/AI-Assisted-success?style=flat-square&logo=robot)

A comprehensive skill guide for Atari Jaguar game development, covering hardware architecture, toolchain setup, coding best practices, and performance optimization techniques for bare-metal game development.

---

## About

This repository contains a **skill** - a specialized knowledge base for developing games on the **Atari Jaguar** console. The Atari Jaguar (1993) features a unique multi-processor architecture that requires specific development approaches.

## Features

- **Hardware Architecture Documentation**: Detailed breakdown of the Jaguar's 5 processors (68000, GPU Tom, DSP Jerry, OP, Blitter)
- **Memory Mapping**: Complete memory layout and addressing guides
- **Toolchain Setup**: Configuration for JagStudio (rmac, rln) and emulators
- **Coding Best Practices**: Processor-specific development guidelines
- **Debugging Workflows**: MAME debugging commands and techniques
- **Performance Optimization**: Bus arbitration, caching strategies, and efficiency tips
- **Complete Examples**: Ready-to-use code snippets for graphics, audio, input handling

## Hardware Overview

The Atari Jaguar features a unique architecture with **no shared memory** between main processors:

| Processor | Role | Clock Speed | Local RAM |
|-----------|------|-------------|-----------|
| **M68000** | Main CPU (Game Logic) | 13.295 MHz | None |
| **GPU (Tom)** | Graphics Processor | 26.59 MHz | 64 KB |
| **DSP (Jerry)** | Digital Signal Processor (Audio) | 26.59 MHz | 8 KB |
| **OP** | Object List Processor | - | None |
| **Blitter** | Memory Operations Accelerator | - | None |

## Prerequisites

### Required Tools

1. **JagStudio** - The primary development environment
   - Download: http://www.retremode.com/jagstudio/
   - Required components: `rmac.exe`, `rln.exe`, `jaguar.inc`, `Univ.bin`

2. **Emulators** for testing:
   - **MAME** - Most accurate, with debugging support
   - **BigPEmu** - Windows-native, good compatibility
   - **Virtual Jaguar** - Cross-platform

3. **Python 3.10+** - For build scripts

## Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/your-username/atari-jaguar-game-dev.git
cd atari-jaguar-game-dev
```

### 2. Set Up JagStudio

Download and install **JagStudio** from http://www.retremode.com/jagstudio/

Update the paths in your build scripts to point to your JagStudio installation:

```python
JS = r'C:\path\to\jagstudio\buildfiles'
RMAC = os.path.join(JS, 'bin', 'rmac.exe')
RLN = os.path.join(JS, 'bin', 'rln.exe')
INC = os.path.join(JS, 'include')
```

### 3. Build Your First Game

Create a simple `main.s` file:

```assembly
    include "jaguar.inc"

    org     $4000

Start:
    ; Initialize system
    move.w  #$200,INTCON
    move.w  #$1,VI
    
    ; Set video mode
    move.l  #$00010000,VMODE
    move.l  #$00140001,HDB
    move.l  #$001E0001,HDE
    move.l  #$00140001,VDB
    move.l  #$00FA0001,VDE
    
    ; Main loop
MainLoop:
    bra.s   MainLoop
```

Build using the provided Python script:

```bash
python build.py
```

This will generate a `game.j64` ROM file in the `output/` directory.

### 4. Test in Emulator

```bash
# Using MAME
mame jaguar -cart output/game.j64

# With debugging
mame jaguar -cart output/game.j64 -debug
```

## Project Structure

```
atari-jaguar-game-dev/
├── SKILL.md                 # Main skill documentation
├── .gitignore               # Git ignore patterns
├── README.md                # This file
├── build.py                 # Example build script
├── source/
│   ├── main.s               # 68000 main code
│   ├── gpu.s                # GPU (Tom) code
│   ├── dsp.s                # DSP (Jerry) code
│   └── hal/
│       └── defs.inc          # Hardware definitions
├── include/
│   └── jaguar.inc           # JagStudio header (from JagStudio)
└── output/
    └── game.j64             # Compiled ROM
```

## Key Development Guidelines

### 68000 Development

- **Use address registers** for repeated memory access
- **Use `movem`** for saving/restoring multiple registers
- **Watch expression evaluation**: rmac evaluates LEFT TO RIGHT with NO precedence
  ```assembly
  ; WRONG: 10+20*3 = 90, not 70
  move.l  #10+20*3,d0
  
  ; CORRECT
  move.l  #10+(20*3),d0
  ```
- **Avoid** local labels named `.s`, `.w`, `.l`, `.b` (conflict with size suffixes)

### GPU (Tom) Development

- **CRITICAL**: GPU local RAM is **LONG-ACCESS ONLY**
  ```assembly
  ; CORRECT
  load    (r1),r2    ; Load long
  store   r2,(r1)    ; Store long
  
  ; WRONG - WILL CRASH
  loadb   (r1),r2
  loadw   (r1),r2
  ```
- **68000 `move.l` to Main RAM = Two Word Writes** - GPU can read torn values
- Use `move.w` for 16-bit shared values or implement handshake mechanism
- **Minimize Main RAM access** (~30% 68000 performance loss)
- Use local caches in GPU RAM

### DSP (Jerry) Development

- **Clock**: 26.59 MHz
- **Local RAM**: 8 KB ($F10000-$F13FFF)
- **Register Banks**: Bank 0 for ISRs, Bank 1 for main code
- **I2S Interface** for audio output

## Memory Management

### Recommended Memory Layout

```
$000000-$0FFFFF : ROM header + boot (64 KB)
$010000-$3FFFFF : Game code and data
$800000-$803FFF : Stack and variables (16 KB)
$804000-$80FFFF : 68000 code (60 KB)
$810000-$83FFFF : General data (192 KB)
$840000-$87FFFF : Framebuffer 0 (128 KB)
$880000-$8BFFFF : Framebuffer 1 (128 KB)
$F03000-$F03FFF : GPU code (4 KB) + data
$F1B000-$F1BFFF : DSP code (1 KB) + data
```

## Debugging

### MAME Debugging Commands

```
break 0x4000         - Set breakpoint at address
break 0x4000,r       - Set read breakpoint
trace on             - Enable instruction tracing
trace off            - Disable instruction tracing
regs                - Show all registers
memory 0x800000,20  - Show 20 bytes of memory at address
step                - Single step (into functions)
nstep               - Single step (over functions)
continue            - Continue execution
```

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Black screen | Video mode not set | Check VMODE, HDE, VDE registers |
| No sound | DSP not running | Verify D_CTRL, D_IMASK settings |
| Slow performance | Too much main RAM access | Cache data in local RAM |
| GPU crash | Byte access to GPU RAM | Use only long (32-bit) accesses |
| 68000 crash | Stack overflow | Check stack usage and size |
| Incorrect colors | Wrong RGB format | Use RGB16 (5-6-5) format |

## Performance Optimization

### Bus Arbitration Priority

**Priority Order**: GPU/OP/Blitter > 68000 > DSP

- GPU accessing Main RAM **starves the 68000** (~30% performance loss)
- **Solution**: Cache frequently accessed data in local processor RAM

### Optimization Tips

1. **68000**: Use address registers, `movem` instructions, appropriate data sizes
2. **GPU**: Use local RAM, batch operations, avoid conditional branches
3. **DSP**: Minimize ISR overhead, use efficient audio mixing algorithms
4. **General**: Profile before optimizing, focus on hot code paths

## Resources

### Official Resources
- **JagStudio**: http://www.retremode.com/jagstudio/
- **MAME**: https://www.mamedev.org/
- **BigPEmu**: https://www.ocm.gg/bigpemu/
- **Virtual Jaguar**: https://github.com/virtualjaguar/virtualjaguar

### Community
- **AtariAge Forum**: http://atariage.com/forums/forum/23-atari-jaguar/
- **Jaguar Wiki**: https://jaguar.wiki/
- **Jaguar Development Discord**: Various communities available

### Documentation
- **Jaguar Technical Reference Manual** (if available)
- **68000 Programming Manual**
- **GPU/DSP Architecture Documents**

## Contributing

Contributions are welcome! Please follow these guidelines:

1. **Fork** the repository
2. Create a **feature branch** (`git checkout -b feature/amazing-feature`)
3. **Commit** your changes (`git commit -m 'Add amazing feature'`)
4. **Push** to the branch (`git push origin feature/amazing-feature`)
5. Open a **Pull Request**

### Contribution Guidelines

- Follow existing code style and formatting
- Add documentation for new features
- Include examples where applicable
- Test your changes thoroughly
- Keep commits atomic and well-described

## License

This project is licensed under the **MIT License** - see the [LICENSE](LICENSE) file for details.

This skill is provided as-is for educational and development purposes. The information is based on analysis of existing Jaguar projects and public documentation.

## Acknowledgments

- Based on analysis of Wacky-Races-Jaguar project
- Thanks to the Jaguar development community
- Special thanks to JagStudio developers

---

*Last updated: 2026-09-30*
*Compatible with any LLM*
