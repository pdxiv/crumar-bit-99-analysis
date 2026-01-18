# Crumar BIT 99 MIDI Implementation

## Overview

The BIT 99 implements a comprehensive MIDI interface for external control and patch management. MIDI communication is handled through an external UART connected to I/O register 0x1000, with interrupt-driven transmission and reception for low latency.

## Hardware Interface

### MIDI UART Hardware

**Hardware Configuration:**

* NEC µPD71051 Serial Control Unit (CMOS USART) connected to I/O register 0x1000
* P3.5: USART chip select (1 = enable, 0 = disable)
* P3.2 (INT0): TX ready interrupt
* P3.3 (INT1): RX ready interrupt
* MIDI IN/OUT isolation via external optocouplers
* Standard MIDI baud rate: 31.25 kbaud

### USART Initialization Sequence

During system startup (routine at 0x19CE), the NEC µPD71051 USART is initialized with the following sequence:

1. Set P3.5 high (enable USART chip select)
2. Write initialization sequence to register 0x1000:
   * Three 0x00 bytes (reset sequence)
   * 0x40 (command byte: reset/configure)
   * 0xCE (mode byte: 8 data bits, 1 stop bit, no parity)
   * 0x05 (enable transmit and receive)
3. Clear P3.5 low to complete initialization

**Interrupt Configuration:**

* IE register bit EX1 (0x08): Enable INT1 for MIDI RX
* IE register bit EX0 (0x01): Enable INT0 for MIDI TX
* INT0 and INT1 configured as low priority interrupts
* Interrupts can be preempted by Timer 0 (high priority)

## Buffer Management

### MIDI RX Buffer

**Buffer Specifications:**

* 256-byte circular buffer at 0x0100-0x01FF
* Interrupt-driven reception for low latency
* Buffer pointers and counters in RAM

**Buffer Pointers:**

* Read pointer: Maintained by main loop processing routine
* Write pointer: Updated by INT1 interrupt handler
* Byte counter: Tracks buffer fullness

**Overflow Protection:**

* When buffer full, oldest data discarded
* No error indication to sender (MIDI protocol limitation)
* Running status support improves bandwidth efficiency

### MIDI TX Buffer

**Transmission Management:**

* Interrupt-driven transmission via INT0
* TX ready interrupt fires when UART ready for next byte
* SysEx and program dump transmission managed by main loop
* No dedicated TX buffer (messages assembled on-demand)

## Basic MIDI Channels and Modes

### MIDI Channel Configuration

**Default Settings:**

* Factory default: MIDI Channel 1
* User selectable: Channels 1-16 via front panel programming
* Stored in non-volatile memory at 0xCE
* Immediately effective on change

**OMNI Mode:**

* OMNI mode default: Enabled at startup (flag at 0x29 bit 0 = 1)
* OMNI ON: Receive on all MIDI channels
* OMNI OFF: Receive only on selected channel
* Controlled via MIDI CC 124 (OMNI OFF) and CC 125 (OMNI ON)

**Split Mode Channel Behavior:**

* Lower section: Uses base MIDI channel
* Upper section: Uses base MIDI channel + 1
* Split point stored at 0xCF (default 0x18 = MIDI note 24, C1)
* Independent transpose for upper section

**Double Mode Channel Behavior:**

* Both layers use same MIDI channel
* All notes trigger both upper and lower programs
* Independent voice allocation between layers

## MIDI Message Types

### Channel Voice Messages

**Note On (9nH vv)**

* Status byte: 0x90-0x9F (n = MIDI channel 0-15)
* Data byte 1: Note number (0-127)
* Data byte 2: Velocity (1-127)
* Note number range: Full MIDI range supported, filtered by synthesis engine
* Velocity 0 treated as Note Off
* 6-voice polyphonic with dynamic voice allocation
* Last note priority with voice stealing when all voices active

**Note Off (8nH vv)**

* Status byte: 0x80-0x8F (n = MIDI channel 0-15)
* Data byte 1: Note number (0-127)
* Data byte 2: Velocity (ignored, always treated as 64)
* Alternative: Note On with velocity 0 (preferred by firmware)
* Triggers envelope release phase
* Voice remains allocated until envelope completes

**Pitch Bend (EnH ll hh)**

* Status byte: 0xE0-0xEF (n = MIDI channel 0-15)
* Data byte 1: LSB (0-127) - low 7 bits
* Data byte 2: MSB (0-127) - high 7 bits
* 14-bit resolution: 0x0000-0x3FFF
* Center value: 0x2000 (8192 decimal)
* Stored internally as 8-bit value (MSB only) at 0xCC and 0xD5
* Default range: ±2 semitones
* Applied to all active voices simultaneously
* Updated in real-time via DAC channel VBEND (0x1D)

### Control Change Messages

**CC1: Modulation Wheel (BnH 01 vv)**

* Affects LFO depth when LFO routing enabled
* Value range: 0-127
* Stored at 0xCD and 0xD6
* Real-time modulation of assigned parameters
* No effect if LFO routing disabled

**CC64: Sustain Pedal (BnH 40 vv)**

* Also labeled "Release Pedal" on BIT 99
* 0-63: Pedal OFF (normal note release)
* 64-127: Pedal ON (sustain active notes)
* Prevents envelope release phase while held
* Applied to all voices

**CC123: All Notes Off (BnH 7B 00)**

* Immediately terminates all active voices
* Envelopes forced to release phase
* Does not reset controllers or program state
* Emergency panic function

**CC124: OMNI Mode Off (BnH 7C 00)**

* Disables OMNI mode
* Synth responds only to its assigned MIDI channel
* Flag cleared at 0x29 bit 0

**CC125: OMNI Mode On (BnH 7D 00)**

* Enables OMNI mode
* Synth responds to all MIDI channels
* Flag set at 0x29 bit 0

### Program Change Messages

**Program Change (CnH pp)**

* Status byte: 0xC0-0xCF (n = MIDI channel 0-15)
* Data byte: Program number (0-98)
* Maps to internal programs 1-99 (received value + 1)
* Values >98 wrap around: PC 99 = Program 1, PC 100 = Program 2, etc.

**Program Types:**

* PC 0-74: Single programs (1-75)
* PC 75-98: Split/Double programs (76-99)

**Program Change Disable:**

* Can be disabled via front panel parameter 71
* When disabled, MIDI program changes ignored
* Current program selection unchanged

**Program Loading:**

* Immediate effect on reception
* 37-byte program data loaded from SRAM storage
* All synthesis parameters updated
* Envelopes reset if notes currently playing

## System Exclusive (SysEx) Implementation

### SysEx Message Structure

**General Format:**

```
F0 25 [Device_ID] [Function] [Data...] F7
```

**Header Components:**

* `F0`: SysEx start byte (standard MIDI)
* `25`: Manufacturer ID for BIT synthesizers
* `Device_ID`: Format `2n` where n = MIDI channel (0-15)
  * Channel 1: Device ID = 0x20
  * Channel 2: Device ID = 0x21
  * Channel 16: Device ID = 0x2F
* Alternative Device ID format: `1n` for BIT-01 compatibility
* `Function`: Command code (0-9)
* `F7`: SysEx end byte (standard MIDI)

**Device ID Filtering:**

* Only responds to messages matching current MIDI channel
* OMNI mode: Responds to all device IDs
* Invalid device ID: Message silently ignored

### SysEx Function Codes

#### Function 0: Activate Split Mode

**Format:**

```
F0 25 2n 00 [Split_Point] [Upper_Transpose] F7
```

**Parameters:**

* `Split_Point`: MIDI note number (0-127)
  * Notes below split point use lower program/channel
  * Notes at or above split point use upper program/channel
  * Stored at memory location 0xCF
* `Upper_Transpose`: Transpose value for upper section
  * Range: 0-72 (represents -36 to +36 semitones)
  * Value 36 = no transpose (0 semitones)
  * Stored at memory location 0xD8

**Implementation:**

* Sets split mode flag in system flags (0x28)
* Updates split point and transpose values
* Reconfigures voice allocation for split operation
* Upper section uses base MIDI channel + 1

#### Function 1: Disable Split Mode

**Format:**

```
F0 25 2n 01 F7
```

**Implementation:**

* Clears split mode flag in system flags (0x28)
* Returns to normal (single program) operation
* All keys use lower program and base MIDI channel
* Split point value preserved in memory

#### Function 2: Activate Double Mode

**Format:**

```
F0 25 2n 02 F7
```

**Implementation:**

* Sets double mode flag in system flags (0x28)
* Both programs triggered by all keys
* Both layers use same MIDI channel
* Voice allocation distributes between layers

#### Function 3: Disable Double Mode

**Format:**

```
F0 25 2n 03 F7
```

**Implementation:**

* Clears double mode flag in system flags (0x28)
* Returns to normal (single program) operation
* All voices use lower program only

#### Function 5: Lower Program Change

**Format:**

```
F0 25 2n 05 [Program_Number] F7
```

**Parameters:**

* `Program_Number`: 0-74 (maps to programs 1-75)
* Only single programs supported (not split/double)

**Implementation:**

* Loads specified program into lower program buffer (0x0EFF-0x0F23)
* If not in split/double mode, affects entire synth
* If in split/double mode, affects only lower section
* Program data copied from storage at 0x0380-0x0E56

#### Function 6: Upper Program Change

**Format:**

```
F0 25 2n 06 [Program_Number] F7
```

**Parameters:**

* `Program_Number`: 0-74 (maps to programs 1-75)
* Only single programs supported (not split/double)

**Implementation:**

* Loads specified program into upper program buffer (0x0F24-0x0F48)
* Only affects sound if in split or double mode
* If in split mode, affects keys above split point
* If in double mode, affects layered sound

#### Function 7: Single Program Receive

**Format:**

```
F0 25 2n 07 [Program_Number] [Data_Bytes...] F7
```

**Parameters:**

* `Program_Number`: 0-98 (maps to programs 1-99)
  * 0-74: Single programs
  * 75-98: Split/Double programs
* `Data_Bytes`: Program data encoded as nibbles

**Data Encoding:**
Program data uses nibble splitting to handle 8-bit values in 7-bit MIDI:

* Each program byte (0-255) split into two 4-bit nibbles
* Low nibble sent first, then high nibble
* Example: Byte value 0xAB becomes: `0x0B` (low), `0x0A` (high)

**Program Data Structure:**

*Single Programs (1-75): 37 bytes = 74 SysEx nibbles*

* See software_implementation.md for complete 37-byte patch format
* Offset 0x00-0x24: Parameter values (6-bit and 8-bit)
* Offset 0x22-0x24: Flag bytes with bit-encoded on/off settings

*Split/Double Programs (76-99): 7 bytes = 14 SysEx nibbles*

* Byte 0: Lower program number (0-74)
* Byte 1: Upper program number (0-74)
* Byte 2: Split point (MIDI note number)
* Byte 3: Upper transpose (0-72 = -36 to +36 semitones)
* Bytes 4-6: Mode flags and configuration

**Reception Process:**

1. Verify device ID matches MIDI channel (or OMNI mode active)
2. Extract program number (validate 0-98 range)
3. Receive nibble pairs, reconstruct bytes
4. Verify message ends with F7
5. Write reconstructed data to program storage location
6. If program currently selected, reload into working buffers

#### Function 8: Full Bank Receive

**Format:**

```
F0 25 2n 08 [Data_Bytes...] F7
```

**Parameters:**

* `Data_Bytes`: All 75 single programs sequentially encoded
* Total size: 75 programs × 37 bytes × 2 nibbles/byte = 5550 nibbles
* Does not include split/double programs (76-99)

**Data Structure:**

* Program 1 data (74 nibbles)
* Program 2 data (74 nibbles)
* ...
* Program 75 data (74 nibbles)

**Reception Process:**

1. Verify device ID matches
2. Receive complete bank data stream
3. Validate total byte count (2775 bytes)
4. Write programs sequentially to storage (0x0380-0x0E56)
5. Verify F7 terminator
6. If current program affected, reload into working buffer

**Error Handling:**

* Invalid nibble values: Abort reception, preserve existing data
* Incomplete message: Timeout after ~1 second, discard partial data
* Checksum failure: Reject bank, keep existing programs
* Buffer overflow: Abort, programs may be partially updated

#### Function 9: Request Program Send

**Format:**

```
F0 25 2n 09 [Target_Device_ID] [Program_Number] F7
```

**Parameters:**

* `Target_Device_ID`: Device to send patch to
  * Format `2n` (n = MIDI channel 0-15) or `0n` (BIT-01 format)
  * Can address different BIT 99 in multi-synth setup
* `Program_Number`: 0-98 (maps to programs 1-99)

**Implementation:**

1. Verify sender device ID matches this synth
2. Read specified program from storage
3. Encode program data as nibbles
4. Transmit Function 7 SysEx message to target device
5. Include target device ID in transmitted message header

**Multi-Device Communication:**

* Allows patch copying between BIT 99 units
* Each synth must be on different MIDI channel
* Requesting synth sends to target device ID
* Target synth receives and stores program
* Useful for live performance setups with multiple units

**Example Usage:**

```
BIT 99 #1 (Channel 1, ID 0x20) requests program 5 from BIT 99 #2 (Channel 2, ID 0x21):
F0 25 20 09 21 05 F7

BIT 99 #2 responds by transmitting:
F0 25 21 07 05 [program 5 data nibbles...] F7

BIT 99 #1 receives and stores program at target device ID location
```

### SysEx Error Handling

**Invalid Device ID:**

* Message silently ignored
* No error response transmitted
* OMNI mode: All device IDs accepted

**Invalid Function Code:**

* Message ignored after device ID match
* No error response transmitted
* Future expansion possible with new codes

**Invalid Program Number:**

* Values >98 clamped to valid range using modulo operation
* PC 99 wraps to Program 1, PC 100 to Program 2, etc.
* No error indication

**Data Corruption:**

* Invalid nibble values (>0x0F): Abort reception
* Incomplete message: Timeout protection (~1 second)
* Premature F7: Data up to that point discarded
* Checksum validation on critical data only

**Buffer Overflow:**

* MIDI RX buffer overflow: Oldest data discarded
* Program storage overflow: Write operation aborted
* Prevents memory corruption

## MIDI Implementation Chart

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
| Exclusive     |               |               | format above  |
+---------------+---------------+---------------+---------------+

Mode 1: OMNI ON, POLY    Mode 2: OMNI ON, MONO     O: Yes
Mode 3: OMNI OFF, POLY   Mode 4: OMNI OFF, MONO    X: No
```

## MIDI Message Processing Flow

### Reception Processing (Main Loop)

The main loop checks the MIDI RX buffer (routine at 0x1B00) and processes messages sequentially:

**Message Parsing:**

1. Read status byte from buffer
2. Identify message type (channel voice, system, SysEx)
3. Check channel match (or OMNI mode)
4. Read required data bytes
5. Execute message handler
6. Update buffer read pointer

**Running Status Support:**

* Status bytes 0x80-0xEF cached
* Data bytes without status use last status
* Improves bandwidth efficiency for repeated messages
* Real-time messages (0xF8-0xFF) don't cancel running status

### Note On/Off Processing

**Voice Allocation Algorithm:**

1. Scan voice state array (0x08-0x13) for available voice
2. If all voices busy, find oldest note (voice stealing)
3. Assign MIDI note number to voice slot
4. Store velocity value for envelope scaling
5. Trigger envelope attack phase
6. Update DCO frequency from lookup table
7. Queue control voltage updates

**Note Off Processing:**

1. Search voice state array for matching note number
2. If sustain pedal active, mark for later release
3. Otherwise trigger envelope release phase
4. Voice remains allocated until envelope completes
5. Mark voice as available when envelope reaches zero

### Pitch Bend Processing

**Bend Value Handling:**

1. Receive 14-bit value (LSB first, MSB second)
2. Extract MSB for 8-bit internal representation
3. Store at 0xCC (and mirror at 0xD5)
4. Calculate bend CV value (center = 0x80)
5. Write to DAC via VBEND channel (0x1D)
6. Applied to all active voices simultaneously

**Bend Range:**

* Hardware range: ±2 semitones (factory fixed)
* Software scaling: Linear 0x00-0xFF → -2 to +2 semitones
* Center position 0x80 = no bend

### SysEx Reception State Machine

**State Tracking:**

* Current state: Idle / Receiving Header / Receiving Data / Complete
* Expected byte count based on function code
* Nibble pair assembly for data reconstruction
* Timeout counter for incomplete messages

**Processing Steps:**

1. Detect F0 byte → Enter SysEx mode
2. Verify manufacturer ID (0x25)
3. Check device ID match
4. Identify function code
5. Receive data bytes (nibbles for Functions 7/8)
6. Detect F7 byte → Execute function
7. Return to normal message processing

## Real-Time Performance

### Latency Specifications

* **MIDI RX interrupt**: <50 microseconds from byte arrival to buffer storage
* **MIDI-to-sound**: <10ms (interrupt fills buffer, main loop processes within ~24ms cycle)
* **Note-to-CV**: <3.1ms (updated next Timer 0 interrupt after note processing)
* **Program change**: <50ms (37-byte program load + CV updates)

### Throughput Limits

* **MIDI bandwidth**: 3125 bytes/second (31.25 kbaud hardware limit)
* **Note capacity**: ~375 notes/second theoretical (3 bytes per note)
* **Practical limit**: ~60 notes/second with running status
* **Voice limit**: 6 simultaneous notes (hardware voice allocation)

### CPU Loading

* **MIDI RX interrupt**: ~10-20 machine cycles per byte (minimal)
* **MIDI message processing**: ~500-2000 cycles depending on message type
* **SysEx processing**: Can occupy main loop for several milliseconds during bank dump
* **Background processing**: MIDI handled in remaining 22-35% CPU time after Timer 0 interrupt

### Buffer Management Performance

* **256-byte RX buffer**: ~82ms buffering at maximum MIDI rate
* **Overflow protection**: Oldest data discarded when buffer full
* **Average occupancy**: <10% during typical playing
* **Peak occupancy**: ~50% during dense chord passages or SysEx reception

## Compatibility Notes

### BIT-01 Compatibility

* Alternative device ID format `1n` supported for BIT-01 messages
* Same SysEx function codes used
* Program data format identical
* Cross-compatible patch management

### Standard MIDI Compliance

* Fully compliant with MIDI 1.0 specification
* Running status support per MIDI standard
* No active sensing implementation
* System reset (0xFF) resets synthesizer state
* Standard 5-pin DIN MIDI connectors with optoisolation

### Multi-Timbral Limitations

* Not multi-timbral (single program at a time, or two in split/double)
* Split mode: Two programs, two MIDI channels (base and base+1)
* Double mode: Two programs, one MIDI channel
* 6 total voices shared between programs in split/double mode

## MIDI Initialization Memory Locations

### Critical MIDI Variables

| Address | Purpose                           | Default Value | Notes                          |
|---------|-----------------------------------|---------------|--------------------------------|
| 0x29    | OMNI mode flag (bit 0)            | 0x01          | 1 = OMNI ON                   |
| 0xCC    | Pitch bend value (primary)        | 0x80          | Center = 0x80                 |
| 0xCD    | Modulation wheel value            | 0x00          | 0 = minimum                   |
| 0xCE    | MIDI channel (0-15)               | 0x00          | Channel 1                     |
| 0xD5    | Pitch bend value (mirror)         | 0x80          | Duplicate for split/double    |
| 0xD6    | Modulation wheel (mirror)         | 0x00          | Duplicate for split/double    |
| 0xFFF   | Pitch bend CV output              | 0x80          | DAC value for VBEND           |
| 0xFFE   | Modulation CV output              | 0x00          | DAC value for VMOD            |

### MIDI Buffer Pointers

| Address | Purpose                           | Notes                          |
|---------|-----------------------------------|--------------------------------|
| 0x0100-0x01FF | MIDI RX circular buffer   | 256 bytes                      |
| Read pointer  | Maintained by main loop   | Tracks next byte to process    |
| Write pointer | Updated by INT1 handler   | Tracks next free buffer slot   |
| Byte counter  | Tracks buffer occupancy   | Prevents overflow              |
