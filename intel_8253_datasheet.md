# 8253/8253-5 PROGRAMMABLE INTERVAL TIMER

The Intel® 8253 is a programmable counter/timer device designed for use as an Intel microcomputer peripher-al. It uses NMOS technology with a single + 5V supply and is packaged in a 24-pin plastic DIP.

It is organized as 3 independent 16-bit counters, each with a count rate of up to 2.6 MHz. All modes of operation are software programmable.

* MCS-85T™ Compatible 8253-5
* 3 Independent 16-Bit Counters
* DC to 2.6 MHz
* Programmable Counter Modes
* Count Binary or BCD
* Single + 5V Supply
* Available in EXPRESS
  + Standard Temperature Range
  + Extended Temperature Range

## FUNCTIONAL DESCRIPTION

### General

The 8253 is programmable interval timer/counter specifically designed for use with the Intel™ Micro-computer systems. Its function is that of a general purpose, multi-timing element that can be treated as an array of I/O ports in the system software.

The 8253 solves one of the most common problems in any microcomputer system, the generation of ac-curate time delays under software control. Instead of setting up timing loops in systems software, the pro-grammer configures the 8253 to match his require-ments, initializes one of the counters of the 8253 with the desired quantity, then upon command the 8253 will count out the delay and interrupt the CPU when it has completed its tasks. It is easy to see that the software overhead is minimal and that multiple delays can easily be maintained by assignment of priority levels.

Other counter/timer functions that are non-delay in nature but also common to most microcomputers can be implemented with the 8253.

* Programmable Rate Generator
* Event Counter
* Binary Rate Multiplier
* Real Time Clock
* Digital One-Shot
* Complex Motor Controller

### Data Bus Buffer

The 3-state, bi-directional, 8-bit buffer is used to in-terface the 8253 to the system data bus. Data is transmitted or received by the buffer upon execution of INput or OUTput CPU instructions. The Data Bus Buffer has three basic functions.

1. Programming the MODES of the 8253.
2. Loading the count registers.
3. Reading the count values.

### Read/Write Logic

The Read/Write Logic accepts inputs from the sys-tem bus and in turn generates control signals for overall device operation. It is enabled or disabled by CS so that no operation can occur to change the function unless the device has been selected by the system logic.

#### /RD (Read)

A "low" on this input informs the 8253 that the CPU is inputting data in the form of a counters value.

#### /WR (Write)

A "low" on this input informs the 8253 that the CPU is outputting data in the form of mode information or loading counters.

#### A0, A1

These inputs are normally connected to the address bus. Their function is to select one of the three coun-ters to be operated on and to address the control word register for mode selection.

#### /CS (Chip Select)

A "low" on this input enables the 8253. No reading or writing will occur unless the device is selected. The /CS input has no effect upon the actual operation of the counters.

| /CS | /RD | /WR | A1 | A0 | Description          |
|-----|-----|-----|----|----|----------------------|
|   0 |   1 |   0 |  0 |  0 | Load Counter No. 0   |
|   0 |   1 |   0 |  0 |  1 | Load Counter No. 1   |
|   0 |   1 |   0 |  1 |  0 | Load Counter No. 2   |
|   0 |   1 |   0 |  1 |  1 | Write Mode Word      |
|   0 |   0 |   1 |  0 |  0 | Read Counter No. 0   |
|   0 |   0 |   1 |  0 |  1 | Read Counter No. 1   |
|   0 |   0 |   1 |  1 |  0 | Read Counter No. 2   |
|   0 |   0 |   1 |  1 |  1 | No-Operation 3-State |
|   1 |   X |   X |  X |  X | Disable 3-State      |
|   0 |   1 |   1 |  X |  X | No-Operation 3-State |

### Control Word Register

The Control Word Register is selected when A0, A1 are 11. It then accepts information from the data bus buffer and stores it in a register. The information stored in this register controls the operation MODE of each counter, selection of binary or BCD counting and the loading of each count register.

The Control Word Register can only be written into; no read operation of its contents is available.

### Counter #0, Counter #1, Counter #2

These three functional blocks are identical in opera-tion so only a single counter will be described. Each Counter consists of a single, 16-bit, pre-settable, DOWN counter. The counter can operate in either binary or BCD and its input, gate and output are con-figured by the selection of MODES stored in the Control Word Register.

The counters are fully independent and each can have separate MODE configuration and counting op-eration, binary or BCD. Also, there are special fea-tures in the control word that handle the loading of the count value so that software overhead can be minimized for these functions.

The reading of the contents of each counter is avail-able to the programmer with simple READ opera-tions for event counting applications and special commands and logic are included in the 8253 so that the contents of each counter can be read "on the fly" without having to inhibit the clock input.

## 8253 SYSTEM INTERFACE

The 8253 is a component of the IntelTM Microcom-puter systems and interfaces in the same manner as all other peripherals of the family. It is treated by the systems software as an array of peripheral I/O ports; three are counters and the fourth is a control register for MODE programming.

Basically, the select inputs A0, A1 connect to the A0, A1 address bus signals of the CPU. The CS can be derived directly from the address bus using a lin-ear select method. Or it can be connected to the output of a decoder, such as an Intel 8205 for larger systems.

## OPERATIONAL DESCRIPTION

### General

The complete functional definition of the 8253 is programmed by the systems software. A set of con-trol words must be sent out by the CPU to initialize each counter of the 8253 with the desired MODE and quantity information. Prior to initialization, the MODE, count, and output of all counters is unde-fined. These control words program the MODE, Loading sequence and selection of binary or BCD counting.

Once programmed, the 8253 is ready to perform whatever timing tasks it is assigned to accomplish.

The actual counting operation of each counter is completely independent and additional logic is pro-vided on-chip so that the usual problems associated with efficient monitoring and management of exter-nal, asynchronous events or rates to the microcom-puter system have been eliminated.

### Programming the 8253

All of the MODES for each counter are programmed by the systems software by simple I/O operations.

Each counter of the 8253 is individually programmed by writing a control word into the Control Word Register. (A0, A1 = 11)

### Control Word Format

|  D7 |  D6 |  D5 |  D4 |  D3 |  D2 |  D1 |  D0 |
|-----|-----|-----|-----|-----|-----|-----|-----|
| SC1 | SCO | RL1 | RLO |  M2 |  M1 |  M0 | BCD |

### Definition Of Control

#### SC - SELECT COUNTER:

| SC1 | SC0 |      Description |
|-----|-----|------------------|
|   0 |   0 | Select Counter 0 |
|   0 |   1 | Select Counter 1 |
|   1 |   0 | Select Counter 2 |
|   1 |   1 |          Illegal |

#### RL - READ/LOAD:

| RL1 | RL0 |                                                         Description |
|-----|-----|---------------------------------------------------------------------|
|   0 |   0 |      Counter Latching operation (see READ/WRITE Procedure Section). |
|   1 |   0 |                               Read/Load most significant byte only. |
|   0 |   1 |                              Read/Load least significant byte only. |
|   1 |   1 | Read/Load least significant byte first, then most significant byte. |

#### M - MODE:

| M2 | M1 | M0 |   Mode |
|----|----|----|--------|
|  0 |  0 |  0 | Mode 0 |
|  0 |  0 |  1 | Mode 1 |
|  X |  1 |  0 | Mode 2 |
|  X |  1 |  1 | Mode 3 |
|  1 |  0 |  0 | Mode 4 |
|  1 |  0 |  1 | Mode 5 |

#### BCD:

| 0 | Binary Counter 16-Bits |
|---|------------------------|
| 1 | Binary Coded Decimal (BCD) Counter (4 Decades) |

#### Counter Loading

The count register is not loaded until the count value is written (one or two bytes, depending on the mode selected by the RL bits), followed by a rising edge and a falling edge of the clock. Any read of the coun-ter prior to that falling clock edge may yield invalid data.

### MODE DEFINITION

**MODE 0: Interrupt on Terminal Count.** The output will be initially low after the mode set operation. After the count is loaded into the selected count register, the output will remain low and the counter will count. When terminal count is reached, the output will go high and remain high until the selected count regis-ter is reloaded with the mode or a new count is load-ed. The counter continues to decrement after termi-nal count has been reached.

Rewriting a counter register during counting results in the following:

1. Write 1st byte stops the current counting.
2. Write 2nd byte starts the new count.

**MODE 1: Programmable One-Shot.** The output will go low on the count following the rising edge of the gate input.

The output will go high on the terminal count. If a new count value is loaded while the output is low it will not affect the duration of the one-shot pulse until the succeeding trigger. The current count can be read at any time without affecting the one-shot pulse.

The one-shot is retriggerable, hence the output will remain low for the full count after any rising edge of the gate input.

**MODE 2: Rate Generator.** Divide by N counter. The

output will be low for one period of the input clock. The period from one output pulse to the next equals the number of input counts in the count register. If the count register is reloaded between output pulses the present period will not be affected, but the sub-sequent period will reflect the new value.

The gate input, when low, will force the output high. When the gate input goes high, the counter will start from the initial count. Thus, the gate input can be used to synchronize the counter.

When this mode is set, the output will remain high until after the count register is loaded. The output then can also be synchronized by software.

**MODE 3: Square Wave Rate Generator.** Similar to MODE 2 except that the output will remain high until one half the count has been completed (or even numbers) and go low for the other half of the count. This is accomplished by decrementing the counter by two on the falling edge of each clock pulse. When the counter reaches terminal count, the state of the output is changed and the counter is reloaded with the full count and the whole process is repeated.

If the count is odd and the output is high, the first clock pulse (after the count is loaded) decrements the count by 1. Subsequent clock pulses decrement the clock by 2. After timeout, the output goes low and the full count is reloaded. The first clock pulse (following the reload) decrements the counter by 3. Subsequent clock pulses decrement the count by 2 until timeout. Then the whole process is repeated. In this way, if the count is odd, the output will be high for (N + 1)/2 counts and low for (N-1)/2 counts.

In Modes 2 and 3, if a CLK source other than the system clock is used, GATE should be pulsed imme-diately following WR of a new count value.

**MODE 4: Software Triggered Strobe.** After the mode is set, the output will be high. When the count is loaded, the counter will begin counting. On termi-nal count, the output will go low for one input clock period, then will go high again.

If the count register is reloaded during counting, the new count will be loaded on the next CLK pulse. The count will be inhibited while the GATE input is low.

**MODE 5: Hardware Triggered Strobe.** The counter will start counting after the rising edge of the trigger input and will go low for one clock period when the terminal count is reached. The counter is retriggera-ble. The output will not go low until the full count after the rising edge of any trigger.

#### Gate Pin Operations Summary

| Signal Status Modes | Low Or Going Low | Rising | High |
|---------------------|------------------|--------|------|
| 0 | Disables counting | - | Enables counting |
| 1 | - | 1) Initiates counting 2) Resets output after next clock | - |
| 2 | 1) Disables counting 2) Sets output immediately high | 1) Reloads counter 2) Initiates counting | Enables counting |
| 3 | 1) Disables counting 2) Sets output immediately high | 1) Reloads counter 2) Initiates counting | Enables counting |
| 4 | Disables counting | - | Enables counting |
| 5 | - | Initiates counting | - |

## 8253 READ/WRITE PROCEDURE

### Write Operations

The systems software must program each counter of the 8253 with the mode and quantity desired. The programmer must write out to the 8253 a MODE control word and the programmed number of count register bytes (1 or 2) prior to actually using the se-lected counter.

The actual order of the programming is quite flexible. Writing out of the MODE control word can be in any sequence of counter selection, e.g., counter #0 does not have to be first or counter #2 last. Each counter's MODE control word register has a sepa-rate address so that its loading is completely se-quence independent. (SCO, SC1).

The loading of the Count Register with the actual count value, however, must be done in exactly the sequence programmed in the MODE control word (RLO, RL1). This loading of the counter's count reg-ister is still sequence independent like the MODE control word loading, but when a selected count reg-ister is to be loaded it must be loaded with the num-ber of bytes programmed in the MODE control word (RLO, RL1). The one or two bytes to be loaded in the count register do not have to follow the associated MODE control word. They can be programmed at any time following the MODE control word loading as long as the correct number of bytes is loaded in order.

All counters are down counters. Thus, the value loaded into the count register will actually be decre-mented. Loading all zeros into a count register will result in the maximum count (216 for Binary or 104 for BCD). In MODE 0 the new count will not restart until the load has been completed. It will accept one of two bytes depending on how the MODE control words (RLO, RL1) are programmed. Then proceed with the restart operation.

### Read Operations

In most counter applications it becomes necessary to read the value of the count in progress and make a computational decision based on this quantity. Event counters are probably the most common ap-plication that uses this function. The 8253 contains logic that will allow the programmer to easily read the contents of any of the three counters without disturbing the actual count in progress.

There are two methods that the programmer can use to read the value of the counters. The first meth-od involves the use of simple I/O read operations of the selected counter. By controlling the A0, A1 in-puts to the 8253 the programmer can select the counter to be read (remember that no read opera-tion of the mode register is allowed A0, A1-11). The only requirement with this method is that in order to assure a stable count reading the actual operation of the selected counter must be inhibited either by con-trolling the Gate input or by external logic that inhibits the clock input. The contents of the counter selected will be available as follows:

First I/O Read contains the least significant byte

(LSB).

Second I/O Read contains the most significant byte

(MSB).

Due to the internal logic of the 8253 it is absolutely necessary to complete the entire reading procedure. If two bytes are programmed to be read, then two bytes must be read before any loading WR command can be sent to the same counter.

### Read Operation Chart

| A1 | A0 | RD | Description        |
|----|----|----|--------------------|
|  0 |  0 |  0 | Read Counter No. 0 |
|  0 |  1 |  0 | Read Counter No. 1 |
|  0 |  0 |  1 | Read Counter No. 2 |
|  1 |  1 |  0 | Illegal            |

### Reading While Counting

In order for the programmer to read the contents of any counter without effecting or disturbing the count-ing operation the 8253 has special internal logic that can be accessed using simple /WR commands to the MODE register. Basically, when the programmer wishes to read the contents of a selected counter "on the fly" he loads the MODE register with a spe-cial code which latches the present count value into a storage register so that its contents contain an accurate, stable quantity. The programmer then is-sues a normal read command to the selected coun-ter and the contents of the latched register is available.

### MODE Register for Latching Count

A0, A1 = 11

|  D7 |  D6 |  D5 |  D4 |  D3 |  D2 |  D1 |  D0 |
|-----|-----|-----|-----|-----|-----|-----|-----|
| SC1 | SC0 |   0 |   0 |   X |   X |   X |   X |

* SC1, SC0: specify counter to be latched.
* D5, D4: 00 designates counter latching operation.
* X: don't care.

The same limitation applies to this mode of reading the counter as the previous method. That is, it is mandatory to complete the entire read operation as programmed. This command has no effect on the counter's mode.
