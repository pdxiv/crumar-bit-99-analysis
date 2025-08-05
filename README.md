# Crumar BIT 99 Synthesizer Analysis

This repository contains a comprehensive analysis of the Crumar BIT 99 synthesizer. The BIT 99 is a 6-voice polyphonic synthesizer from 1984 featuring full MIDI implementation, multiple operating modes, and sophisticated real-time control voltage generation.

## Documentation

### Technical Analysis

* **[Hardware Implementation](hardware_implementation.md)** - Complete hardware architecture, memory mapping, I/O registers, and interface specifications
* **[Software Implementation](software_implementation.md)** - Firmware functionality, MIDI implementation, voice management, and real-time processing algorithms

### User Documentation  

* **[User Manual](manual.md)** - Complete user manual for the Crumar BIT 99 synthesizer in markdown format

### Additional documentation

* **[8253 Datasheet](intel_8253_datasheet.md)** - Intel 8253 Programmable Interval Timer official datasheet

## Key Features

* **6-voice polyphonic synthesis** with dynamic voice allocation
* **Comprehensive MIDI implementation** including SysEx patch dumps
* **Multiple operating modes**: Single, Split, Double, and Park modes
* **99 program storage slots** with battery-backed memory
* **Cassette tape interface** for patch backup and transfer
* **Real-time control voltage generation** using time-multiplexed DAC and Sample & Hold circuits
* **Professional keyboard interface** with velocity sensitivity

## Architecture Highlights

* **Intel 8031 microcontroller** @ 12MHz with external ROM/RAM
* **4K battery-backed SRAM** for patch storage and system variables  
* **Single 8-bit DAC** generates all control voltages via time-multiplexing
* **Four 8253 timer chips** provide DCO frequency generation
* **Interrupt-driven MIDI and keybed processing** for low latency
* **Sophisticated voice management** with note priority and voice stealing
