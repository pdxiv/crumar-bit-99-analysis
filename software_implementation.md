# Crumar BIT 99 Software Implementation

## Software Architecture Overview

The BIT 99 firmware implements a sophisticated real-time polyphonic synthesizer with comprehensive MIDI support, multiple operating modes, and non-volatile patch storage. The software manages 6-voice polyphony, real-time control voltage generation, user interface, and communication protocols.

**Voice Architecture:** 6-voice polyphonic with dynamic voice allocation

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

### Basic MIDI Support

**MIDI Channels:**
* Factory default: Channel 1
* User selectable via front panel programming
* Stored in non-volatile memory
* OMNI mode can be enabled to receive on all channels

**Note Handling:**
* 6-voice polyphonic with dynamic voice allocation
* Note priority: Last note priority with voice stealing
* Note velocity (1-127) affects envelope dynamics
* Velocity 0 treated as Note OFF
* Split mode: Lower section uses base channel, upper uses base+1
* Double mode: Both layers use same channel

### MIDI Implementation Chart

```
CRUMAR BIT 99 SYNTHESIZER                    [MIDI Implementation Chart]
Model: BIT 99                                 Version: 1.0
Date: 1984

+---------------+---------------+---------------+---------------+
| Function      | Transmitted   | Recognized    | Remarks       |
+---------------+---------------+---------------+---------------+
| Basic         | Default       | 1             | Memorized     |
| Channel       | Changed       | 1-16          | Via front     |
|               |               |               | panel or MIDI |
+---------------+---------------+---------------+---------------+
| Mode          | Default       | Mode 1: OMNI  | Via MIDI CC   |
|               |               | ON, POLY      | 124,125       |
|               | Messages      | 124,125       |               |
|               | Altered       | ************  |               |
+---------------+---------------+---------------+---------------+
| Note          | Number        | 0-127         | 6-voice       |
| Number        | True Voice    | ************  | polyphonic    |
+---------------+---------------+---------------+---------------+
| Velocity      | Note ON       | O 9nH v=1-127 | Velocity      |
|               | Note OFF      | X 8nH v=64    | affects       |
|               |               | O 9nH v=0     | envelopes     |
+---------------+---------------+---------------+---------------+
| After         | Key's         | X             |               |
| Touch         | Ch's          | X             |               |
+---------------+---------------+---------------+---------------+
| Pitch Bend    |               | O             | ±2 semitones  |
|               |               |               | (default)     |
+---------------+---------------+---------------+---------------+
| Control       | 1 Mod Wheel   | O             | LFO depth     |
| Change        | 64 Sus Pedal  | O             | Release pedal |
|               | 123 All Off   | O             | All notes off |
|               | 124 OMNI OFF  | O             | Mode control  |
|               | 125 OMNI ON   | O             | Mode control  |
+---------------+---------------+---------------+---------------+
| Program       |               | O 0-98        | Programs 1-99 |
| Change        |               |               | (wraparound)  |
+---------------+---------------+---------------+---------------+
| System        | Song Pos      | X             |               |
| Common        | Song Sel      | X             |               |
|               | Tune          | X             |               |
+---------------+---------------+---------------+---------------+
| System        | Clock         | X             |               |
| Real Time     | Commands      | X             |               |
+---------------+---------------+---------------+---------------+
| Aux           | Local ON/OFF  | X             |               |
| Messages      | All Notes OFF | O             | CC 123        |
|               | Active Sense  | X             |               |
|               | Reset         | O             | FFH           |
+---------------+---------------+---------------+---------------+
| System        |               | O             | See detailed  |
| Exclusive     |               |               | format below  |
+---------------+---------------+---------------+---------------+

Mode 1: OMNI ON, POLY    Mode 2: OMNI ON, MONO     O: Yes
Mode 3: OMNI OFF, POLY   Mode 4: OMNI OFF, MONO    X: No
```

### MIDI Controller Support

**Implemented Controllers:**
* CC1 (Mod Wheel): 0-127, affects LFO depth when enabled
* CC64 (Sustain Pedal): 0-63=OFF, 64-127=ON, release pedal function
* CC123 (All Notes Off): Immediate voice termination
* CC124 (OMNI Mode Off): Disables OMNI mode
* CC125 (OMNI Mode On): Enables OMNI mode

**Pitch Bend:**
* 14-bit resolution (0000H-3FFFH)
* Default range: ±2 semitones
* Range not user-adjustable via MIDI
* Applied to all active voices

**Program Change:**
* Receives PC 0-98 (maps to internal programs 1-99)
* PC values >98 wrap around (PC 99 = Program 1, etc.)
* Programs 1-75: Single programs
* Programs 76-99: Split/Double programs
* Program changes can be disabled via parameter 71

### MIDI SysEx Implementation

The BIT 99 uses a comprehensive SysEx protocol for patch management and system control:

**SysEx Message Format:**

```
F0 25 [Device_ID] [Function] [Data...] F7
```

**Header Bytes:**
* `F0`: SysEx start
* `25`: BIT manufacturer ID  
* `Device_ID`: Format `2n` where n = MIDI channel (0-15)
  + Examples: `20` = BIT 99 on MIDI channel 1,  `21` = MIDI channel 2
  + Alternative: `1n` for BIT-01 compatibility
* `Function`: Command code (0-9)
* `F7`: SysEx end

**SysEx Function Codes:**

*Function 0: Activate Split Mode*

```
F0 25 2n 00 [Split_Point] [Upper_Transpose] F7
```

* `Split_Point`: MIDI note number (0-127)
* `Upper_Transpose`: Transpose value for upper split (-36 to +36)

*Function 1: Disable Split Mode*

```
F0 25 2n 01 F7
```

*Function 2: Activate Double Mode*  

```
F0 25 2n 02 F7
```

*Function 3: Disable Double Mode*

```
F0 25 2n 03 F7
```

*Function 5: Lower Program Change*

```
F0 25 2n 05 [Program_Number] F7
```

* `Program_Number`: 0-74 (for programs 1-75)

*Function 6: Upper Program Change*

```
F0 25 2n 06 [Program_Number] F7
```

* `Program_Number`: 0-74 (for programs 1-75)

*Function 7: Single Program Receive*

```
F0 25 2n 07 [Program_Number] [Data_Bytes...] F7
```

* `Program_Number`: 0-98 (for programs 1-99)
* `Data_Bytes`: Program data encoded as nibbles (see below)

*Function 8: Full Bank Receive*

```
F0 25 2n 08 [Data_Bytes...] F7
```

* Receives all 75 single programs sequentially
* Each program is 37 bytes encoded as 74 nibbles

*Function 9: Request Program Send*

```
F0 25 2n 09 [Target_Device_ID] [Program_Number] F7
```

* `Target_Device_ID`: Device to send patch to (format `2n` or `0n`)
* `Program_Number`: 0-98 (for programs 1-99)

**Data Encoding:**
Program data is encoded using nibble splitting to avoid MIDI's 7-bit limitation:
* Each program byte (0-255) is split into two 4-bit nibbles
* Low nibble sent first, then high nibble
* Example: Byte value `0xAB` becomes two SysEx bytes: `0x0B`,    `0x0A`

**Multi-Device Support:**
* Each BIT 99 can be assigned a unique MIDI channel (0-15)
* Device ID in SysEx messages allows selective addressing
* Broadcast possible using OMNI mode
* Compatible with BIT-01 using alternative device ID format

## Buffer Management

### MIDI Buffer Management

* 256-byte circular MIDI input buffer at 0x0100-0x01FF
* Interrupt-driven reception for low latency
* Running status support for efficiency
* Overflow protection with oldest data discard

### Keybed Interface Protocol

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
   - Receive note number from keybed
   - Add 0x0C to form MIDI note number
   - Store in location 0xD9 (current key note)
   - Set flag to expect velocity byte next

2. **Waiting for Velocity** (flag 0x3D bit 5 set):
   - Receive velocity value from keybed
   - Store in location 0xDA (current key velocity)
   - Set flag 0x3D bit 4 to trigger note processing
   - Clear velocity flag, return to waiting for note

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
   - Attack, Decay, Sustain, Release phases
   - Separate VCF and VCA envelopes
   - Velocity-sensitive envelope scaling

2. **LFO Processing**:
   - Dual LFOs with multiple waveforms (triangle, sawtooth, square)
   - Configurable routing to DCO, VCF, VCA
   - Delay and rate parameters

3. **Modulation Matrix**:
   - LFO-to-destination routing
   - Keyboard tracking for VCF
   - Velocity sensitivity
   - Pitch bend and mod wheel integration

4. **Control Voltage Calculation**:
   - Per-voice filter cutoff (VC1-VC6)
   - Per-voice amplitude control (VEN1-VEN6)
   - DCO pulse width modulation (VPU1-VPU4)
   - Global modulation sources

### Lookup Table Processing

The firmware uses several lookup tables for efficient real-time processing:

**Envelope Tables:**
* Rising envelope/Triangle LFO table (0x2000-0x20FF): Attack phases and triangle LFO
* Falling envelope/Sawtooth LFO table (0x2100-0x21FF): Decay/release phases and sawtooth LFO
* Secondary modulation curve (0x2200-0x22FF): Special modulation shapes

**Timing Tables:**
* Primary timer values (0x2300-0x237F): Envelope timing calculations
* Secondary timer values (0x2380-0x23FF): Extended timing ranges

**Frequency Tables:**
* MIDI note to timer divider (0x2400-0x24F3): DCO frequency generation
* DCO pulse width values (0x2500-0x25FF): Pulse width modulation

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

## Error Handling and Data Validation

### MIDI Error Handling

* Invalid device ID: Message ignored
* Invalid function code: Message ignored  
* Invalid program number: Clamped to valid range
* Checksum validation on received data
* Timeout protection during multi-byte receives

### System Reliability

* Framing errors ignored (no parity checking)
* Out-of-sequence bytes handled gracefully
* Invalid note numbers clamped to valid MIDI range
* Zero velocity always treated as note off regardless of sequence
* Buffer overflow protection with graceful degradation

### Data Integrity

* Battery-backed SRAM preserves patches during power-off
* Checksums used for tape storage validation
* Parameter range checking prevents invalid states
* Default program loading on corruption detection
