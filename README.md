# Crumar Bit 99 Memory and I/O Analysis

An analysis of the Crumar Bit 99 synthesizer, from the perspective of the CPU. This document lists qualified guesses about the usage of ROM, RAM and I/O addresses inside the Bit 99 synthesizer firmware.

## 1  Internal RAM ( `0x00-0xFF`, on-chip data)

| Addr range  | Meaning                                                           |
| ----------- | ----------------------------------------------------------------- |
| `0x00-0x07` | Register bank 0                                                   |
| `0x08-0x0F` | Register bank 1                                                   |
| `0x10-0x13` | Register bank 2 (partial)                                         |
| `0x20-0x2F` | Bit-addressable work area                                         |
| `0x26-0x2B` | UI/LFO flags & split bits (used heavily in `FUN_0861` & friends)  |
| `0x30-0x34` | Key-scanner queue head/tail                                       |
| `0x35-0x3F` | LED/scanner temporary bytes                                       |
| `0x65-0x66` | 16-bit math buffer used during EG calculations                    |
| `0x82-0x90` | Key-scanner debounced state tables                                |
| `0xFA-0xFA` | MIDI byte count ("RX pending")                                    |
| `0xFB-0xFC` | MIDI ring-buffer R/W pointers                                     |
| `0xFD-0xFE` | 1 kHz system-tick down-counters                                   |
| `0xFF-0xFF` | ISR re-entry guard                                                |

## 2  External RAM ( `0x0000-0x0FFF`, battery-backed)

| Addr range      | Purpose (updated notes)                                                                                     |
| --------------- | ----------------------------------------------------------------------------------------------------------- |
| `0x0030-0x003B` | Per-voice oscillator registers (six voices x 2 bytes) - written every panel scan                            |
| `0x003C-0x003E` | **System-mode & panel flag bits** (bit 5 = "panel busy", bit 6 = "queue full")                              |
| `0x0041-0x0041` | 1 ms software down-counter; decremented in the LED/T0 service loop                                          |
| `0x0042-0x0042` | Low-speed "beat" counter copied from the free-running T1 latch; used by `FUN_18xx` for envelope/LFO pacing  |
| `0x0043-0x004E` | **12-byte modulation-target scratch table** (pairs written during patch load in `FUN_1F81` )                 |
| `0x0059-0x005A` | Additional per-voice parameters (detune etc.)                                                               |
| `0x0067-0x007E` | Current program-RAM mirror for DCO/VCF write routine                                                        |
| `0x009D-0x00BC` | Runtime voice parameter block (envelopes, detune, velocity)                                                 |
| `0x00C7-0x00CF` | **UI/sequencer state bytes** - cleared at boot, updated in front-panel ISR                                  |
| `0x00D0-0x00D6` | Captured 16-bit free-running counter (latched from I/O `0x1007` ) and MIDI working bytes                     |
| `0x00D7-0x00DF` | Serial/MIDI communication buffer (rx & tx)                                                                  |
| `0x00E0-0x00E2` | Tempo map / metronome tick bytes                                                                            |
| `0x0FF0-0x0FF0` | **Global system-flag byte** (bit 0 = "audio-IRQ armed", bit 1 = "LFO wrap")                                 |
| `0x0FF1` | Current LED row index - written in `FUN_3019` just before LED data update                                   |
| `0x0FF4-0x0FF7` | **Main LFO phase accumulators & overflow flags** (read/advanced in `FUN_181C` and `FUN_0EB0` )               |
| `0x0FF8` | Pointer to *active* patch block (set by voice-update routine)                                               |
| `0x0FFA` | Patch-load state machine (0 = idle, 1 / 2 = copy phases)                                                    |
| `0x0FFB-0x0FFF` | **Scratch area used by cassette verify, checksum & LED refresh** ( `0xFFF` read for LED matrix inversion)    |

## 3  I/O Space ( `0x1000-0x10FF`, memory-mapped peripherals)

| Addr     | Function                                                                                     |
| -------- | -------------------------------------------------------------------------------------------- |
| `0x1000` | System bus latch / power-on handshake (written with `00 00 00 40 CE 05` early in `FUN_17A8` ) |
| `0x1001` | 8-bit DAC - pitch-bend & mod-wheel values ( `FUN_3236` )                                       |
| `0x1002` | Voice gate/trigger bits (one per voice)                                                      |
| `0x1003` | Synth address/command selector (first byte of register pair)                                 |
| `0x1004` | 7-segment & front-panel LED row driver (row masks `3F/5F/7F` )                                |
| `0x1005` | Synth *data* register (second byte after writing `0x1003` )                                   |
| `0x1006` | Global 8-bit DAC (master VCF/VCA level)                                                      |
| `0x1007` | 16-bit free-running counter latch - captured to `0x00D5/0x00D6` every ms                     |
| `0x100F` | Per-voice parameter port (two-byte writes during voice update)                               |
| `0x1013` | LED-row strobe pattern register (initialised with `0x36, 0x76, ...)                          |

## 4  Interrupt Vector Table (internal ROM)

| Vector   | Source  | Handler                                  |
| -------- | ------- | ---------------------------------------- |
| `0x0000` | Reset   | `FUN_17A8` - boot & hardware init        |
| `0x0003` | INT0    | `FUN_1A55` - front-panel / key-scan ISR  |
| `0x000B` | Timer-0 | `FUN_3000` - 1 kHz system-tick, LED scan |
| `0x0013` | Timer-1 | `FUN_1A88` - UART baud-rate generator    |
| `0x0023` | Serial  | `FUN_1AB9` - MIDI receive ISR            |

## 5  Fixed ROM lookup tables ( `0x2300-0x25FF` )

**Legend**

✅  = proven by direct code reads/writes  
⚠️  = almost certain, one small unknown remains  
❓  = still speculative – more work needed

| Range | Size | Description | Status |
|-------|------|-------------|--------|
| `2000-20FF` | 256 B | *Unclassified ROM constants.* The patch-initialiser copies blocks directly from here when zeroing memory ( `0x2833-0x2841` ). Tentatively looks like **factory default patch data**. | ❓ |
| `2100-21FF` | 256 B | **Detune-spread / pitch-bend taper.** Routines at `0x283B-0x2C24` preload per-voice detune & slew tables, toggling flag F0 to select *coarse* vs *fine* spreads. One nibble appears to hold a 4-bit DCO-mix mask. :contentReference[oaicite:12]{index=12} | ⚠️ (format decoded, musical intent still being mapped) |
| `2300-237F` | 2 x 64 B | **Osc 1 note-number -> coarse-freq words** (fetched in the voice-update state machine) :contentReference[oaicite:8]{index=8} | ✅ |
| `2380-23FF` | 2 x 64 B | **Osc 2 note-number -> coarse-freq** (identical code path after first table) :contentReference[oaicite:9]{index=9} | ✅ |
| `2400-24FF` | 256 B | **Piece-wise-log envelope/LFO rate curve.** Index is folded & bias-corrected in the EG routine at `0x1974-0x19B2` . :contentReference[oaicite:10]{index=10} | ✅ |
| `2500-25FF` | 256 B | **VCF/VCA exponential DAC lineariser.** A 7-bit control * product -> table -> 8-bit output ( `0x1634-0x1648` ). :contentReference[oaicite:11]{index=11} | ✅ |
