---
name: atari-jaguar-game-dev
description: Generic Atari Jaguar game development skill. Covers Jaguar hardware architecture (68000, GPU Tom, DSP Jerry, OP, Blitter), toolchain setup (JagStudio rmac/rln), memory mapping, coding best practices for each processor, debugging workflows, and performance optimization techniques for bare-metal game development on Atari Jaguar.
---

# Atari Jaguar Game Development Guide

## Overview

Atari Jaguar (1993) features a unique multi-processor architecture with **no shared memory** between main processors. Understanding this architecture is crucial for effective development.

### Hardware Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  M68000 (13.295MHz)    GPU Tom (26.59MHz)    DSP Jerry (26.59MHz) │
│     │                    │                    │                   │
│     └──────────────────┴──────────────────┴───────────────┘ │
│                              Main RAM (2MB)                         │
└─────────────────────────────────────────────────────────────┘
```

| Processor | Role | Local RAM | Characteristics |
|-----------|------|-----------|----------------|
| **68000** | Game Logic, Main CPU | None | 32-bit data, 24-bit address |
| **GPU (Tom)** | Graphics | 64KB | RISC, fast graphics ops |
| **DSP (Jerry)** | Audio | 8KB | RISC, I2S audio interface |
| **OP** | Object List | None | Hardware display list processor |
| **Blitter** | Memory Ops | None | Fast memory moves/fills |

### Memory Map

```
$000000-$3FFFFF  : ROM/Cartridge (up to 4MB)
$800000-$8FFFFF  : Main RAM (2MB)
$F00000-$F03FFF  : GPU local RAM
$F04000-$F07FFF  : GPU registers
$F10000-$F13FFF  : DSP local RAM
$F14000-$F17FFF  : DSP registers
$F18000-$F1BFFF  : OP registers
$F1C000-$F1CFFF  : Blitter registers
$FF0000-$FFFFFF  : System registers
```

## Toolchain

### Required Tools

1. **JagStudio** - http://www.retremode.com/jagstudio/
   - `rmac.exe` - Assembler
   - `rln.exe` - Linker
   - `jaguar.inc` - Header file with registers
   - `Univ.bin` - Standard cartridge header

2. **Emulators**
   - **MAME** - Most accurate, debugging support
   - **BigPEmu** - Windows-native, good compatibility
   - **Virtual Jaguar** - Cross-platform

3. **Python 3.10+** - For build scripts

### Build Script Example

```python
import subprocess, os

JS = r'D:\source_codes\jagstudio\buildfiles'
RMAC = os.path.join(JS, 'bin', 'rmac.exe')
RLN = os.path.join(JS, 'bin', 'rln.exe')
INC = os.path.join(JS, 'include')

TEXT_ADDR = 0x4000
DATA_ADDR = 0xA00000
ROM_BASE = 0x800000
ROM_SIZE = 0x400000

def build():
    os.makedirs('build', exist_ok=True)
    
    # Assemble
    subprocess.run([RMAC, '-fb', '-i' + INC, '-o', 'build/main.o', 'main.s'], check=True)
    
    # Link
    subprocess.run([RLN, '-z', '-n', '-m', '-a', hex(TEXT_ADDR), hex(DATA_ADDR),
                   '-o', 'build/main.bin', 'build/main.o'], check=True)
    
    # Create ROM
    univ = open(os.path.join(INC, 'Univ.bin'), 'rb').read()
    img = open('build/main.bin', 'rb').read()
    
    rom = bytearray(b'\xff' * ROM_SIZE)
    rom[0:len(univ)] = univ
    rom[0x2100:0x2100+len(img)] = img
    
    open('output/game.j64', 'wb').write(rom)
```

## 68000 Development

### Critical rmac Pitfalls

1. **Expression Evaluation**: LEFT TO RIGHT, NO precedence
   ```assembly
   ; WRONG: 10+20*3 = 90, not 70
   move.l  #10+20*3,d0
   
   ; CORRECT
   move.l  #10+(20*3),d0
   ```

2. **Addressing Mode Syntax**
   ```assembly
   ; WRONG
   move.l  (x)+9(a0),d0
   
   ; CORRECT
   move.l  9(a0),d0
   ```

3. **Local Labels**
   ```assembly
   ; AVOID: .s, .w, .l, .b (conflict with size suffixes)
   .label:
   ```

4. **Range Limits**
   ```assembly
   ; WRONG: $99c5 > 32767
   move.w  #$99c5,d0
   
   ; CORRECT
   move.l  #$99c5,d0
   ```

### Memory Access

```assembly
; Use address registers for repeated access
lea     table,a0
move.l (a0)+,d0        ; Post-increment
move.l (a0,d1.w),d2    ; Indexed (sign-extended)

; Use movem for register save/restore
movem.l d0-d2/a0-a2,-(a7)
; ...
movem.l (a7)+,d0-d2/a0-a2
```

### Interrupts

```assembly
vbl_isr:
    movem.l d0-d2/a0-a2,-(a7)
    
    ; Acknowledge
    move.w  #C_VIDCLR|C_VIDENA,INT1
    move.w  #0,INT2
    
    ; Handle VBlank
    bsr     update_sprites
    bsr     read_controllers
    
    movem.l (a7)+,d0-d2/a0-a2
    rte
```

## GPU (Tom) Development

### CRITICAL RULES

1. **GPU Local RAM is LONG-ACCESS ONLY**
   ```assembly
   ; CORRECT
   load    (r1),r2    ; Load long
   store   r2,(r1)    ; Store long
   
   ; WRONG - WILL CRASH
   loadb   (r1),r2
   loadw   (r1),r2
   ```

2. **68000 move.l to Main RAM = Two Word Writes**
   GPU can read a torn value! Use `move.w` for 16-bit shared values or implement handshake.

### GPU/68000 Communication

```assembly
; Shared variables (16-bit)
G_HEAD      equ $F03FE0  ; Write pointer
G_TAIL      equ $F03FE2  ; Read pointer
G_STATUS    equ $F03FE4  ; Status flag

; 68000 side - send command
    move.w  d0,G_HEAD+2  ; Write 16-bit value (safe)

; Handshake for 32-bit values
    move.l  #1,G_UPDATING
    move.l  d0,G_VALUE
    move.l  #0,G_UPDATING
```

### Command Queue Pattern

```assembly
send_gpu_command:
    tst.l   G_STATUS
    bne.s   .wait
    
    move.l  d0,G_CMD
    move.l  d1,G_PARAM1
    move.w  #1,G_STATUS
    rts

; GPU side
process_commands:
    load    (r31+G_STATUS),r1
    cmpq    #0,r1
    beq     .done
    
    store   r0,(r31+G_STATUS)
    load    (r31+G_CMD),r2
    ; ... process ...
.done:
    rts
```

### Performance Tips

- **Minimize Main RAM Access** (~30% 68000 slowdown)
- **Use Local Caches** in GPU RAM
- **Use `loadw`/`load`** instead of `loadb`
- **Batch operations** to minimize bus switching
- **Keep code small** (64KB limit)

## DSP (Jerry) Development

### DSP Architecture

- **Clock**: 26.59 MHz
- **Local RAM**: 8KB ($F10000-$F13FFF)
- **Register Banks**: Bank 0 (ISRs), Bank 1 (main code)
- **I2S Interface**: Audio output

### I2S Setup

```assembly
    movei   #$15,r0        ; SCLK=19, SMODE
    store   r0,D_CTRL
    
    movei   #$40,r0        ; Full volume
    store   r0,D_LEFTVOL
    store   r0,D_RIGHTVOL
```

### DSP/68000 Communication

```assembly
; Shared queue in main RAM
DQHEAD      equ $F1C800  ; Head pointer
DQTAIL      equ $F1C804  ; Tail pointer
DQBUF       equ $F1C900  ; Buffer start

; DSP ISR
    movei   #0,r0
    store   r0,D_FLAGS     ; Clear interrupt
    bsr     dsp_process_queue
    jump    (r28)          ; Return
```

## Memory Management

### Memory Layout Example

```
$000000-$0FFFFF : ROM header + boot (64KB)
$010000-$3FFFFF : Game code and data
$800000-$803FFF : Stack and variables (16KB)
$804000-$80FFFF : 68000 code (60KB)
$810000-$83FFFF : General data (192KB)
$840000-$87FFFF : Framebuffer 0 (128KB)
$880000-$8BFFFF : Framebuffer 1 (128KB)
$F03000-$F03FFF : GPU code (4KB) + data
$F1B000-$F1BFFF : DSP code (1KB) + data
```

### Double Buffering

```assembly
FB0         equ $840000
FB1         equ $860000
FB_PITCH   equ 352

swap_framebuffers:
    move.l  disp_fb,d0
    cmp.l   #FB0,d0
    beq.s   .use_fb1
    
    move.l  #FB1,current_fb
    move.l  #FB0,disp_fb
    bra.s   .update
.use_fb1:
    move.l  #FB0,current_fb
    move.l  #FB1,disp_fb
.update:
    move.l  disp_fb,G_DISP
    rts
```

## Input Handling

### Joypad Reading

```assembly
; Button masks (from jaguar.inc)
JOY_UP      equ $00000001
JOY_DOWN    equ $00000002
JOY_LEFT    equ $00000004
JOY_RIGHT   equ $00000008
JOY_A       equ $00000010
JOY_B       equ $00000020
JOY_C       equ $00000040

read_joypad:
    movem.l d1-d2/a0,-(a7)
    
    move.l  #$f0fffffc,d1
    moveq   #-1,d2
    
    ; Read 4 rows
    move.w  #$81fe,JOYSTICK
    move.l  JOYSTICK,d0
    or.l    d1,d0
    ror.l   #4,d0
    and.l   d0,d2
    
    move.w  #$81fd,JOYSTICK
    move.l  JOYSTICK,d0
    or.l    d1,d0
    ror.l   #8,d0
    and.l   d0,d2
    
    move.w  #$81fb,JOYSTICK
    move.l  JOYSTICK,d0
    or.l    d1,d0
    rol.l   #12,d0
    and.l   d0,d2
    
    move.w  #$81f7,JOYSTICK
    move.l  JOYSTICK,d0
    or.l    d1,d0
    rol.l   #4,d0
    and.l   d0,d2
    
    move.l  d2,d0
    movem.l (a7)+,d1-d2/a0
    rts
```

## Graphics System

### RGB16 Color Format

```
Bit 15-11: Red (5 bits)
Bit 10-5:  Green (6 bits)
Bit 4-0:   Blue (5 bits)
```

### Object List (Display List)

```assembly
; Scaled bitmap object
OPLIST  equ $3000
FB0     equ $840000
FB_PITCH equ 352
FB_WIDTH equ 320
FB_HEIGHT equ 240

build_op_list:
    lea     OPLIST,a0
    
    ; Phrase 0: Link and data pointer
    move.l  #FB0>>3,d0
    moveq   #11,d1
    lsl.l   d1,d0
    move.l  #(OPLIST+32)>>3,d1
    lsr.l   #8,d1
    or.l    d1,d0
    move.l  d0,(a0)+      ; Phrase 0 high
    
    andi.l  #$ff,d1
    ror.l   #8,d1
    move.w  #60,d0        ; Y position
    andi.l  #$7ff,d0
    lsl.l   #3,d0
    or.l    d0,d1
    or.l    #SCBITOBJ,d1
    move.l  d1,(a0)+      ; Phrase 0 low
    
    ; Phrase 1: Dimensions
    move.l  #(FB_WIDTH>>4),(a0)+  ; IWIDTH/16
    move.l  #0|(4<<12)|(1<<15)|(FB_PITCH/8<<18)|((FB_WIDTH&15)<<28),(a0)+
    
    ; Phrase 2: Scaling
    clr.l   (a0)+
    move.l  #48|(48<<8)|(48<<16),(a0)+  ; Scale 1.5x
    
    ; Phrase 3-4: Stop
    clr.l   (a0)+
    clr.l   (a0)+
    move.l  #STOPOBJ,(a0)+  ; Stop object
    rts
```

## Sound System

### Audio Mixing

```assembly
; Channel structure
CH_SIZE    equ 20
CH_SAMPLE  equ 0
CH_POS     equ 4
CH_LEN     equ 8
CH_VOL     equ 12
CH_ACTIVE  equ 16

play_sfx:
    lsl.w   #5,d0       ; Channel * CH_SIZE
    ext.l   d0
    lea     channels,a0
    adda.l  d0,a0
    
    move.l  a1,CH_SAMPLE(a0)
    clr.l   CH_POS(a0)
    move.w  d1,CH_VOL(a0)
    move.b  #1,CH_ACTIVE(a0)
    rts
```

## Performance Optimization

### Bus Arbitration Priority

**Priority**: GPU/OP/Blitter > 68000 > DSP

- GPU main RAM access **starves 68000** (~30% performance loss)
- **Solution**: Cache data in local RAM, minimize GPU external access

### Optimization Tips

1. **68000**: Use address registers, movem, appropriate sizes
2. **GPU**: Use local RAM, batch operations, avoid branches
3. **DSP**: Minimize ISR overhead, use efficient audio algorithms
4. **General**: Profile before optimizing, focus on hot paths

## Debugging

### MAME Debugging

```powershell
mame jaguar -cart game.j64 -debug

# Commands:
# break 0x4000         - Set breakpoint
# trace on             - Enable tracing
# regs                - Show registers
# memory 0x800000,20  - Show memory
# step                - Single step
# continue            - Continue
```

### Logging

```assembly
LOG_BUFFER equ $140000
log_ptr    ds.l 1

log_string:
    movem.l d0-d1/a0-a1,-(a7)
    move.l  log_ptr,a1
    lea     LOG_BUFFER,a2
    adda.l  a1,a2
.copy:
    move.b  (a0)+,(a2)+
    bne.s   .copy
    move.l  a2,d0
    sub.l   #LOG_BUFFER,d0
    move.l  d0,log_ptr
    movem.l (a7)+,d0-d1/a0-a1
    rts
```

## Project Structure

```
my_game/
├── build.py
├── source/
│   ├── main.s      ; Entry point
│   ├── gpu.s       ; GPU code
│   ├── dsp.s       ; DSP code
│   └── hal/
│       └── defs.inc ; Definitions
├── include/
│   └── jaguar.inc   ; From JagStudio
└── output/
    └── game.j64    ; Final ROM
```

## Resources

- **JagStudio**: http://www.retremode.com/jagstudio/
- **MAME**: https://www.mamedev.org/
- **BigPEmu**: https://www.ocm.gg/bigpemu/
- **AtariAge Forum**: http://atariage.com/forums/forum/23-atari-jaguar/
- **Jaguar Wiki**: https://jaguar.wiki/

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Black screen | Video mode not set | Check VMODE, HDE, VDE |
| No sound | DSP not running | Verify D_CTRL, D_IMASK |
| Slow | Too much main RAM access | Cache in local RAM |
| GPU crash | Byte access to GPU RAM | Use only long accesses |
| 68000 crash | Stack overflow | Check stack usage |

---
*Based on analysis of Wacky-Races-Jaguar project*
*Last updated: 2026-09-30*
*Compatible with any LLM*
