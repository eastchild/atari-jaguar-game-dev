# Atari Jaguar Game Development Skill

![AI Assisted](https://img.shields.io/badge/AI-Assisted-success?style=flat-square&logo=robot)

A skill (knowledge file for AI coding assistants) for bare-metal Atari Jaguar game development in
assembly: hardware, toolchain, cartridge build, the rules the real console enforces but emulators
do not, the documented hardware bugs, and testing on emulators and real hardware.

---

## About

The Atari Jaguar (1993) combines a 68000 with four custom processing units in two chips, Tom and
Jerry. Programs that run fine in MAME, BigPEmu or Virtual Jaguar often hang or glitch on the
console, because the emulators ignore access-size rules and hardware bugs. This skill collects
what an assistant needs to write Jaguar code that works on the real machine.

The whole skill is [SKILL.md](SKILL.md). It targets bare-metal development with JagStudio's
assembler and linker (no Raptor or U235 engine).

## What the skill covers

- **Hardware**: 68000, GPU and Blitter (Tom), DSP (Jerry), Object Processor, memory map, bus
  priorities
- **Real hardware rules**: long-only 68000 accesses to GPU/DSP memory and registers (no `clr.l`,
  no `-(an)`), registers never to write, RISC processors never stopped from outside, OLP written
  by the GPU, mandatory object list structure
- **Hardware bugs and workarounds**: GPU/DSP scoreboard, divide and external-write bugs, Blitter
  and Object Processor bugs (from Atari's "Hardware Bugs & Warnings")
- **Toolchain**: rmac/rln pitfalls, cartridge layout (`Univ.bin`, entry at $802000), build script
- **Startup**: video init, object list, VBlank interrupt
- **GPU/DSP programming**: interrupts, register banks, DSP audio over I2S
- **Pipeline scheduling**: the wait states of the RISC processors (result of the previous
  instruction, flags, register write-back port, indexed stores, jumps, external loads), how to
  interleave instructions, branch-free tricks, and how to estimate the lost cycles offline from a
  MAME trace
- **Blitter**: block copies, phrase mode, Gouraud/Z limits
- **Joypad, cartridge EEPROM saves, Atari game standards**
- **Performance on the console**: why emulator speed is misleading, an on-screen measurement HUD,
  68000 bus starvation
- **Testing**: MAME (headless Lua probes, memory taps, instruction traces, WAV and video capture),
  Virtual Jaguar, SkunkBoard, diagnostic ROMs, audio checks without listening

## Hardware overview

| Unit | Role | Clock | Local RAM |
|------|------|-------|-----------|
| **68000** | game logic, system control | 13.295 MHz | none |
| **GPU** (Tom) | 32-bit RISC: rendering, heavy computation | 26.59 MHz | 4 KB ($F03000-$F03FFF) |
| **DSP** (Jerry) | 32-bit RISC: audio, extra computation | 26.59 MHz | 8 KB ($F1B000-$F1CFFF) |
| **Object Processor** (Tom) | builds each display line from an object list | video clock | none |
| **Blitter** (Tom) | copies, fills, lines, rotation, Gouraud/Z shading | | none |

All units share 2 MB of 64-bit main RAM ($000000-$1FFFFF). Cartridge ROM sits at
$800000-$DFFFFF, with the program starting at $802000 after the 8 KB security header.

## Installation

Clone the repository:

```bash
git clone https://github.com/eastchild/atari-jaguar-game-dev.git
```

**Claude Code**: copy (or link) the folder into your skills directory, so that the file ends up
at `~/.claude/skills/atari-jaguar-game-dev/SKILL.md` (personal) or
`.claude/skills/atari-jaguar-game-dev/SKILL.md` inside a project. Claude loads it when you work on
Jaguar code.

**Other assistants**: give `SKILL.md` to the assistant as context (system prompt, rules file or
attached document).

## Prerequisites for Jaguar development

1. **JagStudio**, for `rmac.exe` (68000/GPU/DSP assembler), `rln.exe` (linker), `JAGUAR.INC` and
   `Univ.bin` (cartridge header): http://www.retremode.com/jagstudio/
2. **Emulators** for quick iteration:
   - **MAME**: scriptable in Lua, debugger, audio capture
   - **BigPEmu**
   - **Virtual Jaguar**
3. **Real hardware** to validate: a Jaguar console with a SkunkBoard or a flash cartridge. An
   emulator alone does not prove that a ROM works.
4. **Python 3.10+** for build scripts (assembling the cartridge image).

## Key rules at a glance

These are the mistakes that most often give a ROM that works in emulators but shows a black screen
on the console. SKILL.md explains each one.

- The 68000 reads and writes GPU/DSP local RAM and GPU/DSP/Blitter registers with `move.l` only.
  No `move.w`/`move.b`, no `clr.l`, no `move.l ...,-(an)`.
- GPU/DSP code uses `load`/`store` (32-bit) only in its own local RAM.
- Never write MEMCON1/2, CLK1-3 or the video timing registers set by the boot ROM; never write 0
  to VMODE.
- Never stop the GPU or DSP by writing 0 to G_CTRL/D_CTRL from the 68000: ask it to stop itself.
- Every object list starts with two branch objects, VDE = $FFFF; bitmap objects are 16-byte
  aligned, scaled bitmaps 32-byte aligned; OLP is written by the GPU.
- `jr`/`jump` only work in GPU/DSP local RAM; mind the documented scoreboard and divide bugs.
- Audio is muted after reset: keep bit 8 set on every JOYSTICK write.
- rmac evaluates expressions left to right, with no operator precedence: `10+20*3` is 90. Write
  `10+(20*3)`.

And the ones that make a ROM much slower on the console than in an emulator:

- A GPU/DSP instruction that reads the result of the previous one waits a cycle; so does a
  conditional jump right after the instruction that set the flags. Interleave two independent
  computations in hot loops: naive code runs at about half speed, scheduled code at two thirds.
- Emulators model neither these wait states nor bus contention: measure speed on the console.

```assembly
; dependency chain: three wait states          ; scheduled: none
        shlq    #24,r2                         ;       and     r9,r2      ; r9 = $FF
        shrq    #24,r2                         ;       addqt   #4,r14     ; independent work
        cmp     r2,r1                          ;       cmp     r2,r1
        jr      eq,.x                          ;       addqt   #2,r5      ; flags untouched
                                               ;       jr      eq,.x
```

## Documentation used

The skill was checked against:

- Stephen Moss, *Software Reference Manual (Tom & Jerry)*, *Technical Reference Manual* and
  *Technical Overview*, version 10 (2021), updated from Atari's documentation
- Atari, *Hardware Bugs & Warnings*, developer kit appendices, libraries and workshop documents
  (1994-1995)
- *Jaguar Technical Reference Manual*, revision 8 (2001)

These documents are not distributed with this repository.

## Resources

- **JagStudio**: http://www.retremode.com/jagstudio/
- **MAME**: https://www.mamedev.org/
- **BigPEmu**: https://www.richwhitehouse.com/jaguar/
- **Virtual Jaguar**: https://github.com/virtualjaguar/virtualjaguar
- **AtariAge Jaguar forum**: https://forums.atariage.com/forum/23-atari-jaguar/

## Contributing

Contributions are welcome, especially rules confirmed on real hardware:

1. **Fork** the repository
2. Create a **feature branch** (`git checkout -b feature/amazing-feature`)
3. **Commit** your changes (`git commit -m 'Add amazing feature'`)
4. **Push** to the branch (`git push origin feature/amazing-feature`)
5. Open a **Pull Request**

Please say where a rule comes from (documentation reference or hardware test) and keep SKILL.md
concise.

## License

This project is licensed under the **MIT License**; see the [LICENSE](LICENSE) file for details.

This skill is provided as-is for educational and development purposes. It is based on public
Jaguar documentation and on the analysis of existing Jaguar projects.

## Acknowledgments

- Thanks to the Jaguar development community
- Special thanks to the JagStudio developers and to Stephen Moss for the updated documentation

---

*Last updated: 2026-10-12*
*Compatible with any LLM*
