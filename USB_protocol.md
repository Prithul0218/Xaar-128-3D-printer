# Xaar 128 Printhead Fire Protocol Datasheet

## Version 1.0
**Date:** December 12, 2025  
**Author:** Prithul & Grok 4  

This datasheet defines a lightweight, packet-based binary protocol for sending fire commands to control the 128 nozzles of a Xaar 128 printhead via an Arduino Nano over UART. The protocol is designed for integration with a Raspberry Pi running Klipper, where the Pi sends nozzle dispensing data to the Arduino. It emphasizes reliability, low overhead, and error detection for real-time 3D printing applications.  

Currently, the protocol supports only the "fire nozzles" command, but it is extensible for future commands (e.g., configuration or status queries). UART configuration: 115200 baud, 8 data bits, no parity, 1 stop bit (8N1).  

## Protocol Overview
- **Transport:** UART (serial communication).  
- **Packet Type:** Fixed-length for fire command (24 bytes total).  
- **Endianness:** Little-endian for multi-byte fields (e.g., sequence ID).  
- **Framing:** Start and end markers for synchronization.  
- **Error Detection:** CRC-8 checksum.  
- **Addressing:** Device ID for multi-device support.  
- **Reliability Features:** Sequence ID for detecting drops/duplicates; checksum for corruption.  
- **Payload:** 128-bit bitmask (16 bytes) representing nozzle states (1 = fire, 0 = idle).  
- **Flow Control:** Recommended software ACK/NACK responses from receiver (not detailed in this version).  

Packets are sent from the host (Raspberry Pi/Klipper) to the device (Arduino Nano). The device processes valid packets by shifting the payload into the Xaar 128's data registers and issuing a FIRE pulse.

## Packet Structure
The packet is a fixed 24-byte binary structure for the fire command. Below is a table detailing each field:

| Byte Offset | Field Name      | Size (Bytes) | Value/Example (Hex) | Description |
|-------------|-----------------|--------------|---------------------|-------------|
| 0          | Start Marker   | 1            | `AA`               | Fixed synchronization byte to indicate the start of a packet. Alternating bit pattern (10101010b) aids in bit-level sync and noise resistance. |
| 1          | Command Type   | 1            | `01`               | Identifies the command. Currently fixed to `0x01` for "fire nozzles". Other values are reserved for future extensions (e.g., `0x02` for waveform config). Invalid commands should be discarded. |
| 2          | Device ID      | 1            | `01` (example)     | User-defined address (0x00 to 0xFF) for targeting specific devices. The receiver checks if this matches its configured ID; if not, discard the packet silently. |
| 3-4        | Sequence ID    | 2            | `00 01` (example, decimal 1) | Unsigned 16-bit integer (0-65535, wraps around). Incremented by the sender for each packet. Receiver uses this to detect dropped, duplicated, or out-of-order packets. |
| 5          | Data Length    | 1            | `10` (16 decimal)  | Length of the payload in bytes. For command `0x01`, this must be `0x10` (16). Allows for variable-length payloads in future commands (max 255 bytes with 1-byte field). Receiver validates this value. |
| 6-21       | Payload        | 16           | `00 00 ... 00` (example, all off) | 128-bit bitmask for nozzle states. Each bit represents one nozzle:<br>- Bit 0 (LSB of byte 6): Nozzle 0<br>- Bit 7 (MSB of byte 6): Nozzle 7<br>- ...<br>- Bit 7 (MSB of byte 21): Nozzle 127<br>1 = Fire (dispense ink), 0 = Idle. Packed densely; no padding. For advanced features like drop size, bits could be repurposed (e.g., 2 bits per nozzle), but this increases payload size. |
| 22         | Checksum       | 1            | `XX` (calculated)  | CRC-8-CCITT of bytes 1 through 21 (command type to end of payload). See Checksum Calculation section for details. Receiver recomputes and compares; mismatch indicates corruption—discard and optionally send NACK. |
| 23         | End Marker     | 1            | `55`               | Fixed synchronization byte to indicate the end of a packet. Alternating bit pattern (01010101b) complements the start marker. |

**Total Packet Length:** 24 bytes.  
**Transfer Time Estimate:** Approximately 2 ms at 115200 baud (including overhead).

## Field Descriptions
- **Start/End Markers:** Chosen for their bit patterns to minimize false positives in noisy environments. If markers appear in payload data (rare but possible), consider adding byte stuffing (e.g., escape with 0x7D) in future revisions. Alternatives like 0x7E (HDLC-style) could be adopted for better convention alignment.  
- **Command Type:** Extensible; receiver should ignore unknown types or respond with an error ACK.  
- **Device ID:** Enables daisy-chaining or multi-head setups. Default to 0x01 if not using multiples.  
- **Sequence ID:** Sender starts at 0 or 1; receiver tracks the last valid ID and flags discrepancies (e.g., gap >1 indicates drop).  
- **Data Length:** Fixed for this command but included for protocol flexibility.  
- **Payload:** Directly maps to Xaar 128's serial data input. Arduino shifts this out via CLK and DATA pins, then latches and fires. Refer to Xaar 128 datasheet for exact timing (e.g., 1-10 MHz clock rate).  
- **Checksum:** Provides basic integrity; not for security (no authentication).  

## Checksum Calculation
The checksum uses CRC-8-CCITT with the following parameters:  
- **Polynomial:** 0x07 (x^8 + x^2 + x + 1)  
- **Initial Value:** 0x00  
- **Final XOR:** 0x00 (none)  
- **Input Reflection:** No  
- **Output Reflection:** No  

Compute the CRC over bytes 1 to 21 (command type through payload).  

### Algorithm (Bit-Wise Implementation)
For efficiency on microcontrollers like Arduino Nano:  
```
uint8_t calculateCRC8(const uint8_t* data, size_t length) {
    uint8_t crc = 0x00;  // Initial value
    for (size_t i = 0; i < length; i++) {
        uint8_t byte = data[i];
        for (uint8_t j = 0; j < 8; j++) {
            uint8_t bit = (crc >> 7) ^ (byte >> (7 - j));
            crc = (crc << 1) | bit;
            if (bit) {
                crc ^= 0x07;  // Polynomial
            }
        }
    }
    return crc;
}
```  
- **Input Data:** Array of bytes from offset 1 to 21.  
- **Example:** For a packet with command `0x01`, device `0x01`, seq `0x0001` (bytes `00 01`), length `0x10`, payload all `0x00`:  
  - Data for CRC: `01 01 00 01 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` (21 bytes)  
  - Calculated CRC: `0xA9` (verify with your own implementation).  

Full example packet (hex): `AA 01 01 00 01 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 A9 55`

## Implementation Notes
- **Sender (Raspberry Pi/Klipper):** Use Python's `serial` library. Compute sequence ID and CRC before sending. Pace packets based on ACKs to avoid overflows.  
- **Receiver (Arduino Nano):** Use `Serial` library. Implement a state machine: Hunt for `0xAA`, read 23 more bytes, validate end marker, length, command, device ID, and CRC. If valid, process payload. Buffer size: Ensure RX buffer (64 bytes) isn't overrun—read promptly.  
- **Error Handling:**  
  - Invalid marker/checksum/length: Discard and resync on next `0xAA`.  
  - Sequence gap: Optionally log or request resend via NACK packet (e.g., 4-byte response: `AA FF XX 55`, where `XX` is error code).  
  - Overruns: Use flow control (e.g., XON/XOFF) if high throughput.  
- **Extensions:** Add ACK/NACK responses, variable commands, or compression for sparse payloads (e.g., RLE).  
- **Testing:** Start with loopback tests on UART. Monitor with logic analyzer for timing. Ensure compatibility with Xaar 128's electrical requirements (e.g., 3.3V/5V logic, high-voltage drivers separate).  
- **Limitations:** No encryption/authentication; assumes trusted link. Not suitable for long-distance or high-noise environments without shielding.  

This protocol is optimized for simplicity and performance on resource-constrained hardware. For revisions or additions, contact the author.
