# NEC µPD71051 USART Documentation

This is an attempt to convert the µPD71051 USART documentation to markdown format, for aiding with reverse-engineering of the Crumar Bit99 hardware and firmware. Many mistakes are present in this particular markdown version, however, so take everything in here with a grain of salt.

## Description

The **µPD71051** serial control unit is a CMOS USART designed to provide serial data communications in microcomputer systems. The CPU uses it as a peripheral I/O device and programs it to communicate in synchronous or asynchronous serial data transmission protocols, including IBM bisync.

The USART receives serial data streams and converts them into parallel data characters for the CPU. While receiving serial data, the USART can also accept parallel data from the CPU, convert it to serial, and transmit the data. The USART signals the CPU when it has received or transmitted a character and requires service. The CPU may read complete USART status data at any time.

## Features

* **Synchronous operation**
  + One or two SYNC characters
  + Internal/external synchronization
  + Automatic SYNC character insertion
* **Asynchronous operation**
  + Clock rate (baud rate): x1, x16, or x64
  + Send stop bits: 1, 1.5, or 2 bits
  + Break transmission
  + Automatic break detection
  + Valid start bit detection
* **Baud rate:** DC-240 kbit/s at x1 clock
* **Full duplex**, double-buffered transmitter/receiver
* **Error detection:** parity, overrun, and framing
* **Five- to eight-bit characters**
* **Low-power standby mode**
* Compatible with standard microcomputers
* Functionally equivalent to (except standby mode) and can replace the **µPD8251AF**
* CMOS technology
* Single **+5 V ±10%** power supply
* Industrial temperature range **-40 to +85°C**
* Packages:
  + 28-pin plastic DIP or PLCC
  + 44-pin plastic QFP
* 8 MHz and 10 MHz versions

## Pin Identification

| Pin | Symbol   | Function                              |
| --- | -------- | ------------------------------------- |
|   1 | D2       | Data bus                              |
|   2 | D3       | Data bus                              |
|   3 | RxDATA   | Receive data input                    |
|   4 | GND      | Ground                                |
|   5 | D4       | Data bus                              |
|   6 | D5       | Data bus                              |
|   7 | D6       | Data bus                              |
|   8 | D7       | Data bus                              |
|   9 | /TxCLK   | Transmitter clock input               |
|  10 | /WR      | Write strobe input                    |
|  11 | /CS      | Chip select input                     |
|  12 | C//D     | Control or data input                 |
|  13 | /RD      | Read strobe input                     |
|  14 | RxRDY    | Receiver ready output                 |
|  15 | TxRDY    | Transmitter ready output              |
|  16 | SYNC/BRK | Synchronization/Break input/output    |
|  17 | /CTS     | Clear to send input                   |
|  18 | TxEMP    | Transmitter empty output              |
|  19 | TxDATA   | Transmit data output                  |
|  20 | CLK      | Clock input                           |
|  21 | RESET    | Reset input                           |
|  22 | /DSR     | Data set ready input                  |
|  23 | /RTS     | Request to send output                |
|  24 | /DTR     | Data terminal ready output            |
|  25 | /RxCLK   | Receiver clock input                  |
|  26 | VDD      | +5 V power supply                     |
|  27 | D0       | Data bus                              |
|  28 | D1       | Data bus                              |

## Pin Functions

### D7-D0 - Data Bus

D7-D0 are an 8-bit, 3-state, bidirectional data bus. The bus transfers data by connecting to the CPU data bus.

### RESET - Reset

A high level to the RESET input resets the µPD71051 and puts it in an idle state.  
The device performs no operatisns in the idle state.

The µPD71051 enters standby mode when RESET transitions from high to low. Standby mode ends when the CPU writes a mode byte to the chip.

* Minimum reset pulse width: **≥ 6 tCYK cycles**
* Clock must be active during reset

## Control, Clock, and Strobe Inputs

### CLK - Clock

This clock input produces internal timing for the µPD71051.

* Clock frequency must be **≥ 30×** the TxCLK or RxCLK input in sync/async mode (x1 clock)
* For async mode using x16 or x64 clock modes, frequency must be **≥ 4.5×** TxCLK or RxCLK

### CS - Chip Select

The CS input selects the µPD71051.

* **CS = 0** -> device selected
* **CS = 1** -> device not selected; 
  + Data bus (D7-D0) becomes high-impedance
  + RD and WR are ignored

### RD - Read Strobe

RD is active low. It is used when reading data or status from the device.

### WR - Write Strobe

WR is active low. It is used when writing data or command bytes to the µPD71051.

### C/‾D - Control or Data Select

This pin determines the access type:

* **C/‾D = 1** -> control byte (command or status)
* **C/‾D = 0** -> data byte

Typically connected to A0 on the CPU address bus.

## Modem and Serial Interface Pins

### DSR - Data Set Ready

General-purpose input for modem control.  
Status is reflected in **status byte bit 7**.

### DTR - Data Terminal Ready

General-purpose modem-control output. Controlled via **command byte bit 1**:

* bit1 = 0 -> DTR = 1
* bit1 = 1 -> DTR = 0

### RTS - Request to Send

General-purpose modem-control output. Controlled via **command byte bit 5**:

* bit5 = 1 -> RTS = 0
* bit5 = 0 -> RTS = 1

### CTS - Clear to Send

Controls data transmission.

* µPD71051 transmits when **CTS = 0** *and* **TxEN = 1**
* If CTS becomes 1 during transmission, the USART:
  + stops after completing buffered data
  + sets TxDATA high

## Data Signals

### TxDATA - Transmit Data

Serial data output pin.

### TxRDY - Transmitter Ready

Indicates the transmit buffer is empty and new data may be written.

* Masked by **TxEN** and **CTS**
* Status available in **status byte bit 0**
* Cleared by WR falling edge
* Writing while TxRDY = 0 overwrites unsent data

### TxEMP - Transmitter Empty

Indicates both transmit buffers (first and second) are empty.

* Becomes 1 when the second buffer empties
* In async mode -> TxDATA goes high
* Reset to 0 when CPU writes new transmit data

Used for detecting when to switch between send/receive in half-duplex applications.

### TxCLK - Transmitter Clock

Reference clock input determining transmit rate.

* In sync mode, data transmitted at same rate as TxCLK
* In async mode, TxCLK = 1×, 16×, or 64× baud rate

Example (for 2400 baud async):

* x1 -> 2.4 kHz
* x16 -> 38.4 kHz
* x64 -> 153.6 kHz

### RxDATA - Receive Data

Serial data input pin.

### RxRDY - Receiver Ready

Goes high when a received character is moved to the receive data buffer.

* Status available in **status byte bit 1**
* Cleared when CPU reads the data
* If CPU fails to read before next character arrives -> **overrun error (OVE)**
* Disabled by clearing RxEN bit in command byte

### SYNC/BRK - Synchronization / Break

#### SYNC (sync mode)

* Detects SYNC characters
* Internal or external sync selectable
* SYNC status readable in **status byte bit 6**

#### BRK (async mode)

Indicates detection of a break condition.

* BRK = high when RxDATA stays low for two character lengths
* Status also held in bit 6 of status byte
* Cleared when RxDATA returns high or on reset

### RxCLK - Receiver Clock

`RxCLK` is a reference clock input that controls the receive data rate.

* **Sync mode:** receiving rate = RxCLK
* **Async mode:** RxCLK may be 1×, 16×, or 64× the receive rate
* Serial data from RxDATA is sampled on the **rising edge** of RxCLK.

### VDD - Power

+5 V power supply.

### GND - Ground

Ground reference.

## µPD71051 Functions

The **µPD71051** is a CMOS serial control (USART) unit that provides serial communications in microcomputer systems.  
The CPU uses it as an ordinary I/O device.

The µPD71051 can operate in **synchronous** or **asynchronous** mode.

* **Sync mode:**  
    Bit length, number of SYNC characters, and SYNC detection mode must be specified.

* **Async mode:**  
    Baud rate, character length, stop bits, etc., must be specified.

Parity may be used in either mode.

The µPD71051 converts:

* CPU parallel data -> serial transmit data (TxDATA)
* Serial input data (RxDATA) -> CPU-readable parallel data

## Mode Register

When the µPD71051 is in standby mode, writing a **mode byte** releases standby mode.

* **Figure 4**: async mode format
* **Figure 5**: sync mode format

To designate sync mode, **bits 0 and 1 = 00**.  
Any other combination selects async mode.

### Parity Bits (P1, P0)

* P1, P0 = `01` -> odd parity
* P1, P0 = `11` -> even parity
* P0 = `0` -> parity disabled

### Character Length (L1, L0)

These bits select **n** data bits per character (parity excluded).

* The µPD71051 uses the **lower n bits** of written data
* Upper (8-n) bits read back as 0

### Stop Bits (Async Only - ST1, ST0)

ST1, ST0 determine number of stop bits added during transmission.

### Clock Multiplier Bits (Async Only - B1, B0)

Select TxCLK/RxCLK multiplier:  
`1×` , `16×` , or `64×`

> Note: 1× mode requires synchronous matching between sender and receiver; it is rarely used in async mode.

### Sync-Mode Bits (SSC, EXSYNC)

* **SSC** = number of SYNC characters
  + `1` = one SYNC character
  + `0` = two SYNC characters

* **EXSYNC** selects sync detection source
  + `0` = internal sync detection
  + `1` = external sync detection

SYNC characters are written immediately after the mode byte.

## Status Summary

The CPU may read µPD71051 status at any time (except standby).  
The status register:

* Reports errors
* Indicates transmit/receive readiness
* Assists in data reading/writing operations

The device may be reset by hardware or software into **standby mode**, releasing prior configuration.

## Status Register

The status register reports device status:

* CPU must set **C/‾D = 1** and **RD = 0** to read status
* Status is not updated while being read
* Updates propagate after **≥ 28 clock cycles**

(Format shown in Figure 7)

## Receive Data Buffer

After serial->parallel conversion:

* Data moves into the receive buffer
* `RxRDY` becomes **1** when a character is ready
* CPU reads the buffer to clear RxRDY
* If CPU fails to read before next character arrives -> **overrun error (OVE)**

## Connecting the µPD71051 to the System

The CPU uses two I/O addresses selected by **C/‾D**:

* **C/‾D = 0:** transmit/receive data register
* **C/‾D = 1:** mode, command, and status registers

Typically, CPU address bit A0 connects to C/‾D to form consecutive I/O addresses.

`TxRDY` and `RxRDY` connect to the CPU or interrupt controller.

## Operating the µPD71051

### Initialization

1. Apply hardware reset (RESET high) after power-up -> enters standby
2. CPU writes a **mode byte**
3. In async mode -> CPU may immediately send command byte
4. In sync mode -> CPU writes SYNC character(s) next
5. Then a command byte may be written

After first command byte, device is active.

Writing another command byte:

* If it includes software reset -> device returns to standby mode
* Waits for a new mode byte

## Command Register

Commands control sending/receiving operations.

* CPU sets **C/‾D = 1**
* Command byte is written after the mode byte
* In sync mode, command byte may be written *only after* SYNC characters

### Important Command Bits

* **EH** - enter hunt phase (sync)
* **SRES** - software reset; enters standby
* **RTS** - controls RTS output (low when RTS bit = 1)
* **ECL** - clear error flags (PE, OVE, FE)
* **SBRK** - force break (TxDATA low); SBRK=0 releases break
* **RxEN** - receiver enable (RxEN=1) / disable
* **DTR** - controls DTR output (low when bit = 1)
* **TxEN** - transmitter enable/disable

When TxEN=0:

* Transmission stops after current data
* TxDATA goes high (mark)

## Transmit Data Buffer

* Holds written parallel data
* Transfers contents to the transmitter
* Transmitter adds start, stop, and parity bits
* Output flows through TxDATA

## Control Register

Stores both the **mode** and **command** bytes.

## Control Logic

Manages internal block control according to mode, commands, and signals.

## Synchronous Character Register

Holds 1 or 2 SYNC characters.

* During transmission: SYNC characters are sent when no new data is available and TxEMP = 1
* During reception: synchronization achieved when incoming characters match stored SYNC characters

## Transmitter

* Converts transmit data buffer contents parallel -> serial
* Sends via TxDATA
* Adds start/stop/parity as needed

## Receiver

* Converts RxDATA serial input to parallel
* Stores result in receive buffer
* CPU reads buffer
* Sync mode: checks SYNC and parity
* Async mode: checks start, stop, parity

Async reception does not begin until:

* One valid stop bit (high level) is seen on RxDATA
* **RxEN = 1**

## Modem Control

Controls:

* **CTS**
* **RTS**
* **DSR**
* **DTR**

These pins may also be used as general-purpose I/O.

The **TxEMP** and **RxRDY** bits have the same meaning as the pins of the same name.

The **SYNC/BRK** bit generally has the same meaning as the SYNC/BRK pin. In *external* synchronization mode, the status of this bit does not always coincide with the pin. In this case:

* SYNC pin becomes an **input**
* Status bit goes to **1** when a rising edge is detected
* Status bit remains **1 until read**, even if pin returns low
* Status bit also becomes **1** when a SYNC character is input at the RxDATA pin, even if the SYNC pin is low

### DSR Status Bit

The **DSR bit** reflects the level of the DSR pin:

* Status bit = **1** when the DSR pin is **low**

### FE - Framing Error

The **FE bit** becomes 1 when **less than one stop bit** is detected at the end of a data block during asynchronous reception. Figure 8 illustrates how a framing error can occur.

### OVE - Overrun Error

The **OVE bit** becomes 1 when:

* The CPU delays reading the received data
* Two new data bytes arrive
* The first byte is overwritten and lost

Figure 9 shows a typical overrun condition.

### PE - Parity Error

The **PE bit** becomes 1 when a parity error occurs during receive operations.

Framing, overrun, and parity errors **do not disable** µPD71051 operation.  
All three are cleared to **0** by writing a command byte with **ECL = 1**.

### TxRDY Status Bit

The **TxRDY bit** becomes 1 when the transmit data buffer is **empty**.

The **TxRDY output pin** becomes 1 when:

* Transmit data buffer is empty
* CTS pin is low
* TxEN = 1

That is:

    TxRDY (bit) = Transmit Data Buffer Empty
    TxRDY (pin) = (Transmit Data Buffer Empty) • (CTS = 0) • (TxEN = 1)

## Sending in Asynchronous Mode

The **TxDATA** pin is normally **high** (marking state) when no data is being sent.

When the CPU writes transmit data:

1. µPD71051 moves data from transmit buffer to send buffer
2. Sends 1 start bit (low)
3. Sends **data bits**
4. If parity enabled -> inserts parity bit
5. Sends programmed stop bit(s)

Figure 10 shows async character format.

Serial data is sent on the **falling edge** of:

* TxCLK
* or TxCLK ÷ {1, 16, 64}

### Break Operation

When **SBRK = 1**:

* TxDATA pin is forced low
* Occurs regardless of whether data is pending

Figure 11 shows a sample async send program.  
Figure 12 shows TxDATA waveforms.

## Sending in Synchronous Mode

After entering sync mode and enabling the transmitter:

* TxDATA remains **high** until the CPU writes a character (typically a SYNC byte)

During transmission:

* One bit is sent per **falling edge of TxCLK**, if CTS is low
* **Start/stop bits are not used**
* Parity may be enabled

Figure 15 shows sync data formats.

### Transmit Behavior

Once transmission begins:

* CPU must write data at the same rate as TxCLK
* If **TxEMP = 1** due to CPU delay
  + µPD71051 automatically sends SYNC characters
  + Resumes data transmission when CPU writes next byte
* TxEMP resets to 0 when data is written

### Automatic SYNC Transmission Details

SYNC characters are sent **only after** the CPU writes new data.  
Enabling the transmitter alone does **not** send SYNC characters.

Figure 16 shows these timing details.

### Sending Commands During SYNC Transmission

If a command is written while:

* SYNC characters are being transmitted
* TxEMP = 1

The device may misinterpret the command as **data**.

To safely send commands:

1. CPU must write a SYNC character
2. Then immediately write the command byte while SYNC is being transmitted

Figure 17 illustrates this sequence.

Figure 18 shows a typical sync-mode send routine.

## Receiving in Asynchronous Mode

The **RxDATA** pin is normally **high** when idle (figure 13).

µPD71051 detects the **falling edge** that begins a valid start bit.

### Start Bit Detection

When using ×16 or ×64 clocks:

* Device samples RxDATA **½ bit time** after the falling edge
* If level is low -> valid start bit
* If high -> ignore and continue waiting

### Data Sampling

After the start bit:

* Data bits, parity, and stop bit sampling points are determined by a bit counter
* Using X1 clock: sample on rising edges of RxCLK
* Using ×16 or ×64 clock: sample at the nominal middle of each bit

### Receive Buffer Behavior

When a character is assembled:

* It is transferred to the receive buffer
* `RxRDY` is set to **1**
* CPU must read data -> RxRDY returns to **0**

### Error Handling - Async Receive

During stop bit:

* If low -> **framing error (FE)**
* If parity mismatch -> **parity error (PE)**
* If previous data unread and new data arrives -> **overrun error (OVE)**

Receive operations continue despite errors.

### Break Detection

If RxDATA remains low for more than **two data blocks**:

* Device enters **break state**
* SYNC/BRK pin status becomes **1**

### Additional Async Behavior

* A valid start bit is not recognized until RxDATA is high for at least **one bit time**
* A typical receive program is shown in figure 14 (corresponds to async transmit example)

## Standby Mode

The **µPD71051** is a low-power CMOS device.  
In **standby mode**, it disables the external input clocks to the internal circuitry (**CLK**, **TxCLK**, and **RxCLK**), reducing power consumption.

A **hardware reset** is one way to enter standby mode:

* When the **RESET** pin is driven high, 
* The µPD71051 enters standby mode on the **falling edge** of RESET.

A **software reset command** is the other way to enter standby mode.

The **only** way to exit standby mode is for the CPU to write a **mode byte**.

### Pin States in Standby Mode

While in standby:

* **TxRDY**, **TxEMP**, **RxRDY**, and **SYNC/BRK** pins -> **low**
* **TxDATA**, **DTR**, and **RTS** pins -> **high**

Figure 21 shows the timing for entering standby mode.

While the internal standby signal is high, external clocks to the µPD71051 are ignored.  
If **data** (C/‾D = 0) is written to the µPD71051 during standby, operation becomes undefined and unpredictable behavior may occur.

## Receiving in Synchronous Mode

To receive in sync mode, the receiver must first **synchronize** with the transmitting side.

The very first command after:

1. Setting sync mode
2. Writing the SYNC character(s)

must have:

* **EH = 1**
* **ECL = 1**
* **RxEN = 1**

### Hunt Phase

When hunt phase begins, all bits in the receive buffer are set to 1.

#### Internal Synchronization

During internal sync:

* Data on **RxDATA** is clocked into the receive buffer on each rising edge of **RxCLK**
* Each sampled value is compared with the SYNC character(s)

Figure 19 shows this internal detection mechanism.

### Detecting SYNC

* If parity is **not** used:  
    SYNC is detected when the receive buffer exactly matches the SYNC character.  
    SYNC goes high during the *center* of the last SYNC bit.

* If parity **is** used:  
    SYNC becomes high in the center of the **parity bit**.

Receiving begins with the bit **after** the one where SYNC is set to 1.

#### External Sync Detection

In external sync mode, synchronization happens when:

* An external circuit holds the **SYNC pin high**
* For at least **one period** of RxCLK

The hunt phase ends and data reception starts.

At this moment:

* SYNC status bit becomes **1**
* It returns to **0** when the CPU reads the status
* The SYNC bit is also set when the SYNC pin has a rising edge followed by a high level for more than one RxCLK period

### Resynchronizing

The µPD71051 may regain lost sync at any time by issuing an **enter hunt phase** command.

### After Synchronization

After sync is established:

* Received characters are compared to the SYNC character continuously
* SYNC becomes 1 each time a SYNC byte is received
* SYNC bit becomes 0 again when status is read

### Error Handling - Sync Receive

Overrun and parity errors are handled the same way as in async mode:

* Errors affect **only the status flags**
* Parity is **not** checked during hunt phase

Figure 20 shows a sample program illustrating sync-mode reception (paired with the sync-mode transmit program).

**Important:**  
The **TxCLK** frequency of the transmitter and the **RxCLK** frequency of the receiver must be **identical**.

## Figure 4 - Mode Byte for Setting Asynchronous Mode

(C/‾D = 1)

The **Mode Byte** is an 8-bit value written by the CPU to configure the µPD71051 for **asynchronous** operation.

    Bit:   7    6    5    4    3    2    1    0
           ST1  ST0  P1   P0   L1   L0   B1   B0

## Bit 1-0: Baud Rate Selection (B1, B0)

| B1 | B0 | Baud Rate Mode |
| -- | -- | -------------- |
| 0  | 1  | x1 clock       |
| 1  | 0  | x16 clock      |
| 1  | 1  | x64 clock      |

> **Note:** `B1 B0 = 00` is not used for async mode.

### Bit 3-2: Character Length (L1, L0) - Async Mode

| L1 | L0 | Character Length |
| -- | -- | ---------------- |
| 0  | 0  | 5-bit            |
| 0  | 1  | 6-bit            |
| 1  | 0  | 7-bit            |
| 1  | 1  | 8-bit            |

### Bit 5-4: Parity Generate / Check (P1, P0) - Async Mode

| P1 | P0 | Parity Mode |
| -- | -- | ----------- |
| x  | 0  | No parity   |
| 0  | 1  | Odd parity  |
| 1  | 1  | Even parity |

> **x = don’t care**  
> For P0 = 0, parity is disabled and P1 is ignored.

### Bit 7-6: Transmit Stop Bits (ST1, ST0)

| ST1 | ST0 | Stop Bit Setting     |
| --- | --- | -------------------- |
| 0   | 0   | Illegal (do not use) |
| 0   | 1   | 1-bit                |
| 1   | 0   | 1.5-bit              |
| 1   | 1   | 2-bit                |

## Summary Table (All Bits) - Async Mode

| Bits | Name     | Purpose                     |
| ---- | -------- | --------------------------- |
| 7-6  | ST1, ST0 | Stop bit length             |
| 5-4  | P1, P0   | Parity generate/check       |
| 3-2  | L1, L0   | Character length            |
| 1-0  | B1, B0   | Baud rate divisor selection |

## Figure 5 - Mode Byte for Setting Synchronous Mode

(C/‾D = 1)

The **Mode Byte** configures the µPD71051 for **synchronous** operation.  
In synchronous mode, **bits 1 and 0 must be `00` **, as shown in the diagram.

    Bit:   7      6       5    4    3    2    1    0
           SSC   EXSYNC   P1   P0   L1   L0   0    0

### Bit 3-2: Character Length (L1, L0) - Sync Mode

| L1 | L0 | Character Length |
| -- | -- | ---------------- |
| 0  | 0  | 5-bit            |
| 0  | 1  | 6-bit            |
| 1  | 0  | 7-bit            |
| 1  | 1  | 8-bit            |

### Bit 5-4: Parity Generate / Check (P1, P0) - Sync Mode

| P1 | P0 | Parity Mode |
| -- | -- | ----------- |
| x  | 0  | No parity   |
| 0  | 1  | Odd parity  |
| 1  | 1  | Even parity |

> **x = don’t care**  
> Parity is optional in sync mode.

### Bit 6: EXSYNC - Sync Detect Mode

| EXSYNC | Sync Detection Mode    |
| ------ | ---------------------- |
| 0      | Internal (SYNC output) |
| 1      | External (SYNC input)  |

**Internal detection**: SYNC characters are detected by comparing received bits with SYNC registers.  
**External detection**: Sync detection occurs when the external SYNC pin changes state.

### Bit 7: SSC - Number of SYNC Characters

| SSC | SYNC Characters              |
| --- | ---------------------------- |
| 0   | 2 SYNC characters (BSC mode) |
| 1   | 1 SYNC character             |

> In synchronous mode, these SYNC characters must be written immediately after the mode byte.

### Summary Table (All Bits) - Sync Mode

| Bits | Name   | Purpose                           |
| ---- | ------ | --------------------------------- |
| 7    | SSC    | Number of SYNC characters         |
| 6    | EXSYNC | Internal vs. external sync detect |
| 5-4  | P1, P0 | Parity generation/check           |
| 3-2  | L1, L0 | Character length                  |
| 1-0  | 0, 0   | Required value for sync mode      |

## Figure 6 - Command Byte Format

(C/‾D = 1)

The **Command Byte** controls transmitter/receiver enablement, modem-control pin states, error-flag clearing, break generation, software reset, and (in synchronous mode) entry into the hunt phase.

    Bit:   7     6      5     4     3      2      1      0
           EH   SRES    RTS   ECL   SBRK   RxEN   DTR    TxEN

### Bit 0 - TxEN (Transmit Enable)

| TxEN | Function         |
| ---- | ---------------- |
| 0    | Disable transmit |
| 1    | Enable transmit  |

When disabled, transmission stops after any ongoing character finishes, and **TxDATA goes high** (mark).

### Bit 1 - DTR (DTR Pin Control)

| DTR Bit | DTR Output |
| ------- | ---------- |
| 0       | DTR = 1    |
| 1       | DTR = 0    |

(*Active-low output control*)

### Bit 2 - RxEN (Receive Enable)

| RxEN | Function        |
| ---- | --------------- |
| 0    | Disable receive |
| 1    | Enable receive  |

If disabled, incoming data is ignored and **RxRDY will not assert**.

### Bit 3 - SBRK (Send Break)

| SBRK | Function             |
| ---- | -------------------- |
| 0    | Normal TxDATA output |
| 1    | Force TxDATA = 0     |

Break condition forces TxDATA low regardless of the transmit buffer.

### Bit 4 - ECL (Error Clear)

| ECL | Function             |
| --- | -------------------- |
| 0   | No operation         |
| 1   | Clear PE/OVE/FE bits |

Clears all receive-error flags (parity, overrun, framing).  
Does **not** affect enabling/disabling of transmitter or receiver.

### Bit 5 - RTS (RTS Pin Control)

| RTS Bit | RTS Output |
| ------- | ---------- |
| 0       | RTS = 1    |
| 1       | RTS = 0    |

(*Active-low modem-control signal*)

### Bit 6 - SRES (Software Reset)

| SRES | Function                            |
| ---- | ----------------------------------- |
| 0    | No operation                        |
| 1    | Software reset -> enter standby mode |

A software reset behaves like a hardware reset:  
the device enters **standby mode** and remains idle until a **mode byte** is written.

### Bit 7 - EH (Enter Hunt Phase)

(Valid only in synchronous mode)

| EH | Function         |
| -- | ---------------- |
| 0  | No operation     |
| 1  | Enter hunt phase |

Hunt phase forces the receiver to search for SYNC characters again.

> **Note:** EH is *ignored* in asynchronous mode.

### Summary Table - Command Byte

| Bits | Name | Purpose                                  |
| ---- | ---- | ---------------------------------------- |
| 7    | EH   | Enter hunt phase (sync mode only)        |
| 6    | SRES | Software reset -> enters standby mode     |
| 5    | RTS  | RTS output control                       |
| 4    | ECL  | Clear parity/overrun/framing error flags |
| 3    | SBRK | Force break (TxDATA = 0)                 |
| 2    | RxEN | Receiver enable/disable                  |
| 1    | DTR  | DTR output control                       |
| 0    | TxEN | Transmitter enable/disable               |

## Figure 7 - Status Register Format

(C/‾D = 1)

The **Status Register** provides information about transmit/receive readiness, buffer state, and error conditions.  
Each bit corresponds directly to internal state or an external pin with the same name.

    Bit:   7      6        5     4     3      2      1       0
           DSR   SYNC/BRK  FE    OVE   PE    TxEMP  RxRDY   TxRDY

### Bit 0 - TxRDY (Transmit Ready)

Indicates whether the **transmit data buffer** can accept a new character.

| TxRDY | Meaning                           |
| ----- | --------------------------------- |
| 0     | Buffer full (cannot write)        |
| 1     | Buffer empty (ready for new data) |

This bit corresponds directly to the **TxRDY output pin**, except that the pin may also be masked by CTS and TxEN.

### Bit 1 - RxRDY (Receive Ready)

Asserted when the **receive data buffer** contains a character waiting to be read.

* Cleared automatically when CPU reads the receive buffer.

### Bit 2 - TxEMP (Transmit Empty)

Indicates whether **both** transmit buffers (first and second) are empty.

* 1 -> Both buffers empty
* 0 -> Transmission active or buffered data pending

### Bit 3 - PE (Parity Error)

| PE | Meaning               |
| -- | --------------------- |
| 0  | No parity error       |
| 1  | Parity error detected |

Occurs if incoming parity bit does not match the expected value.

### Bit 4 - OVE (Overrun Error)

| OVE | Meaning       |
| --- | ------------- |
| 0   | No overrun    |
| 1   | Overrun error |

Occurs if:

* CPU fails to read Rx buffer in time
* New received byte overwrites unread data

### Bit 5 - FE (Framing Error)

| FE | Meaning                |
| -- | ---------------------- |
| 0  | No framing error       |
| 1  | Framing error detected |

Occurs when receiver does not detect a valid stop bit.

### Bit 6 - SYNC/BRK

Meaning depends on operating mode:

* **Synchronous mode:** SYNC detection status
* **Asynchronous mode:** Break detection status

Set when a SYNC character is detected (sync mode), 
or when RxDATA remains low for an extended period (break).

### Bit 7 - DSR (DSR Input Pin State)

Reflects the real-time level of the **DSR pin**:

| DSR Bit | DSR Pin State |
| ------- | ------------- |
| 0       | DSR = 1       |
| 1       | DSR = 0       |

(*Active-low modem input.*)

### Summary Table - Status Register

| Bit | Name     | Meaning                                                 |
| --- | -------- | ------------------------------------------------------- |
| 7   | DSR      | DSR pin input state                                     |
| 6   | SYNC/BRK | Sync detected (sync mode) / Break detected (async mode) |
| 5   | FE       | Framing error                                           |
| 4   | OVE      | Overrun error                                           |
| 3   | PE       | Parity error                                            |
| 2   | TxEMP    | Transmit buffers empty                                  |
| 1   | RxRDY    | Receive buffer has unread data                          |
| 0   | TxRDY    | Transmit buffer ready for new data                      |
