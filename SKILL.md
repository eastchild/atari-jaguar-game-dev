---
name: atari-jaguar-game-dev
description: Generic Atari Jaguar game development skill for bare-metal assembly (no Raptor/U235). Covers the hardware (68000, GPU/Tom, DSP/Jerry, Object Processor, Blitter), the memory map, the JagStudio toolchain (rmac/rln), cartridge layout and boot, the rules the real console enforces but emulators do not (long-only accesses to GPU/DSP memory, registers never to touch, RISC processors never stopped from outside, OLP written by the GPU, object list structure and alignment), the documented GPU/DSP/Blitter/OP hardware bugs and their workarounds, GPU/DSP pipeline scheduling (wait states, interleaving, offline estimation from MAME traces), audio on the DSP, joypad, cartridge EEPROM saves, Atari's game standards, measuring performance on the console (emulators model no bus contention), and testing with MAME, Virtual Jaguar and real hardware (SkunkBoard, diagnostic ROMs). Use before writing or changing Jaguar 68000, GPU, DSP or Blitter code, building a cartridge, optimising a hot loop, or debugging a ROM that works in an emulator but not on the console or runs slower there.
---

# Atari Jaguar Game Development Guide

Bare-metal development: JagStudio's assembler/linker, `JAGUAR.INC` and the `Univ.bin` cartridge
header only. **An emulator is not enough to validate a ROM**: MAME, BigPEmu and Virtual Jaguar
accept things the console does not (see "Real hardware rules"). Test on the console (SkunkBoard,
flash cart) before calling anything done.

## Hardware

| Unit | Clock | Local RAM | Role |
|------|-------|-----------|------|
| 68000 | 13.295 MHz (NTSC) | none | game logic, 16-bit bus |
| GPU (in Tom) | 26.59 MHz | 4 KB SRAM $F03000-$F03FFF | 32-bit RISC: rendering, any heavy work |
| DSP (in Jerry) | 26.59 MHz | 8 KB SRAM $F1B000-$F1CFFF | 32-bit RISC: audio (I2S), 16-bit bus to main RAM |
| Object Processor (Tom) | video clock | none | builds every line from an object list in RAM |
| Blitter (Tom) | | none | fills, copies, lines, rotation, Gouraud/Z (16 bpp) |

Main RAM: 2 MB, 64 bits wide, shared by everyone; DRAM page = 2 KB (a page change costs 3-7
cycles: keep data accessed together in the same page). Bus priority, high to low: refresh, DSP/GPU
at DMA priority and Blitter BUSHI (all forbidden, rule 4), **OP**, DSP, 68000 inside an interrupt
(until it writes INT2), GPU, Blitter, 68000. A GPU that reads main RAM a lot slows the 68000 down
(~30 % measured). ROM is up to 10x slower than DRAM.

### Memory map

```
$000000-$1FFFFF  main DRAM (2 MB)  - $000000-$0003FF 68000 vectors
                                     (the dev-kit stub uses $0000-$3FFF: start programs at $4000)
$800000-$DFFFFF  cartridge ROM (up to 6 MB, 8/16/32-bit wide)
                   $800000-$801FFF security header (Univ.bin), code starts at $802000
$E00000-$E1FFFF  boot ROM (128 KB)
$F00000-$F000FF  Tom video / OP / interrupt registers (16-bit: VMODE, VI, BG, OLP, INT1...)
$F00400-$F007FF  CLUT          $F00800-$F01D9F  line buffers
$F02100-$F0211F  GPU registers (G_FLAGS, G_END, G_PC, G_CTRL, G_HIDATA, G_REMAIN/G_DIVCTRL)
$F02200-$F0229F  Blitter registers
$F03000-$F03FFF  GPU local RAM (4 KB, code + data)
$F10000-$F1003F  Jerry timers / interrupt control (JPIT1-4, J_INT)
$F14000 JOYSTICK, $F14002 JOYBUTS (also CONFIG: bit 4 = 1 NTSC, 0 PAL)
$F1A100-$F1A11F  DSP registers (D_FLAGS, D_END, D_PC, D_CTRL, D_MOD, D_REMAIN, D_MACHI)
$F1A140/$F1A144  PWM DACs: not connected on the console (use I2S)
$F1A148-$F1A154  I2S: L_I2S, R_I2S, SCLK, SMODE
$F1B000-$F1CFFF  DSP local RAM (8 KB)
$F1D000-$F1DFFF  DSP wave-table ROM (8 tables x 128 signed 16-bit samples, one per long:
                 triangle, sine, AM sine, sine+harmonic, chirp, noisy triangle, spike, noise)
```

GPU/Blitter space and the line buffers also appear +$8000 higher as a 32-bit write-only window
for the Blitter (fast code loads in phrase mode); the 68000 cannot use it. DSP space has no such
window.

## Real hardware rules (emulators do NOT enforce these)

1. **The 68000 accesses GPU/DSP local RAM and GPU/DSP/Blitter registers ($F02000-$F07FFF,
   $F1A000-$F1FFFF) with LONG moves only.** These are 32-bit locations behind a latch: the first
   word (low address) is latched, the second commits the long. No `move.w`/`move.b`, no
   `tst`/`cmp.w`/`or.b` on them, and **no `clr.l` and no `move.l ...,-(an)`**: those write the two
   words in reverse order (Atari's documented cases; treat any `-(an)` destination there as
   suspect). Use `moveq #0,dN` + `move.l dN,addr`, and `(an)+` copies. A program that publishes a
   pointer with `move.w d0,SHARED+2` runs in every emulator and hangs on a black screen on the
   console. (Tom's video registers at $F000xx are 16-bit: `move.w` is right there.)
2. **GPU/DSP code accesses its own local RAM with `load`/`store` (32-bit) only**: no
   `loadb`/`loadw`/`storeb`/`storew` there (a byte/word store becomes a 32-bit write and clobbers
   the neighbours). Byte/word tables go to main RAM or are packed in longs.
3. **Never write** MEMCON1/2, CLK1-3, HP, HBB, HBE, HS, HVS, HEQ, VP, VBB, VBE, VS, VEB, VEE: the
   boot ROM sets them. **Never write 0 to VMODE** (video off) and never turn the OP off: to blank
   the screen, point the list at a stop object. Yours: HDB1/2, HDE, VDB, VDE, VI, VMODE (non-zero),
   BG, BORD1/2, OLP, PIT0/1, INT1/2.
4. **Never stop the GPU or DSP from outside**: writing 0 to G_CTRL/D_CTRL from the 68000 is
   unreliable (hardware bug); only a RISC may clear its own GO bit. To stop one, set a semaphore in
   its local RAM (and interrupt it); its code then stops itself. Always 0: DMAEN in G_FLAGS/D_FLAGS
   (the DSP hangs), BUS_HOG in G_CTRL/D_CTRL, BUSHI in B_CMD (nothing may outrank the OP: black
   stripes).
5. A 68000 `move.l` to **main RAM** is two bus writes: a RISC can read it half-written. Keep
   shared counters there below $10000 (high word 0) or use a handshake. Longs in RISC local RAM
   are seen whole (latch, rule 1).
6. JOYSTICK writes: bit 15 enables the J0-J7 outputs, **bit 8 = audio on** (muted after reset: set
   it on every write). Bit 0 read = EEPROM data (random). Never touch $F14800-$F15FFF (GPIO0/1:
   breaks EEPROM access).
7. **OLP**: the boot ROM leaves video running with a stop object as the list, and that list is
   in low RAM: OLP = **$1000** when the cartridge starts (observed in MAME booting through the
   BIOS logo), i.e. inside the variables a program usually clears first. Point OLP at your own
   stop object first thing, before clearing or loading anything into RAM. Atari: never write OLP
   with the 68000 (the OP can read it between the two words): write it from the GPU (startup's
   `set_olp` below), then keep it fixed and update the list in place. When updating in place,
   write so that the list is valid at every step (the OP may be reading it): stop object first,
   then bitmaps, then branches, the head object last, each phrase high long first.
8. Code the 68000 runs often, object lists and displayed bitmaps live in RAM, not ROM (a list or
   bitmap fetched from ROM eats the OP's line time and distorts the display). Never write to
   $800000-$DFFFFF: a dev flash board may accept it, a cartridge does not.
9. Verify rules 1 and 2 mechanically after changes: disassemble the 68000 code (Python `capstone`,
   CS_ARCH_M68K, skipping the GPU/DSP code embedded as data) and list every operand in
   $F02000-$F07FFF / $F1A000-$F1FFFF whose size is not `.l`, any `clr`, and any `-(an)` write there.

## Toolchain (JagStudio)

`bin/rmac.exe` (68000 + GPU + DSP assembler), `bin/rln.exe` (linker), `include/JAGUAR.INC`,
`include/Univ.bin` (universal encrypted cartridge header, 8 KB), `bin/virtualjaguar.exe`,
`bin/jcp.exe` (SkunkBoard upload).

### rmac pitfalls

- Expressions are evaluated LEFT TO RIGHT with no precedence: write `a+(b*c)`, `((x>>3)&$ff)<<24`.
- `(x)+9(a0)` is a syntax error: use a literal displacement.
- Local labels `.l .w .s .b .x .p` clash with size suffixes (errors such as "undefined RISC
  register" or "syntax error; expected symbol" far from the cause): use `.lp`, `.wt`, `.pos`...
- `move.w #$99c5,dN` may be "out of range": use `move.l`.
- `.if`/`.else`/`.endif`; defaults: `.if ^^defined FLAG` / `.else` / `FLAG equ 0` / `.endif`;
  `-dFLAG=1` on the command line. `.if X = 0` is not accepted: use `.if X` / `.else`.
- Sections: `.gpu` / `.dsp` + `.org $F03000` / `$F1B000`, back with `.68000`. Assemble RISC code
  into the 68000 image (`gpu_code:` ... `gpu_code_end:`) and copy it with `move.l` at boot.
- Alignment: `.long` (4), `.phrase` (8), `.dphrase` (16), `.qphrase` (32). RISC code is not a
  segment of its own: align before and after it so the `move.l` copy reads whole longs.
- Macros: `.macro NAME arg` / `\arg` / `.endm`. A macro named like an instruction is silently
  replaced by it (the DSP has a `mirror` instruction: a macro `MIRROR` never runs).
- Check that RISC code fits: `.if gpu_end > TABLES_START` / `.error` / `.endif`.
- Listing: `-l<file>` with no space; a long absolute path can give an empty listing with no
  message: use a relative name. Addresses in a listing are relative to the section's `.org`
  (GPU code listed from 0 = $F03000). To list only the RISC code, assemble a small wrapper
  that includes `jaguar.inc`, your definitions and the GPU/DSP sources.

### Cartridge build

ROM = 4 MB (or 2/6 MB) of $FF: `Univ.bin` at offset 0, entry code at $802000. A small boot stub
there copies the program to RAM and jumps to it; data can stay in ROM:

```assembly
; boot stub, linked at $802000 (rln -a 802000 x x)
boot:   move.w  #$2700,sr
        lea     $3ff0,a7
        lea     $802100,a0          ; program image in ROM
        lea     $4000,a1            ; runs from RAM
        move.l  img_size(pc),d0
        addq.l  #3,d0
        lsr.l   #2,d0
.c:     move.l  (a0)+,(a1)+
        subq.l  #1,d0
        bne.s   .c
        jmp     $4000
img_size: dc.l  0                  ; patched by the build script
```

```python
# rmac -fb -i<inc> -o build/main.o main.s
# rln -z -n -m -a 4000 a00000 x -o build/main.bin build/main.o   (text at $4000, data at $A00000)
rom = bytearray(b'\xff' * 0x400000)
rom[0:len(univ)] = univ                        # $800000: header
rom[0x2000:0x2000 + len(boot)] = boot          # $802000: stub (img_size in its last long)
rom[0x2100:0x2100 + text_len] = img[:text_len] # $802100: program, copied to $4000
rom[0xA00000 - 0x800000:...] = img[text_len:]  # data read in place from ROM
```

The `.j64` is the raw image from $800000; the same file works on a SkunkBoard and flash carts.
NTSC and PAL must work from the same cartridge (read CONFIG bit 4 at boot).

## Startup sequence (68000)

```assembly
start:  move.w  #$2700,sr
        lea     STACK,a7
        move.l  #$00070007,G_END        ; big-endian I/O, pixels, instructions (both halves)
        move.l  #$00070007,D_END
        move.w  #$100,JOYSTICK          ; audio on
        ; 1. OP onto a stop object of its own (the boot ROM's list is about to be overwritten)
        clr.l   MYSTOP                  ; (main RAM: clr.l is fine there)
        move.l  #STOPOBJ,MYSTOP+4
        ;    copy set_olp_img..+(set_olp_end-set_olp) to $F03000 with move.l, then:
        move.l  #MYSTOP,d0
        bsr     gpu_set_olp
        ; 2. no write of 0 to G_CTRL/D_CTRL (rule 4): load code into the idle RISCs
        ; 3. clear RAM, init video (below), copy GPU/DSP code with move.l, start them
        ;    (their init code sets r31, the interrupt stack: undefined at reset)
        ; 4. build the object list, gpu_set_olp, VBlank interrupt, VMODE, then move.w #$2000,sr

; OLP is written by the GPU (rule 7), as Atari's startup does with gSetOLP
set_olp_img:                            ; image inside the 68000 program
        .gpu
        .org    $F03000
set_olp:
        movei   #olp_val,r1
        load    (r1),r0
        movei   #OLP,r2
        store   r0,(r2)                 ; same word-swapped value a 68000 move.l would write
        movei   #G_CTRL,r1
        moveq   #0,r3
        store   r3,(r1)                 ; the GPU stops itself
        nop
        nop
        .long
olp_val: dc.l   0
set_olp_end:
        .68000
gpu_set_olp:                            ; d0 = list address (GPU idle, set_olp loaded)
        swap    d0                      ; OLP is written word-swapped
        move.l  d0,olp_val
        move.l  #set_olp,G_PC
        move.l  #RISCGO,G_CTRL
.wt:    move.l  G_CTRL,d0
        btst    #0,d0
        bne.s   .wt
        rts
```

Once the real GPU program runs, have it write OLP itself on request (a long in its local RAM).

### Video init (Atari's startup formula, JAGUAR.INC constants)

```assembly
        move.w  CONFIG,d0               ; NTSC_WIDTH/HMID/HEIGHT/VMID or PAL_...
        andi.w  #VIDTYPE,d0             ; 0 = PAL
        ; width = *_WIDTH, d2 = *_HMID, d6 = *_VMID, d4 = *_HEIGHT
        ; HDB1 = HDB2 = HMID - WIDTH/2 + 4 ; HDE = (WIDTH/2 - 1) | $400
        ; a_vdb = VMID - HEIGHT ; a_vde = VMID + HEIGHT (half-lines)
        move.w  d5,VDB                  ; a_vdb
        move.w  #$ffff,VDE              ; hardware bug: the OP must run on every line
        move.w  a_vde,d0
        ori.w   #1,d0                   ; VI must be odd
        move.w  d0,VI                   ; VBlank interrupt line
        move.w  #$2c7,VMODE             ; RGB16, CSYNC, BGEN, pixel width 2, video on
                                        ; (Atari's startup: $6C7, pixel width 4, ~320 square px)
```

VMODE pixel width (bits 9-11 = divisor-1) sets the horizontal resolution: divisor 2 ~ 655 pixels
overscanned, 3 ~ 443, 4 ~ 332 (square pixels), 5 ~ 266. Overscan heights: 240 lines NTSC, 288 PAL;
the safe area is about 200 / 240 lines (horizontally: the middle ~266 of 320 at divisor 4).

### VBlank interrupt

```assembly
        move.l  #vbl_isr,LEVEL0         ; vector $100
        move.w  #C_VIDENA,INT1
vbl_isr:
        movem.l d0-d2/a0,-(a7)
        bsr     update_op_list          ; restore what the OP rewrote (see Object Processor)
        ; ... read joypad, show the frame the GPU finished ...
        move.w  #C_VIDCLR|C_VIDENA,INT1 ; acknowledge
        move.w  #0,INT2                 ; always: restores GPU/Blitter bus priority
        movem.l (a7)+,d0-d2/a0
        rte
```

All Tom interrupts (video, GPU, OP stop object, PIT, Jerry) arrive on LEVEL0: with more than one
enabled, read INT1 (pending bits 0-4) to find the source. Keep the ISR short: a long list rebuilt
by the 68000 can overrun the frame; move that work to the GPU.

## Object Processor

The OP runs the list from OLP on every line between VDB and VDE and **writes back** into bitmap
objects: HEIGHT and DATA (phrase 0), REMAINDER (phrase 2 of a scaled bitmap). Restore those every
VBlank (keep a copy of phrase 0 and write it back) or rebuild the list. List in RAM, never ROM.
Alignment: bitmap objects on 16 bytes, scaled bitmaps (3 phrases) on 32 bytes, others on 8.

Mandatory layout (Atari: every list starts with these two branches), with VDE = $FFFF:

```
LIST+0   branch  CC=2 (VC > YPOS)  YPOS = a_vde  -> STOP
LIST+8   branch  CC=1 (VC < YPOS)  YPOS = a_vdb  -> STOP
LIST+16  (pad)   LIST+32 bitmap / scaled bitmap objects ...   last -> STOP
STOP     stop object
```

The two branches keep the OP off the list outside the display (less bus use, and the OP never
reads a list the VBlank ISR is rebuilding). The limit is objects per line: every object costs OP
time (header fetch + data) on each of its lines, all within one line period (~63.5 us NTSC).

Object fields (64-bit phrases, written as two longs, high long first):

```
phrase 0 (all): TYPE 0-2 (0 bitmap, 1 scaled, 2 GPU, 3 branch, 4 stop), YPOS 3-13 (half-lines),
                HEIGHT 14-23, LINK 24-42 (address>>3), DATA 43-63 (address>>3); branch: CC 14-16
phrase 1 (bitmap): XPOS 0-11, DEPTH 12-14 (4 = 16 bpp), PITCH 15-17, DWIDTH 18-27 (phrases),
                IWIDTH 28-37 (phrases), INDEX 38-44, REFLECT 45, RMW 46, TRANS 47, RELEASE 48,
                FIRSTPIX 49-54
phrase 2 (scaled): HSCALE 0-7, VSCALE 8-15, REMAINDER 16-23 (3.5 fixed point: 32 = x1)
```

```assembly
; scaled bitmap at a0 (32-byte aligned), link to d3 = next object
        move.l  fb_addr,d0
        lsr.l   #3,d0
        moveq   #11,d1
        lsl.l   d1,d0                   ; DATA << 11 (high long)
        move.l  d3,d1
        lsr.l   #3,d1
        move.l  d1,d2
        lsr.l   #8,d2
        or.l    d2,d0                   ; LINK high bits
        move.l  d0,(a0)+
        andi.l  #$ff,d1
        ror.l   #8,d1                   ; LINK low 8 bits -> 31..24
        or.l    #(HEIGHT<<14)|SCBITOBJ,d1
        move.w  ypos,d0
        andi.l  #$7ff,d0
        lsl.l   #3,d0
        or.l    d0,d1
        move.l  d1,(a0)+
        move.l  #(IWIDTH>>4),(a0)+      ; phrase 1 high: IWIDTH bits 4-9
        move.l  #XPOS|(4<<12)|(1<<15)|(DWIDTH<<18)|((IWIDTH&15)<<28),(a0)+
        clr.l   (a0)+                   ; phrase 2
        move.l  #(VSCALE<<16)|(VSCALE<<8)|HSCALE,(a0)+  ; remainder = vscale
        ; branch object: high long = LINK>>11 ; low = ((LINK>>3)&$ff)<<24 | CC<<14 | YPOS<<3 | 3
```

OP rules and bugs:

- YPOS is in half-lines and must be even on a non-interlaced display. IWIDTH must be non-zero;
  image lines are whole phrases (pad with colour 0, transparent when TRANS is set).
- Scaled bitmaps: HEIGHT = bitmap height - 1 (SoftRef V10); VSCALE above 7.0 fails (HSCALE goes
  up to 7.1F); REMAINDER starts equal to VSCALE; no scaled objects in RGB24 (distorted unless
  HSCALE = 1.0).
- RMW object followed by another bitmap on the same lines: its last pixel column can be corrupted.
  Pad it with transparent pixels or put an always-false branch object after it.
- RELEASE (bit 48): set it for 1-8 bpp objects (lets other masters use the bus between fetches).
  CLUT writes are unreliable while an object with RELEASE is in the list.
- GPU object (type 2) has no YPOS: it fires on every line it is reached; guard it with branch
  objects. The GPU ISR (interrupt 3) must write OBF quickly to restart the OP.
- Stop object bit 3 enables the stop interrupt (INT1 C_OPENA).

### Colours

RGB16 (VMODE mode 3): **bits 15-11 red, 10-6 blue, 5-0 green** (6-bit green). BG (line buffer
background, VMODE BGEN) uses the same format: red $F800, blue $07C0, green $003F, white $FFFF.
BORD1/BORD2 are 24-bit: BORD1 = green<<8 | red, BORD2 = blue. CRY16 (mode 0, the boot ROM's):
bits 15-12 cyan, 11-8 red (colour index), 7-0 intensity; the Blitter's Gouraud shading works on the
intensity; convert RGB to CRY with a 32K-entry table indexed by 5:5:5 RGB. VMODE VARMOD (bit 8):
pixel bit 0 picks CRY (0) or RGB (1) per pixel. 1-8 bpp objects go through the CLUT ($F00400).

## GPU / DSP programming

- `jr` reaches -16..+15 instructions; farther: `movei #label,rN` + `jump cc,(rN)`.
- The instruction after `jr`/`jump` (delay slot) ALWAYS executes; never put a jump, a `movei` or
  `move pc,rN` there.
- `movei`, `move`, `moveq` do not change flags. `cmp Rs,Rd` computes Rd-Rs, carry = borrow.
- `sh`/`sha`: positive count = right, negative = left. `shlq`/`shrq` take 1-32.
- `load (r14+n)` / `(r15+n)` / `(r14+rn)` indexed forms save registers in loops (n in longs, 1-32,
  two extra cycles).
- `div` is 32/32 unsigned (G_DIVCTRL/D_DIVCTRL = 0; DIV_OFFSET = 16.16), 16-18 cycles in parallel,
  flags unchanged. G_REMAIN can be negative (= remainder - divisor: add the divisor back).
- `loadp`/`storep` (GPU) move a phrase with G_HIDATA holding the high long.
- Minimise external accesses: keep tables in local RAM, read main RAM in longs or phrases.
  Schedule every hot loop for the pipeline (next section).
- 68000 <-> RISC: shared longs in local RAM (rule 1), or a ring buffer in main RAM written by the
  68000 and consumed by the RISC, with head/tail longs in local RAM.
- Start: `move.l #entry,G_PC` then `move.l #RISCGO,G_CTRL` (DSP: D_PC, DSPGO); write the PC only
  while stopped. A RISC services interrupts only while running (GO bit set): a stopped RISC takes
  none. Idling it in a loop is fine: a DSP main loop reduced to `idle: jr t,idle` / `nop`, all the
  work done in its I2S interrupt, runs correctly on the console (verified on hardware; no
  document of this set forbids a one-instruction loop). Use a semaphore in local RAM when the
  idle code must be asked to do something (or to stop itself, rule 4).
- Interrupt vectors at local RAM base + 16*n. GPU: 0 CPU, 1 Jerry, 2 video timing, 3 OP (GPU
  object), 4 Blitter. DSP: 0 CPU, 1 I2S, 2 timer 1, 3 timer 2, 4-5 external (enable/clear for 5 =
  D_FLAGS bits 16/17). The interrupt pushes on r31 and clobbers r30 (bank 0): never use them in
  code that runs with interrupts on; init r31 to the top of local RAM.
- DSP vs GPU: no `loadp`/`storep`/`pack`/`unpack`/`sat8/16/24`; adds `sat16s`, `sat32s` (40-bit
  MAC, high bits in D_MACHI), `addqmod`/`subqmod` (circular 2^n buffers, mask in D_MOD), `mirror`.

### Pipeline, wait states and interleaving (write every hot loop this way)

The GPU/DSP issue one instruction per tick only when nothing waits. Naive code runs at about half
that rate, well-scheduled code at about two thirds (SoftRef V10, "Writing Fast GPU and DSP
Programs"). Stages: decode, read operands, compute, write back. A wait state (one tick) is added:

| Cause | Cost |
|-------|------|
| reading the result of the PREVIOUS instruction (ALU result, internal load) | 1 |
| using the flags set by the previous instruction (`jr cc`/`jump cc`, `addc`, `subc`) | 1 |
| write-back port clash: the result of instruction i is written while instruction i+2 reads its operands; if i+2 reads TWO registers and neither is i's target, it waits | 1 |
| two results to write back at once (ALU + divide/load) | 1 |
| after any indexed store `store rN,(r14+n)`, `(r15+n)`, `(r14+rn)`, `(r15+rn)` | 1 |
| after a taken `jr`/`jump` (local RAM) | 3 |
| reading a `div` result too early; a `div` while the previous one runs | 1-16 |
| reading a register still loading from external memory | several (bus) |
| a load/store while the memory interface is busy (1 internal or 2 external pending max) | variable |

How to write it:

- **Interleave** two independent computations so that consecutive instructions never use the
  same register, while instructions two apart do (the read then also hides the write-back).
- **Two-register readers** (`add`, `and`, `or`, `xor`, `cmp`, indexed `load`, any `store`): place
  them two instructions after the write of one of their operands, or two after a store / a
  one-register instruction (`moveq`, `shlq`, `addq`, `move`...).
- **Flags**: set them at least two instructions before the conditional jump; `addqt`/`subqt`
  (and moves, loads, stores) do not touch the flags, so they can go in between.
- **External loads** (main RAM, ROM): issue them early, ideally one loop iteration ahead
  (software pipelining), and use the result ten or more instructions later.
- **Jumps**: 3 ticks each; unroll short loops; fill the delay slot with useful work.
- Mask a byte with a register holding $FF (one `and`) rather than `shlq #24` / `shrq #24`.
- **Reordering loads and stores changes memory order**: before moving a load above a store,
  make sure their addresses can never be equal. Classic trap: updating the two halves of a
  pair table (entries (c, x) and (x, c)) interleaved; for x = c both are the same long, and
  loading both before storing either loses one update.
- **Branches cost 3 ticks when taken**: replace short if/else by arithmetic when possible,
  e.g. a conditional flip `v' = flip ? h - 1 - v : v` as `m = (attr << k) sharq 31` (0 or -1),
  `v' = (v ^ m) + (h & m)`; a conditional shift as `sh` by a computed count (0 or 2).
- Verify each rescheduled routine with a bit-exact output comparison (framebuffer dumps,
  recorded audio): the scheduling rules never change results, mistakes do.

Example (from an AtariAge review): a dependency chain, three wait states before the branch:

```assembly
        shlq    #24,r2          ; \
        shrq    #24,r2          ;  waits for shlq
        cmp     r2,r1           ;  waits for shrq
        jr      eq,.x           ;  waits for the flags of cmp
        nop
; better: r9 = $FF loaded once outside the loop, independent work interleaved
        and     r9,r2           ; one instruction instead of two
        addqt   #4,r14          ; (independent work: other registers, no flags)
        cmp     r2,r1           ; reads r2 written two instructions before: no wait
        addqt   #2,r5           ; (independent, flags untouched)
        jr      eq,.x           ; flags set two instructions before: no wait
        nop
```

**Emulators do not model any of this** (MAME counts one cycle per instruction, plus a few per
taken jump): a GPU loop measured fast in MAME can be twice as slow on the console. Estimate
offline: record a MAME instruction trace of the GPU (`debugger:command("trace file,gpu")`), then
replay it with the rules above, per label (a ~150-line Python script: decode bits 15-10 opcode,
9-5 reg1 = source / address / immediate, 4-0 reg2 = destination / store data; `movei` is followed
by the 32-bit value, low word first). Optimise the labels where most cycles are lost, and check
the output stays bit-identical.

### GPU/DSP hardware bugs

- `jr`/`jump` only work reliably in local RAM: code that branches must run there.
- An indexed `store rN,(r14+n)` of a long-latency result (div, external load) is not
  scoreboarded: put `or rN,rN` before it.
- "Write-only" instructions (`move*`, `load*`, `resmac`, `mtoi`, `normi`) can overtake a pending
  div/load to the same register, or the instruction just before: read the register first
  (`or rN,rN`). Typical trap: `div r2,r4` in a delay slot followed by `moveq #4,r4`.
- Two divides less than 16 cycles apart, the second using the first quotient: put `or rQ,rQ`
  between them.
- After a store to G_FLAGS/D_FLAGS (bank switch, IMASK, flags), 2 instructions (`nop`) before
  relying on it; 4 after an indexed store (GPU).
- G_HIDATA changes on ANY external load: if a GPU ISR loads from main RAM, the main code cannot
  use `loadp`.
- `imultn`, `imacn`..., `resmac` must be consecutive; no `load`/`store` just before `mmult`; DSP
  `mmult` matrix in the first 4 KB of DSP RAM. `abs` leaves $80000000 unchanged.
- **DSP external writes**: an external `store` must follow an external `load` that has completed
  (`load (r1),r2` / `or r2,r2` / `store r11,(r3)`), otherwise it can fail. Intermittent, easy to
  miss in testing: apply it to every DSP write to main RAM.

### DSP audio (I2S)

```assembly
; 68000: SCLK = 19 -> 26.59 MHz / (2*(19+1)) / 32 = ~20.8 kHz ; SMODE = $15 (internal clock,
; WSEN, falling): long writes
        move.l  #19,SCLK
        move.l  #$15,SMODE
; DSP code at $F1B000: interrupt vectors, 16 bytes each (0 CPU, 1 I2S at $F1B010)
; enable: D_FLAGS |= D_I2SENA (| REGPAGE to run the main code in register bank 1, ISR in bank 0)
; ISR: write L_I2S / R_I2S (16-bit samples, saturate), then return:
        movei   #D_FLAGS,r30
        load    (r30),r29
        bclr    #3,r29                  ; IMASK off
        bset    #10,r29                 ; clear the I2S interrupt latch
        load    (r31),r28               ; return address
        addq    #2,r28
        addq    #4,r31
        jump    t,(r28)
        store   r29,(r30)               ; (delay slot) restore flags
```

- SMODE bits: 0 INTERNAL (Jerry drives the clocks), 1 MODE32, 2 WSEN, 3 RISING, 4 FALLING,
  5 EVERYWORD. One stereo frame = 32 bit clocks. The PWM DACs are not connected: I2S only.
- Jerry timers: JPIT1/JPIT2 = timer 1, JPIT3/JPIT4 = timer 2 (JPIT4 at $F10006); rate = 26.59 MHz
  / ((prescaler+1) * (divider+1)). They interrupt the DSP (2/3) or the 68000 through J_INT +
  INT1 C_JERENA.
- If the main loop does real work, the ISR must give it back its ALU flags: read D_FLAGS at ISR
  entry (not at exit) and restore those.
- Check every register-write handler of a synthesiser with recorded audio: an off-by-N index in a
  dispatch table can silence a whole channel in every emulator without any visible error.

## Blitter

Usually driven by the GPU (registers in its local bus); the 68000 may drive it with `move.l` only
(rule 1). A1 = destination, A2 = source (DSTA2 swaps). Writing B_CMD starts the blit: write it
last. Reading B_CMD gives the status: wait for bit 0 (IDLE) before touching any register.

```assembly
; block copy (phrase mode), pixel size 2^n: n<<3 in A1/A2_FLAGS (16 bpp: 4<<3), X add = phrase (0)
        move.l  #dst,A1_BASE            ; phrase aligned
        move.l  #src,A2_BASE
        move.l  #PIXFLAGS,A1_FLAGS
        move.l  #PIXFLAGS,A2_FLAGS
        moveq   #0,d0
        move.l  d0,A1_PIXEL             ; offset inside the first phrase if unaligned
        move.l  d0,A2_PIXEL
        move.l  d0,A1_CLIP              ; bug: always 0 unless clipping
        move.l  #(1<<16)|NPIX,B_COUNT   ; outer loop 1, inner = pixels (max 32767)
        move.l  #$01800001,B_CMD        ; SRCEN | LFU = source (%1100<<21) (+SRCENX if unaligned)
```

- Phrase mode needs both A1 and A2 in phrase mode, auto-aligns only 8/16/32 bpp; set SRCENX when
  the first write needs two source reads. Smaller pixels: pixel mode, and DSTEN (destination read).
- Rectangles: window width in A*_FLAGS bits 9-14 (4-bit exponent, 2 mantissa bits after an
  implicit 1: 320 = 1.01b x 2^8), step added between lines (phrase mode leaves X on the next
  phrase: reduce the step accordingly). Counters take 1-65536 (0 = 65536).
- Rewrite the pointer registers (A1/A2_PIXEL, A1_FPIXEL) and B_COUNT before every blit.
- Gouraud (GOURD) and Z (GOURZ, ZMODE) work in 16 bpp only; in phrase mode the 4 pixels go in
  B_PATD and the intensity increment is x4 (max 31).
- Bugs: A1_CLIP must be 0 when not clipping (a non-phrase X clips anyway); unaligned 2 bpp blits
  are unreliable (use 1 bpp); SRCSHADE needs GOURZ set; Z buffering with ADDDSEL or SRCSHADE
  corrupts data (do two blits); the A2 Y-add bit is ignored and A1's applies to both; BUSHI = 0.
- Use it for copies, clears and loading GPU code (phrase mode into the +$8000 window): faster
  than the 68000 or a RISC. Start a blit, compute on the GPU, wait for IDLE before the next one.

## Joypad

```assembly
; d0 = pressed buttons (1 = pressed), 4 rows of port 1 (row code low nibble, port 2 = $F)
read_pad:
        move.l  #$f0fffffc,d1
        moveq   #-1,d2
        move.w  #$81fe,JOYSTICK         ; bit 15 outputs on, bit 8 audio on, row 0
        move.l  JOYSTICK,d0
        or.l    d1,d0
        ror.l   #4,d0
        and.l   d0,d2
        move.w  #$81fd,JOYSTICK         ; row 1
        move.l  JOYSTICK,d0
        or.l    d1,d0
        ror.l   #8,d0
        and.l   d0,d2
        move.w  #$81fb,JOYSTICK         ; row 2
        move.l  JOYSTICK,d0
        or.l    d1,d0
        rol.l   #6,d0
        rol.l   #6,d0
        and.l   d0,d2
        move.w  #$81f7,JOYSTICK         ; row 3
        move.l  JOYSTICK,d0
        or.l    d1,d0
        rol.l   #8,d0
        and.l   d0,d2
        not.l   d2                      ; active high
        move.l  d2,d0                   ; bit numbers: JAGUAR.INC (FIRE_A, JOY_UP, KEY_*)
        rts
```

Row codes: port 1 uses the low nibble, port 2 the high nibble mirrored. Do not send palindromic
codes when reading only one port (advanced controllers bank-switch on each row 0).

Matrix (B0/B1 in JOYBUTS, J8-J11 in JOYSTICK, 0 = pressed): row 0 Pause A Up Down Left Right;
row 1 C1 B * 7 4 1; row 2 C2 C 0 8 5 2; row 3 C3 Option # 9 6 3. C2/C3 identify the device
(11 pad or nothing, 10 rotary, 01 bank-switching). Without a Team Tap (detected by socket 3, row 1,
B0 = 0) sockets 1-3 echo socket 0: do not read them.

## Saving (cartridge EEPROM)

Standard carts hold a 128-byte serial EEPROM (64 words, ~100,000 writes) on JOYSTICK bit 0. Use
Atari's EEPROM.S (eeReadWord, eeWriteWord, eeReadBank, eeWriteBank, eeUpdateChecksum,
eeValidateChecksum; needs a valid a7): word 63 is the checksum. Write only changed words then
update the checksum; on a bad checksum or a timeout use defaults; never hang on an EEPROM fault.

## Atari game standards

Pause stops everything and shows "PAUSED"; * + # (any order) returns to the title at once;
Option opens the options; B = primary action, A = secondary, C = third, remappable; port 1 =
player 1; music on by default; a demo mode on the title screen; # + * + Option on the title or
options screen clears the EEPROM.

## Rendering architecture that works

- GPU renders into framebuffers in main RAM (triple buffering: one shown, one published, one being
  drawn); it publishes the finished buffer index in a local-RAM long; the 68000 VBlank ISR reads
  it (long), points the bitmap object at it and tells the GPU which one is shown.
- Feed the GPU through a command log/ring in main RAM; publish the write index with a long write.
- Clears and copies go to the Blitter; 3D: the GPU computes spans, the Blitter draws them.
- Pace the game loop on the VBlank count; skip drawing (not logic) when late.

## Performance on the real console

Emulator speed says little: MAME models no bus contention, no DRAM page or ROM wait states and
none of the RISC wait states (a program at 60 fps in MAME can run at half speed on the console).
Measure on the console:

- **On-screen HUD** (a small bitmap object, debug build), refreshed every 60 VBlanks: game
  frames, drawn frames, % of the time the 68000 waits for the GPU, % of the time it is idle.
  Time a wait without a timer: at its start and end take `VBlank count * lines per frame +
  half-lines since the VBlank line` (VC & $7FF at $F00006; 525 half-lines NTSC, 625 PAL) and
  add the difference to a counter.
- **A build that draws nothing** (GPU idle, no pacing, only the HUD shown): its game-frame rate is
  the 68000's raw speed. Same speed as the normal build: the 68000 is the limit; much faster: the
  rendering's bus traffic starves the 68000 (or the 68000 waits for the GPU).
- **68000 bus starvation**: the OP, the GPU and the Blitter all outrank the 68000 on the bus
  (rule list under Hardware); every GPU or Blitter access to main RAM steals 68000 cycles. Keep
  RISC data in local RAM, read main RAM in longs/phrases, and do not let the 68000 busy-wait on
  RAM: `stop #$2000` until the next interrupt instead of a polling loop (Atari's Cinepak player:
  5-10 % less contention).
- **Count accesses in MAME** to find what loads the bus: read/write taps on main RAM, ROM and
  RISC RAM, per frame (68000 reads from ROM are up to 10x slower than from DRAM).
- **RISC wait states**: estimate them offline from a MAME trace (Pipeline section).

## Testing and debugging

### MAME (`mame jaguar -cart game.j64`)

- Headless: `-video none -sound none -nothrottle -autoboot_script probe.lua -autoboot_delay 0`;
  `-wavwrite out.wav` records audio; `-snapshot_directory`.
- Lua: `emu.register_frame_done`, `manager.machine.devices[":maincpu"].spaces["program"]`
  (`read_u32`, `install_write_tap` also on `:gpu` / `:dsp`), `manager.machine.video:snapshot()`,
  `manager.machine:exit()`. Write-only Tom registers read back their last value in MAME (OLP
  $F00020, VMODE $F00028, BG $F00058): handy for checks. A write tap on $800000-$DFFFFF catches
  writes to ROM (rule 8).
- Install memory taps in the first `register_frame_done` call, not when the script loads: taps
  installed before the machine starts are dropped (counters stay at 0). Keep a reference to the
  returned tap object.
- `-debug -debugger none` + `debugger:command("trace file,maincpu")` (or `,gpu` / `,dsp`) for
  instruction traces; a repeated loop is written once, then `(loops for N instructions)`.
- Video: `-aviwrite name.avi` works with `-video none` (written in `-snapshot_directory`; raw
  RGB + 48 kHz PCM, about 1.5 GB per minute); convert with ffmpeg (`pip install imageio-ffmpeg`
  bundles a binary). The picture inside the AVI frame is the display area, borders included.
- The MAME config can skip the BIOS logo; also test with a fresh `-cfg_directory` to boot through
  it. TV system (NTSC/PAL) is a machine configuration setting: test both.
- Limits: no bus contention, GPU/DSP ~1 cycle per instruction: performance measured there is
  optimistic. Snapshots do not show the BG colour. Word accesses to RISC RAM are accepted, and none
  of the RISC/Blitter hardware bugs above are emulated.

### Virtual Jaguar 2.1.2

- `virtualjaguar.exe -l game.j64`: writes `virtualjaguar.log` in the working directory (register
  writes, final 68000/GPU/DSP state, object list dump) **only on a clean exit** (close the window:
  `Process.CloseMainWindow()`, not a kill). Starts the cart at $802000 without the BIOS.
- Its window can be captured alone with user32 `PrintWindow` (flag 2). It crops wide scaled
  bitmaps.

### Real hardware

- SkunkBoard: `jcp game.j64` (4 MB, or 6 MB / two 4 MB banks on 8 MB boards). Power the console
  off before plugging power to avoid flash damage.
- Black screen with no clue: build a **diagnostic ROM** that sets BG (and keeps BGEN) to a colour
  per boot stage (start, RAM/video done, GPU started, DSP started, game loop, first frame shown)
  and makes the VBlank ISR blink it (black every 32 frames). The last colour shown, steady or
  blinking, says where it stops and whether interrupts still run.
- Black screen checklist: rule 1 (word/byte/clr.l/-(an) on GPU/DSP RAM), RISC byte/word accesses
  to local RAM, RISC stopped from the 68000, OLP/list validity and alignment, VMODE written,
  forbidden registers, interrupts acknowledged (INT1 + INT2), jumps in RISC code run from main RAM.
- Glitches only on the console: the GPU/DSP bug list (scoreboard, divides, DSP external writes),
  Blitter A1_CLIP, CLUT writes with RELEASE objects, too many objects per line.

### Audio checks without listening

Record a WAV (MAME `-wavwrite`) and compare spectra (numpy FFT: dominant peaks and band energies
per window) with a reference recording; a missing channel shows as a missing peak. Also count the
samples at full scale (±32767): more than a fraction of a percent means the DSP mix saturates
(lower the channel or master gain before the final `sat16s`).

## References

- Stephen Moss, Software Reference Manual (Tom & Jerry) V10, Technical Reference V10 (console,
  video timings, controllers), Technical Overview V10, 2021.
- Atari, Hardware Bugs & Warnings (1995): GPU/DSP, Blitter, OP and 68000 bugs listed above.
- Atari developer kit: Appendix (FAQ, programming guidelines, development standards), Libraries
  (startup code, EEPROM library), Workshop (object list examples), MADMAC manual.
- Jaguar Technical Reference Manual rev. 8 (2001): Blitter registers, Jerry, DSP, instruction set.
- JagStudio: `JAGUAR.INC`, samples. MAME, Virtual Jaguar, BigPEmu. AtariAge Jaguar forum.

---
*Last updated: 2026-10-12*
