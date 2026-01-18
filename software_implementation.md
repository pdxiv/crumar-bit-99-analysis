# Crumar BIT 99 Software Implementation

## Software Architecture Overview

The BIT 99 firmware implements a sophisticated real-time polyphonic synthesizer with comprehensive MIDI support, multiple operating modes, and non-volatile patch storage. The software manages 6-voice polyphony, real-time control voltage generation, user interface, and communication protocols.

**Voice Architecture:** 6-voice polyphonic with dynamic voice allocation

## System Startup Procedure

When the BIT 99 is powered on or reset, the 8051 microcontroller begins execution at address 0x0000, which immediately jumps to the initialization routine at 0x17A8. The startup sequence performs critical hardware and software initialization in the following order:

### 1. Stack Initialization (0x17A8)

```
mov sp,#067H
```

Sets the stack pointer to 0x67, establishing the stack area in internal RAM from 0x68 upward.

### 2. Hardware and NEC µPD71051 USART Initialization (0x19CE)

**8031 Internal Register Setup:**

* `TCON = 0x00`: Clear timer control (timers stopped)
* `TMOD = 0x20`: Configure Timer 1 as 8-bit auto-reload mode for serial baud rate generation
* `SCON = 0x50`: Serial port mode 1 (8-bit UART, variable baud rate)
* `IE = 0x80`: Enable only EA (global interrupt enable) initially
* `IP = 0x14`: Set interrupt priorities (Timer 0 and Serial high priority)
* `TH0 = 0xA0`: Timer 0 reload value (~375Hz interrupt rate for CV updates)
* `TH1 = 0xFF, TL1 = 0xFF`: Timer 1 values for serial baud rate generation

**Port Initialization:**

* `P3 = 0xFF`: All Port 3 pins high (default state)
* `P2 = 0x00`: Address bus high byte cleared
* `P1 = 0xFF`: Port 1 inputs set high (switch matrix)
* `P0 = 0x00`: Address/data bus cleared

**NEC µPD71051 USART Initialization:**

* Set P3.5 (USART chip select) high
* Write initialization sequence to USART at 0x1000 (see [midi_implementation.md](midi_implementation.md) for details)
* Configure: 8 data bits, 1 stop bit, no parity, 31.25 kbaud
* Enable transmit and receive
* Clear P3.5 low to complete USART initialization

### 3. System Memory and Variable Initialization (0x16EE)

**Internal RAM Clearing (0x0FDC):**

* Clear voice state variables (0x08-0x13): Note assignments and voice allocation data
* Clear flag bytes (0x20-0x29): LFO routing, waveform selection, system modes
* Clear first 12 CV storage locations (0x30-0x3B in external RAM)
* Initialize all voice states to "off" with no notes assigned

**Display System Initialization:**

* Disable all Sample & Hold channels (write 0xFF to 0x1004)
* Blank all 8 LED display digits (write 0x1A pattern to buffer 0x67-0x6E)
* Each digit initialized with blank code plus digit index

**Switch Event Buffer Setup:**
The system pre-initializes the switch event buffer to simulate "Lower" and "1" button presses:

* Event count (0xDD) = 0x03 (three events in buffer)
* Read pointer (0xDE) = 0xE0 (start of buffer)
* Write pointer (0xDF) = 0xE3 (after three events)
* Event buffer contents:
  * 0xE0: 0x0D (Lower button press - switch 13)
  * 0xE1: 0x00 (digit "0")
  * 0xE2: 0x01 (digit "1")

This simulates the user selecting "Lower" mode and program "01" at startup, ensuring program 1 loads by default.

**MIDI and Control Initialization:**

* MIDI pitch bend center (0xCC, 0xD5) = 0x80 (center position)
* MIDI modulation wheel (0xCD, 0xD6) = 0x00 (minimum position)
* Pitch bend CV (0xFFF) = 0x80 (center)
* Modulation CV (0xFFE) = 0x00 (off)
* LFO flags (0x26, 0x27) = 0x00 (all LFO routing disabled)
* System flags (0x28) = 0x00 (normal mode, not split/double)
* OMNI mode flag (0x29 bit 0) = 1 (OMNI mode enabled by default)
* MIDI channel (0xCE) = 0x00 (channel 1)
* Operating mode flags (0xFFA, 0xFF9) = 0x00

**Switch Matrix State:**

* Previous column states (0xD0-0xD2) = 0x00
* Switch column control registers cleared

### 4. 8253 Timer Chip Initialization (0x177A)

All four 8253 timer chips (IC35-IC38) are initialized with identical configuration:

**Configuration Loop (for each of 4 chips):**

* Starting at control register 0x1013, 0x1017, 0x101B, 0x101F
* Write three control words per chip:
  * **0x36**: Counter 0 - Mode 3 (square wave), LSB/MSB load, binary counting
  * **0x76**: Counter 1 - Mode 3, LSB/MSB load, binary counting
  * **0xB6**: Counter 2 - Mode 3, LSB/MSB load, binary counting

This configures all 12 DCO timers for square wave generation ready to receive frequency divider values.

### 5. LED and Display Initialization

* LED shadow register (0x3E) = 0x00 (all LEDs off initially)
* Mode tracking variables (0xC7, 0xFF8, 0xD7) = 0x00
* Split point default (0xCF, 0xD8) = 0x18 (MIDI note 24, C1)

### 6. Final Interrupt Enable (0x17B1)

```
mov ie,#097H  ; Enable: EA, ES, EX1, ET0, EX0
setb tr0      ; Start Timer 0 (CV update timer)
setb tr1      ; Start Timer 1 (serial baud rate)
```

Enables all interrupts and starts both timers:

* **EA** (0x80): Global interrupt enable
* **ES** (0x10): Serial port interrupt enable (keybed communication)
* **EX1** (0x08): External interrupt 1 enable (MIDI RX)
* **ET0** (0x02): Timer 0 interrupt enable (CV updates)
* **EX0** (0x01): External interrupt 0 enable (MIDI TX)

### 7. Main Loop Entry (0x17B8)

After initialization completes, the system enters its main execution loop which continuously:

1. Checks keybed RX buffer for key press data (0x0F00)
2. Checks MIDI RX buffer for incoming MIDI messages (0x1B00)
3. Processes switch events from front panel (0x2600)
4. Updates display and system state

The startup procedure takes approximately 5-10 milliseconds to complete, after which the synthesizer is fully operational and responding to MIDI, keybed, and front panel inputs. Timer 0 begins generating interrupts at 325.5Hz (every 3.072ms) to update all control voltages and multiplex the display.

## System Architecture: Interrupt-Driven Real-Time Processing

The BIT 99 uses an interrupt-driven architecture where time-critical tasks execute in Timer 0 ISR, while the main loop handles non-time-critical operations.

### Main Loop (Foreground Processing)

The main loop at 0x17B8 runs continuously when not interrupted, handling:

**1. Keybed Buffer Processing (0x0F00)**

* Checks 256-byte circular buffer at 0x0200-0x02FF
* Processes note on/off messages with velocity
* Updates voice allocation for polyphonic operation
* No timing constraints - processes when CPU available

**2. MIDI Buffer Processing (0x1B00)**  

* Checks 256-byte circular buffer at 0x0100-0x01FF
* Decodes MIDI messages (note on/off, program change, SysEx)
* Updates parameters, loads programs
* Triggered by INT1 interrupt filling buffer
* See [midi_implementation.md](midi_implementation.md) for complete MIDI protocol details

**3. Front Panel Switch Processing (0x2600)**

* Processes switch event circular buffer at 0x00DD-0x00EF
* Updates parameter values from increment/decrement switches
* Handles mode changes (Park, Compare, Address, etc.)
* Switch matrix scanned every 8th interrupt (~24.6ms)

**4. Display and State Updates**

* No direct display writing (handled by interrupt)
* Updates display buffer at 0x0067-0x006E
* Manages system state flags

**Key Point:** Main loop can be interrupted at any time. Tasks here are non-time-critical and will complete "eventually" based on system load.

### Timer 0 Interrupt Handler (0x3000) - Background Processing

Fires every 3.072ms (325.5 Hz) and executes critical real-time tasks:

**Execution Breakdown (from assembly code analysis):**

1. **Display Multiplexing** (~50-100μs)
   * Address: 0x3010-0x3024
   * Increments 8-digit counter (0xFF1)
   * Reads digit data from buffer (0x0067-0x006E)
   * Writes to 7-segment display register (0x1006)
   * Every 8th interrupt sets flag for switch scanning

2. **ADC Reading** (~20μs)
   * Address: 0x302D-0x3045
   * Alternates P3.4 bit: 0=pitch bend, 1=mod wheel
   * Reads ADC result from 0x1007
   * Stores to RAM: pitch bend (0xD5), mod wheel (0xD6)

3. **All 24 S&H Control Voltage Updates** (~716μs total, 23% of interrupt)
   * Address: 0x3045-0x3265
   * Sequential updates in fixed order (assembly code at 0x3045-0x3265):

   **Update Sequence:**
   1. **VC1-VC6** (Filter Cutoff, channels 0x30-0x35) - 6 updates @ 15μs = 90μs
      * Read from RAM 0x30-0x35, write via DAC, enable corresponding S&H

   2. **VEN1-VEN6** (VCA Envelopes, channels 0x36-0x37, 0x28-0x2B) - 6 updates @ 16μs = 96μs
      * Read from RAM 0x36-0x3B, **complement** value (active-low VCA), write via DAC

   3. **VPU1-VPU4** (Pulse Width, channels 0x2C-0x2F) - 4 updates @ 15μs = 60μs
      * Read from RAM 0x93-0x94, 0xBB-0xBC, write via DAC

   4. **VLFO1** (LFO1 Output, channel 0x18) - 1 update @ 175μs
      * **Computed inline** from LFO routing flags and modulation depths (0x315B-0x31A1)
      * Reads parameters at 0xF2, checks bits 030H/034H for LFO1/LFO2→DCO1 enables
      * Multiplies modulation depths, applies scaling/clipping

   5. **VLFO2** (LFO2 Output, channel 0x19) - 1 update @ 175μs
      * **Computed inline** similar to VLFO1 (0x31AD-0x31F3)
      * Checks bits 031H/035H for LFO1/LFO2→DCO2 enables

   6. **VRE1** (Resonance voices 1-3, channel 0x1A) - 1 update @ 15μs
      * Read from RAM 0x7E, write via DAC

   7. **VRE2** (Resonance voices 4-6, channel 0x1B) - 1 update @ 15μs
      * Read from RAM 0xA6, write via DAC

   8. **VDETUNE** (Detune CV, channel 0x1C) - 1 update @ 16μs
      * Read from RAM 0x74, **complement**, write via DAC

   9. **VBEND** (Pitch Bend, channel 0x1D) - 1 update @ 15μs
      * Read from RAM 0xFFF, write via DAC

   10. **VNOISE1** (Noise 1, channel 0x1E) - 1 update @ 15μs
       * Read from RAM 0x76, write via DAC

   11. **VNOISE2** (Noise 2, channel 0x1F) - 1 update @ 15μs
       * Read from RAM 0x9E, write via DAC

   **Total:** 18 simple (270μs) + 6 inverted (96μs) + 2 LFO computed (350μs) = **716μs**

   See hardware_implementation.md for hardware write timing details (machine cycles per channel).

4. **Waveform Register Updates** (~50μs)
   * Address: 0x3236-0x3265
   * Updates registers 0x1001 and 0x1002 from shadows 0x2A/0x2B
   * Applies bit inversion for active-low waveform enables

5. **LFO Processing** (~200-300μs for both LFOs)
   * Address: 0x327D-0x3287
   * Calls L3300 twice (once per LFO)
   * Decrements 16-bit down-counters
   * When counter reaches 0: updates LFO value, reloads from timing table
   * Updates internal RAM locations 0x30-0x31 (current LFO waveform values)

6. **6-Voice Envelope Processing** (~800-1200μs, varies with active voices)
   * Address: 0x3287-0x32A5
   * Calls L2800 six times (once per voice)
   * Each call processes VCF and VCA envelopes for one voice:
     * Decrements envelope phase counters (attack/decay/release)
     * Reads waveform from ROM tables (0x2000-0x21FF)
     * Updates filter cutoff CV (RAM 0x30-0x35) - consumed by S&H update #1
     * Updates VCA envelope CV (RAM 0x36-0x3B) - consumed by S&H update #2
   * Active voices take ~150-200μs each
   * Silent voices process faster (~50μs each)
   * **This is where VC1-VC6 and VEN1-VEN6 values are computed** (used in next interrupt's S&H updates)

7. **Timing Counter Decrements** (~20μs)
   * Address: 0x32A5-0x32CB
   * Decrements various delay/timing counters at 0xFEC-0xFEF

**Total Interrupt Time: ~2000-2400μs (65-78% of 3.072ms period)**

Remaining 700-1100μs available for main loop execution between interrupts.

**Important Timing Note:** Most CV values (22 of 24) are **pre-computed** elsewhere in firmware and stored in RAM. The S&H update sequence reads these pre-computed values and writes them to hardware (15-16μs per channel). Only VLFO1/VLFO2 compute values inline during the S&H update (150-200μs each). This separation allows predictable, fast hardware updates while complex computations happen in dedicated routines.

### Interrupt Priority and Nesting

Priority levels (set by IP register):

* **High Priority**: Timer 0 (CV updates), Serial (keybed)
* **Low Priority**: INT0 (MIDI TX), INT1 (MIDI RX)

High-priority interrupts can interrupt low-priority handlers, ensuring CV updates never miss their deadline.

### Real-Time Performance Characteristics

**Latency Specifications:**

* **Key-to-sound**: <10ms (depends on when keybed interrupt occurs relative to envelope update)
* **MIDI-to-sound**: <10ms (MIDI RX interrupt fills buffer, main loop processes)
* **CV update jitter**: 0μs (all CVs updated every 3.072ms without variation)
* **Envelope resolution**: 3.072ms per counter decrement
* **LFO resolution**: 3.072ms per counter decrement
* **Parameter update**: <25ms (next interrupt after main loop updates RAM)

**Throughput Limits:**

* **Maximum sustained note rate**: ~100 notes/sec (limited by voice allocation processing time)
* **MIDI bandwidth**: 3125 bytes/sec (31.25 kbaud, hardware limited)
* **Switch scan rate**: ~40Hz (every 8th interrupt)
* **Display refresh per digit**: ~40Hz (8 digits × 325.5Hz ÷ 8)

**CPU Loading:**

* **Interrupt overhead**: 65-78% of CPU time
* **Main loop availability**: 22-35% of CPU time
* **Worst case**: All voices active + both LFOs + heavy MIDI traffic = ~78% interrupt + ~15% main loop = 93% total CPU
* **Typical case**: 3-4 voices active = ~70% interrupt + ~20% main loop = 90% total CPU
* **Best case**: No voices active = ~65% interrupt + ~5% main loop = 70% total CPU

The architecture ensures time-critical audio tasks always execute on schedule while allowing flexible processing of user input and MIDI messages in the main loop.

## Program Storage Architecture

The BIT 99 provides **99 user program slots** organized as:

* **Programs 1-75**: Full single programs (37 bytes each)
* **Programs 76-99**: Split/Double programs (7 bytes each)

### Single Program Structure (Programs 1-75)

Each single program consists of 37 bytes storing all synthesizer parameters. **Note: This describes the memory storage format, which is different from the front panel parameter addressing system used for editing.**

```
Offset  Bits  Front Panel Param  Description
------  ----  -----------------  -----------
0x00      6   Parameter 12       Wheel Amount (0-63)
0x01      6   Parameter 11       LFO1 Depth (0-63)
0x02      6   Parameter 10       LFO1 Dynamic Rate (0-63)
0x03      6   Parameter 63       LFO2 Depth (0-63)
0x04      6   Parameter 62       LFO2 Dynamic Rate (0-63)
0x05      8   Parameter 45       Detune (0-255, 128=no detune, (value-128)/2 applied)
0x06      6   Parameter 48       Volume (0-63)
0x07      6   Parameter 34       Noise (0-63)
0x08      6   Parameter 33       DCO1 Dynamic Pulse Width (0-63)
0x09      6   Parameter 44       DCO2 Dynamic Pulse Width (0-63)
0x0A      7   Parameters 24-27   DCO1 Octave/Freq (see note below)
0x0B      7   Parameters 35-38   DCO2 Octave/Freq (see note below)
0x0C      5   Parameter 32       DCO1 Pulse Width (0-31)
0x0D      5   Parameter 43       DCO2 Pulse Width (0-31)
0x0E      6   Parameter 19       VCF Cutoff Frequency (0-63)
0x0F      6   Parameter 20       VCF Resonance (0-63)
0x10      6   Parameter 15       VCF Sustain (0-63)
0x11      6   Parameter 21       VCF Envelope Amount (0-63)
0x12      6   Parameter 18       VCF Keyboard Tracking (0-63)
0x13      6   Parameter 17       VCF Dynamic Attack (0-63)
0x14      6   Parameter 22       VCF Dynamic Envelope (0-63)
0x15      6   Parameter 51       VCA Sustain (0-63)
0x16      6   Parameter 46       VCA Dynamic Attack (0-63)
0x17      6   Parameter 47       VCA Dynamic Volume (0-63)
0x18      6   Parameter 13       VCF Attack (0-63, stored in 8 bits)
0x19      6   Parameter 14       VCF Decay (0-63, stored in 8 bits)
0x1A      6   Parameter 16       VCF Release (0-63, stored in 8 bits)
0x1B      6   Parameter 49       VCA Attack (0-63, stored in 8 bits)
0x1C      6   Parameter 50       VCA Decay (0-63, stored in 8 bits)
0x1D      6   Parameter 52       VCA Release (0-63, stored in 8 bits)
0x1E      6   Parameter 08       LFO1 Delay (0-63, stored in 8 bits)
0x1F      6   Parameter 60       LFO2 Delay (0-63, stored in 8 bits)
0x20      6   Parameter 09       LFO1 Rate (0-63, stored in 8 bits)
0x21      6   Parameter 61       LFO2 Rate (0-63, stored in 8 bits)
0x22      8   Parameters 4-7, 56-59  LFO Flags Byte 1 (see below)
0x23      8   Parameters 1-3, 53-55, 23  LFO Flags Byte 2 (see below)
0x24      8   Parameters 28-30, 39-41    DCO Flags (see below)
```

**DCO Octave/Freq Encoding (Offsets 0x0A, 0x0B):**
These bytes combine octave and frequency settings from the front panel parameters:

* Front panel octave parameters (24-27, 35-38): Only one can be active per DCO
* Front panel frequency parameters (31, 42): 0-11 semitones above octave
* Storage format: `(octave_code * 12) + frequency_semitones`
* Octave codes: 32'=0, 16'=1, 8'=2, 4'=3

### Flag Bit Definitions

**LFO Flags Byte 1 (0x22):**
Maps front panel LFO destination parameters to bit positions:

* Bit 0: LFO1 -> DCO1 (Parameter 4)
* Bit 1: LFO1 -> DCO2 (Parameter 5)
* Bit 2: LFO1 -> VCF (Parameter 6)
* Bit 3: LFO1 -> VCA (Parameter 7)
* Bit 4: LFO2 -> DCO1 (Parameter 56)
* Bit 5: LFO2 -> DCO2 (Parameter 57)
* Bit 6: LFO2 -> VCF (Parameter 58)
* Bit 7: LFO2 -> VCA (Parameter 59)

**LFO Flags Byte 2 (0x23):**

* Bits 1-0: LFO1 waveform (Parameters 1-3: 00=none, 01=triangle, 10=sawtooth, 11=square)
* Bits 3-2: LFO2 waveform (Parameters 53-55: 00=none, 01=triangle, 10=sawtooth, 11=square)
* Bits 6-4: Reserved/unused
* Bit 7: LFO invert flag (Parameter 23)

**DCO Flags (0x24):**
Maps front panel waveform parameters to bit positions. Each DCO has 3 independent waveform enables:

* Bit 0: DCO1 Triangle (Parameter 28)
* Bit 1: DCO1 Sawtooth (Parameter 29)
* Bit 2: DCO1 Pulse (Parameter 30)
* Bit 3: DCO2 Triangle (Parameter 39)
* Bit 4: DCO2 Sawtooth (Parameter 40)
* Bit 5: DCO2 Pulse (Parameter 41)
* Bits 7-6: Reserved/unused

Multiple waveforms can be simultaneously enabled per DCO to create complex combined waveforms, as described in the manual.

### Split/Double Program Structure (Programs 76-99)

Split and Double mode programs use a compressed 7-byte format:

```
Offset  Description
------  -----------
0x00    Lower program number (1-75)
0x01    Upper program number (1-75)  
0x02    Keyboard mode (1=split, 2=double)
0x03    Split point (MIDI note number, split mode only)
0x04    Upper transpose (-36 to +36 semitones)
0x05    Lower program volume/balance
0x06    Upper program volume/balance
```

These programs reference two of the single programs (1-75) and add split/layer configuration data.

## Voice Management System

### Voice Allocation Algorithm

The synthesizer supports 6 voices with dynamic voice allocation. Voice assignment is tracked in internal RAM locations 0x08-0x13, with each voice having a note number and priority value.

**Voice Allocation Strategy:**

* **Last note priority** with voice stealing
* **Voice stealing** using oldest-note priority
* **Dynamic assignment** based on key press order
* **Efficient re-use** of released voices

**Voice State Management:**
Each voice maintains:

* MIDI note number assignment
* Voice priority/age value  
* Envelope state information
* Current control voltage values

**Split Mode Voice Allocation:**

* Lower section: Voices 1-3 (or fewer based on polyphony needs)
* Upper section: Voices 4-6 (or remaining voices)
* Dynamic reallocation based on playing demands

**Double Mode Voice Allocation:**

* Both layers share all 6 voices
* Voice allocation spreads across available voices
* Each key press can trigger multiple voices

## Operating Modes

### 1. Single Mode

* Normal 6-voice polyphonic operation
* All voices available for single program
* Full keyboard range active
* Standard MIDI channel operation

### 2. Split Mode

**Configuration:**

* Keyboard split with independent programs for upper/lower sections
* Split point configurable via front panel or MIDI SysEx
* Independent program selection for each section
* Separate MIDI channel assignment (base channel and base+1)

**Implementation:**

* Lower section: Keys below split point, uses lower program, base MIDI channel
* Upper section: Keys above split point, uses upper program, base+1 MIDI channel  
* Split point stored at memory location 0xCF
* Transpose settings affect upper section independently

### 3. Double Mode

**Configuration:**

* Layer two programs across entire keyboard
* Both programs triggered by all keys
* Same MIDI channel for both layers
* Independent volume/balance control

**Implementation:**

* All keys trigger both upper and lower programs simultaneously
* Voice allocation algorithm distributes voices between layers
* Both layers use same MIDI channel for note reception
* Independent envelope and modulation processing per layer

### 4. Park Mode

**Purpose:**

* Parameter editing without changing the current sound
* Allows auditioning parameter changes before committing
* Temporary editing state that can be discarded

**Implementation:**

* Current program state preserved during editing
* Parameter changes affect temporary buffer only
* Exit Park mode either commits or discards changes
* LED indication shows Park mode status

### 5. Tape Mode

**Purpose:**

* Cassette interface for patch storage and recall
* Essential for patch backup before MIDI SysEx standardization

**Implementation:**

* Activated by front panel "Tape" button (switch 18)
* Patch data serialized into 128-byte buffer (0x0300-0x037F)
* FSK encoding converts digital data to audio tones
* Includes data validation and error detection
* Compatible with standard audio cassette recorders

## MIDI Implementation

The BIT 99 features comprehensive MIDI support including channel voice messages, control changes, program changes, and an extensive System Exclusive implementation for patch management and remote control.

For complete MIDI implementation details, see **[midi_implementation.md](midi_implementation.md)**, which covers:

* Hardware interface and UART configuration
* MIDI channels, OMNI mode, and channel behavior in split/double modes
* Note On/Off, Pitch Bend, and Control Change processing
* Program Change handling (PC 0-98 → Programs 1-99)
* System Exclusive (SysEx) protocol with 10 function codes
* SysEx functions for split/double mode control, program loading, and bank dumps
* Buffer management and reception state machines
* MIDI Implementation Chart
* Latency specifications and real-time performance
* BIT-01 compatibility and multi-device communication

**Quick MIDI Reference:**

* **Channels**: 1-16, selectable via front panel, OMNI mode available
* **Note handling**: 6-voice polyphonic with velocity sensitivity
* **Controllers**: CC1 (mod wheel), CC64 (sustain), CC123 (all notes off), CC124/125 (OMNI mode)
* **Pitch bend**: ±2 semitones (fixed range)
* **Program change**: PC 0-98 (programs 1-99)
* **SysEx**: Manufacturer ID 0x25, device ID format 0x2n (n=channel)

### Switch Event Processing

**Switch Matrix Management:**

* 3-column × 8-row switch matrix
* Circular event buffer at 0x00DD-0x00EF (16 bytes)
* Debouncing and edge detection in software
* Switch codes 1-19 for different front panel functions

**Switch Functions:**

* Switch 11: Address mode
* Switch 12: Compare mode  
* Switch 13: Lower mode
* Switch 14: Park mode
* Switch 15: Upper mode
* Switch 16: Split mode
* Switch 17: Double mode
* Switch 18: Tape operations
* Switch 19: Program Chain mode

## Buffer Management

### Keybed Interface Buffer

* 256-byte circular buffer at 0x0200-0x02FF
* Interrupt-driven reception for low latency
* Buffer pointers at 0xFB (read) and 0xFC (write)
* Byte counter at 0xFA tracks buffer fullness
* Overflow protection discards oldest data when buffer full

**Data Protocol:**
The keybed microcontroller sends a two-byte sequence for each key event:

*Byte 1 - Note Number:*

```
Transmitted Value = Piano Key Number - 12
Range: 0x00-0x7F (for 88-key keyboard)
Main processor adds 0x0C to convert to MIDI note number
Final MIDI range: 0x0C-0x8B (notes 12-139)
```

*Byte 2 - Velocity:*

```
Key Down: 0x01-0x1F (velocity 1-31, scaled to 1-63 for MIDI)
Key Up: 0x00 (note off)
Velocity scaling: MIDI_velocity = (keybed_velocity * 2) + 1
```

**Reception State Machine:**
The main processor uses a flag system to track the two-byte protocol:

1. **Waiting for Note** (flag 0x3D bit 5 clear):
   * Receive note number from keybed
   * Add 0x0C to form MIDI note number
   * Store in location 0xD9 (current key note)
   * Set flag to expect velocity byte next

2. **Waiting for Velocity** (flag 0x3D bit 5 set):
   * Receive velocity value from keybed
   * Store in location 0xDA (current key velocity)
   * Set flag 0x3D bit 4 to trigger note processing
   * Clear velocity flag, return to waiting for note

**Buffer Specifications:**

* 256-byte circular buffer at 0x0200-0x02FF prevents data loss
* Interrupt-driven reception ensures low latency
* Buffer pointers at 0xFB (read) and 0xFC (write)
* Byte counter at 0xFA tracks buffer fullness
* Overflow protection discards oldest data when buffer full

### Switch Event Processing

**Switch Matrix Management:**

* 3-column × 8-row switch matrix
* Circular event buffer at 0x00DD-0x00EF (16 bytes)
* Debouncing and edge detection in software
* Switch codes 1-19 for different front panel functions

**Switch Functions:**

* Switch 11: Address mode
* Switch 12: Compare mode  
* Switch 13: Lower mode
* Switch 14: Park mode
* Switch 15: Upper mode
* Switch 16: Split mode
* Switch 17: Double mode
* Switch 18: Tape operations
* Switch 19: Program Chain mode

## Real-Time Processing

### Voice Processing Algorithm

Each voice undergoes continuous processing for:

1. **Envelope Generation**:
   * Attack, Decay, Sustain, Release phases
   * Separate VCF and VCA envelopes
   * Velocity-sensitive envelope scaling

2. **LFO Processing**:
   * Dual LFOs with multiple waveforms (triangle, sawtooth, square)
   * Configurable routing to DCO, VCF, VCA
   * Delay and rate parameters

3. **Modulation Matrix**:
   * LFO-to-destination routing
   * Keyboard tracking for VCF
   * Velocity sensitivity
   * Pitch bend and mod wheel integration

4. **Control Voltage Calculation**:
   * Per-voice filter cutoff (VC1-VC6)
   * Per-voice amplitude control (VEN1-VEN6)
   * DCO pulse width modulation (VPU1-VPU4)
   * Global modulation sources

### Lookup Table Processing

The firmware uses several lookup tables stored in external ROM for efficient real-time processing:

**Envelope Tables:**

* Rising envelope/Triangle LFO table (0x2000-0x20FF): Attack phases and triangle LFO
* Falling envelope/Sawtooth LFO table (0x2100-0x21FF): Decay/release phases and sawtooth LFO
* Secondary modulation curve (0x2200-0x22FF): Special modulation shapes

**Timing Tables:**

* Primary timer values (0x2300-0x237F): Envelope timing calculations
* Secondary timer values (0x2380-0x23FF): Extended timing ranges (LFO rates)

**Frequency Tables:**

* MIDI note to timer divider (0x2400-0x24F3): DCO frequency generation
* DCO pulse width values (0x2500-0x25FF): Pulse width modulation

## LFO Timing and Frequency Ranges

The BIT 99's two LFOs (Low Frequency Oscillators) use 16-bit down-counters that are decremented at each timer interrupt (325.5 Hz). When a counter reaches zero, the LFO updates to its next value and reloads from the timing table.

### LFO Rate Parameter Timing Table (0x2380-0x23FF)

The LFO Rate parameters (9 and 61, stored at 0xC5 and 0xC6, range 0-63) index into a lookup table containing 64 16-bit timing values. These values determine how many timer interrupts occur between LFO updates.

**LFO Frequency Calculation:**

```
LFO Update Frequency = Timer Interrupt Rate / Counter Value
LFO Update Frequency = 325.5 Hz / Counter Value
```

**Key LFO Timing Values:**

| Parameter | Counter Value | LFO Frequency | Period     | Notes                    |
|-----------|---------------|---------------|------------|--------------------------|
| 0         | 0x0013 (19)   | 17.13 Hz      | 58.4 ms    | Fastest LFO rate         |
| 8         | 0x0062 (98)   | 3.32 Hz       | 301 ms     | Fast audio-rate mod      |
| 16        | 0x0127 (295)  | 1.10 Hz       | 906 ms     | Moderate vibrato         |
| 24        | 0x03D7 (983)  | 0.33 Hz       | 3.02 s     | Slow sweep               |
| 32        | 0x0A1A (2586) | 0.126 Hz      | 7.94 s     | Very slow modulation     |
| 40        | 0x1557 (5463) | 0.060 Hz      | 16.8 s     | Extremely slow           |
| 48        | 0x2666 (9830) | 0.033 Hz      | 30.2 s     | Glacial modulation       |
| 56        | 0x4CCC (19660)| 0.017 Hz      | 60.4 s     | One minute per cycle     |
| 63        | 0x7FFF (32767)| 0.0099 Hz     | 100.7 s    | Slowest: ~1.7 min/cycle  |

**LFO Frequency Range Summary:**

* **Fastest (Parameter 0)**: 17.13 Hz - approaches audio rate, suitable for ring modulation effects
* **Slowest (Parameter 63)**: 0.0099 Hz - extremely slow evolution over nearly 2 minutes per cycle
* **Musical Range (Parameters 8-24)**: 0.33 - 3.32 Hz - typical vibrato and tremolo speeds

### LFO Implementation Details

**Per-Interrupt Processing:**

1. Decrement 16-bit counter for active LFO
2. If counter reaches zero:
   * Lookup next waveform value from shape table (triangle/sawtooth/square)
   * Apply LFO depth parameter scaling
   * Reload counter from timing table (indexed by Rate parameter)
   * Update LFO output value at internal RAM 0x30 (LFO1) or 0x31 (LFO2)

**Waveform Generation:**

* Triangle: Reads from table at 0x2000-0x20FF (256-step rising envelope)
* Sawtooth: Reads from table at 0x2100-0x21FF (256-step falling envelope)  
* Square: Alternates between 0x00 and 0xFF values

**LFO Delay Parameter:**
The LFO Delay parameters (8 and 60, range 0-63) introduce a delay before the LFO begins modulating. The delay time is calculated using the same timing table at 0x2380, providing delays from ~58ms to ~100 seconds.

## Envelope Timing and Attack/Decay/Release Ranges

The BIT 99's envelope generators (for VCF and VCA) use 16-bit down-counters decremented at the timer interrupt rate. The envelope progresses through 256 steps during each phase (attack, decay, release), with timing controlled by values from the lookup table.

### Envelope Timing Table (0x2300-0x237F)

The envelope timing parameters (Attack: 13/49, Decay: 14/50, Release: 16/52, range 0-63) index into a lookup table at 0x2300 containing 64 16-bit timing values. These determine the envelope rate.

**Envelope Time Calculation:**

Each envelope phase progresses through 256 discrete steps. The timing value specifies how many timer interrupts occur per step:

```
Time per Step = Counter Value × 3.072 ms
Total Envelope Phase Time = Counter Value × 3.072 ms × 256 steps
```

**Key Envelope Timing Values:**

| Parameter | Counter Value | Time/Step | Total Phase Time | Notes                 |
|-----------|---------------|-----------|------------------|-----------------------|
| 0         | 0xFFFF (65535)| 201.3 ms  | 51.53 seconds    | Slowest envelope      |
| 8         | 0x4CCC (19660)| 60.4 ms   | 15.46 seconds    | Very slow evolution   |
| 16        | 0x1333 (4915) | 15.1 ms   | 3.86 seconds     | Slow attack/release   |
| 24        | 0x0999 (2457) | 7.55 ms   | 1.93 seconds     | Medium envelope       |
| 32        | 0x04CC (1228) | 3.77 ms   | 0.97 seconds     | Fast attack           |
| 40        | 0x02AA (682)  | 2.09 ms   | 0.54 seconds     | Snappy percussive     |
| 48        | 0x017F (383)  | 1.18 ms   | 0.30 seconds     | Very fast attack      |
| 56        | 0x0093 (147)  | 0.45 ms   | 0.115 seconds    | Near-instant attack   |
| 63        | 0x000B (11)   | 0.034 ms  | 8.6 milliseconds | Fastest possible      |

**Envelope Time Range Summary:**

* **Slowest (Parameter 0)**: 51.5 seconds - extremely long evolving pads
* **Fastest (Parameter 63)**: 8.6 milliseconds - percussive clicks and pops
* **Musical Attack Range (Parameters 40-56)**: 115ms - 540ms - typical synthesizer attacks
* **Musical Release Range (Parameters 24-40)**: 540ms - 1.93s - natural decay tails

### Envelope Implementation Details

**Attack Phase:**

* Reads from rising envelope table at 0x2000-0x20FF
* Progresses from 0 to 255 over time specified by Attack parameter
* Linear interpolation between 256 discrete steps

**Decay Phase:**

* Reads from falling envelope table at 0x2100-0x21FF
* Progresses from 255 down to Sustain level
* Time specified by Decay parameter

**Sustain Phase:**

* Holds constant value (Sustain parameter, 0-63)
* No timing involved -持续 until key release

**Release Phase:**

* Reads from falling envelope table at 0x2100-0x21FF
* Progresses from current level down to 0
* Time specified by Release parameter

**Velocity Sensitivity:**
The Dynamic Attack parameters (17 and 46) modify envelope attack times based on key velocity:

* Low velocity → slower attack (higher counter value)
* High velocity → faster attack (lower counter value)
* Allows expressive playing dynamics

### Combined Envelope Examples

**Slow Pad Sound:**

* Attack: 24 (1.93s)
* Decay: 32 (0.97s)
* Sustain: 48 (75% level)
* Release: 24 (1.93s)
* Total note duration (5s held): 1.93 + 0.97 + 5.0 + 1.93 = 9.83 seconds

**Plucked Sound:**

* Attack: 56 (115ms)
* Decay: 40 (540ms)
* Sustain: 0 (silent)
* Release: 63 (8.6ms - not reached due to zero sustain)
* Total note duration: ~655ms regardless of key hold time

**Organ Sound:**

* Attack: 63 (8.6ms)
* Decay: 63 (8.6ms)
* Sustain: 63 (100% level)
* Release: 63 (8.6ms)
* Instant on/off response with minimal envelope shaping

### Parameter Processing

**Front Panel Parameter System:**
The BIT 99 uses a 75-parameter front panel addressing system for editing, which is different from the internal storage format:

* Parameters 1-75: Complete synthesizer parameter set
* Parameter editing through front panel increment/decrement
* Real-time parameter updates affect sound immediately
* Park mode allows auditioning changes before committing

**Parameter Mapping:**

* Front panel parameters map to internal storage locations
* Some parameters stored as bit flags in control bytes
* Others stored as 6-bit or 8-bit values
* Complex parameters like DCO octave/frequency encoded specially

## Data Validation and System Reliability

### Input Validation

* Invalid note numbers clamped to valid MIDI range (0-127)
* Zero velocity always treated as note off regardless of sequence
* Parameter values range-checked before storage
* Program numbers >98 wrap using modulo operation

### Buffer Protection

* Buffer overflow protection with graceful degradation
* MIDI RX overflow: Oldest data discarded
* Keybed RX overflow: Oldest data discarded
* No error indication to sender (protocol limitation)

### Data Integrity

* Battery-backed SRAM preserves patches during power-off
* Checksums used for tape storage validation
* Parameter range checking prevents invalid states
* Default program loading on corruption detection
