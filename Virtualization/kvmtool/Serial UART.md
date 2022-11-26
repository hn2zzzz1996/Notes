# Serial UART, an introduction

UART: universal asynchronous receiver / transmitter.

A UART is performing the main task in serial communications with computers.

## Serial UART types

Serial communication on PC compatibles started with the 8250 UART in the IBM XT. In the years after, new family members were introduced like the 8250A and 8250B revisions and the 16450. 

The main problem with the original series was the need to perform a software action for each single byte to transmit or receive. To overcome this problem, the 16550 was released which contained two on-board FIFO buffers, each capable of storing 16 bytes. One buffer for incoming, and one buffer for outgoing bytes.

The 16550A is the most common UART at this moment. Newer versions are under development, including the 16650 which contains two 32 byte FIFO’s and on board support for software flow control. 

## Registers

The following table shows the where the register can be found. The `base` address used in the table is `the lowest I/O port number` assigned.

| I/O port | Read (DLAB=0)                    | Write (DLAB=0)              | Read (DLAB=1)                    | Write (DLAB=1)            |
| -------- | -------------------------------- | --------------------------- | -------------------------------- | ------------------------- |
| base     | **RBR** receiver buffer          | **THR** transmitter holding | **DLL** divisor latch LSB        | **DLL** divisor latch LSB |
| base+1   | **IER** interrupt enable         | **IER** interrupt enable    | **DLM** divisor latch MSB        | **DLM** divisor latch MSB |
| base+2   | **IIR** interrupt identification | **FCR** FIFO control        | **IIR** interrupt identification | **FCR** FIFO control      |
| base+3   | **LCR** line control             | **LCR** line control        | **LCR** line control             | **LCR** line control      |
| base+4   | **MCR** modem control            | **MCR** modem control       | **MCR** modem control            | **MCR** modem control     |
| base+5   | **LSR** line status              | *factory test*              | **LSR** line status              | *factory test*            |
| base+6   | **MSR** modem status             | *not used*                  | **MSR** modem status             | *not used*                |
| base+7   | **SCR** scratch                  | **SCR** scratch             | **SCR** scratch                  | **SCR** scratch           |

There are 12 registers controlled by the UART. Each register is 8 bits wide.

### 1. RBR: Receiver buffer register (RO)

The RBR contains the byte received if no FIFO is used, or the oldest byte with FIFO's. If FIFO buffering is used, each new read action of the register will return the next byte, until no more bytes are present.

`Bit 0 in the LSR` line status register can be used to check if all received bytes have been read. This bit will change to zero if no more bytes are present.

### 2. THR: Transmitter holding register (WO)

The THR is used to buffer outgoing characters. If no FIFO buffering is used, `only one` character can be stored. Otherwise the amount of characters depends on the type of UART.

`Bit 5 in the LSR`, line status register can be used to check if new information must be written to THR. The value 1 indicates that the register is empty. If FIFO buffering is used, more than one character can be written to the transmitter holding register when the bit signals an empty state. There is no indication of the amount of bytes currently present in the transmitter FIFO.

The transmitter holding register is not used to transfer the data directly. The byte is first transferred to a shift register where the information is broken in single bits which are sent one by one.

### 3. IER: Interrupt enable register (R/W)

The smartest way to perform serial communications on a PC is using interrupt driven routines. In that configuration, it is not necessary to poll the registers of the UART periodically for state changes. The UART will signal each change by generating a processor interrupt. A software routine must be present to handle the interrupt and to check what state change was responsible for it.

Interrupts are not generated, unless the UART is told to do so. This is done by setting bits in the IER, interrupt enable register. A bit value 1 indicates, that an interrupt may take place.

|  Bit  | Description                          |
| :---: | ------------------------------------ |
| **0** | Received data available              |
| **1** | Transmitter holding register empty   |
| **2** | Receiver line status register change |
| **3** | Modem status register change         |
| **4** | Sleep mode (16750 only)              |
| **5** | Low power mode (16750 only)          |
| **6** | reserved                             |
| **7** | reserved                             |

### 4. IIR: Interrupt identification register (RO)

A UART is capable of generating a processor interrupt when a state change on the communication device occurs. One interrupt signal is used to call attention. This means, that additional information is needed for the software before the necessary actions can be performed. The IIR, interrupt identification register is helpful in this situation. Its bits show the current state of the UART and which state change caused the interrupt to occur.

|    Bit    |    Value     | Description                        | Reset by              |
| :-------: | :----------: | ---------------------------------- | --------------------- |
|   **0**   | xxxxxxx**0** | Interrupt pending                  | –                     |
|           | xxxxxxx**1** | No interrupt pending               | –                     |
| **3,2,1** | xxxx**000**x | Modem status change                | MSR read              |
|           | xxxx**001**x | Transmitter holding register empty | IIR read or THR write |
|           | xxxx**010**x | Received data available            | RBR read              |
|           | xxxx**011**x | Line status change                 | LSR read              |
|           | xxxx**110**x | Character timeout (16550)          | RBR read              |
|   **4**   | xxx**0**xxxx | Reserved                           | –                     |
|   **5**   | xx**0**xxxxx | Reserved (8250, 16450, 16550)      | –                     |
|           | xx**1**xxxxx | 64 byte FIFO enabled (16750)       | –                     |
|  **7,6**  | **00**xxxxxx | No FIFO                            | –                     |
|           | **01**xxxxxx | Unusable FIFO (16550 only)         | –                     |
|           | **11**xxxxxx | FIFO enabled                       | –                     |

### 5. FCR : FIFO control register (WO)

The FCR, FIFO control register is present starting with the 16550 series. This register controls the behavior of the FIFO’s in the UART. If a logical value 1 is written to bits 1 or 2, the function attached is triggered. The other bits are used to select a specific FIFO mode.

|   Bit   |    Value     | Description                             |
| :-----: | :----------: | --------------------------------------- |
|  **0**  | xxxxxxx**0** | Disable FIFO’s                          |
|         | xxxxxxx**1** | Enable FIFO’s                           |
|  **1**  | xxxxxx**0**x | –                                       |
|         | xxxxxx**1**x | Clear receive FIFO                      |
|  **2**  | xxxxx**0**xx | –                                       |
|         | xxxxx**1**xx | Clear transmit FIFO                     |
|  **3**  | xxxx**0**xxx | Select DMA mode 0                       |
|         | xxxx**1**xxx | Select DMA mode 1                       |
|  **4**  | xxx**0**xxxx | Reserved                                |
|  **5**  | xx**0**xxxxx | Reserved (8250, 16450, 16550)           |
|         | xx**1**xxxxx | Enable 64 byte FIFO (16750)             |
|         |              | *Receive FIFO interrupt trigger level*: |
| **7,6** | **00**xxxxxx | 1 byte                                  |
|         | **01**xxxxxx | 4 bytes                                 |
|         | **10**xxxxxx | 8 bytes                                 |
|         | **11**xxxxxx | 14 bytes                                |

### 6. LCR: Line control register (R/W)

The LCR, line control register is used at initialization to set the communication parameters. Parity and number of data bits can be changed for example. The register also controls the accessibility of the DLL and DLM registers. These registers are mapped to the same I/O port as the RBR, THR and IER registers. Because they are only accessed at initialization when no communication occurs this register swapping has no influence on performance.

|    Bit    |    Value     | Description                                                  |
| :-------: | :----------: | ------------------------------------------------------------ |
|  **1,0**  | xxxxxx**00** | 5 data bits                                                  |
|           | xxxxxx**01** | 6 data bits                                                  |
|           | xxxxxx**10** | 7 data bits                                                  |
|           | xxxxxx**11** | 8 data bits                                                  |
|   **2**   | xxxxx**0**xx | 1 stop bit                                                   |
|           | xxxxx**1**xx | 1.5 stop bits (5 bits data word) 2 stop bits (6, 7 or 8 bits data word) |
| **5,4,3** | xxxx**0**xxx | No parity                                                    |
|           | xx**001**xxx | Odd parity                                                   |
|           | xx**011**xxx | Even parity                                                  |
|           | xx**101**xxx | High parity (stick)                                          |
|           | xx**111**xxx | Low parity (stick)                                           |
|   **6**   | x**0**xxxxxx | Break signal disabled                                        |
|           | x**1**xxxxxx | Break signal enabled                                         |
|   **7**   | **0**xxxxxxx | DLAB : RBR, THR and IER accessible                           |
|           | **1**xxxxxxx | DLAB : DLL and DLM accessible                                |

Some remarks about parity:

The UART is capable of generating a trailing bit at the end of each data-word which can be used to check some data distortion. Because only one bit is used, the parity system is capable of detecting only an odd number of false bits. If an even number of bits has been flipped, the error will not be seen.

When even parity is selected, the UART assures that the number of high bit values in the sent or received data is always even. Odd parity setting does the opposite. Using stick parity has very little use. It sets the parity bit to always 1, or always 0.

Common settings are:

- 8 data bits, one stop bit, no parity
- 7 data bits, one stop bit, even parity

### 7. MCR: Modem control register (R/W)

The MCR, modem control register is used to perform handshaking actions with the attached device. In the original UART series including the 16550, setting and resetting of the control signals must be done by software. The new 16750 is capable of handling flow control automatically, thereby reducing the load on the processor.

|  Bit  | Description                   |
| :---: | ----------------------------- |
| **0** | Data terminal ready           |
| **1** | Request to send               |
| **2** | Auxiliary output 1            |
| **3** | Auxiliary output 2            |
| **4** | Loopback mode                 |
| **5** | Autoflow control (16750 only) |
| **6** | Reserved                      |
| **7** | Reserved                      |

The two auxiliary outputs are user definable. Output 2 is sometimes used in circuitry which controls the interrupt process on a PC. Output 1 is normally not used, however on some I/O cards, it controls the selection of a second oscillator working at 4 MHz. This is mainly for MIDI purposes.

### 8. LSR: Line status register (RO)

The LSR, line status register shows the current state of communication. Errors are reflected in this register. The state of the receive and transmit buffers is also available.

|  Bit  | Description                    |
| :---: | ------------------------------ |
| **0** | Data available                 |
| **1** | Overrun error                  |
| **2** | Parity error                   |
| **3** | Framing error                  |
| **4** | Break signal received          |
| **5** | THR is empty                   |
| **6** | THR is empty, and line is idle |
| **7** | Errornous data in FIFO         |

Bit 5 and 6 both show the state of the transmitting cycle. The difference is, that bit 5 turns high as soon as the transmitter holding register is empty whereas bit 6 indicates that also the shift register which outputs the bits on the line is empty.

### 9. MSR : Modem status register (RO)

The MSR, modem status register contains information about the four incoming modem control lines on the device. The information is split in two nibbles. The four most significant bits contain information about the current state of the inputs where the least significant bits are used to indicate state changes. The four LSBs are reset, each time the register is read.

|  Bit  | Description                  |
| :---: | ---------------------------- |
| **0** | change in Clear to send      |
| **1** | change in Data set ready     |
| **2** | trailing edge Ring indicator |
| **3** | change in Carrier detect     |
| **4** | Clear to send                |
| **5** | Data set ready               |
| **6** | Ring indicator               |
| **7** | Carrier detect               |

### 10. SCR : Scratch register (R/W)

The SCR, scratch register was not present on the 8250 and 8250B UART. It can be used to store one byte of information. In practice, it has only limited use. The only real use I know of is checking if the UART is a 8250/8250B, or a 8250A/16450 series. Because the 8250 series are only found in XTs even this use of the register is not commonly seen anymore.

### 11. DLL and DLM : Divisor latch registers (R/W)

For generating its timing information, each UART uses an oscillator generating a frequency of about 1.8432 MHz. This frequency is divided by 16 to generate the time base for communication. Because of this division, the maximum allowed communication speed is 115200 bps. Modern UARTS like the 16550 are capable of handling higher input frequencies up to 24 MHz which makes it possible to communicate with a maximum speed of 1.5 Mbps. On PC’s higher frequencies than the 1.8432 MHz are rarely seen because this would be software incompatible with the original XT configuration.

This 115200 bps communication speed is not suitable for all applications. To change the communication speed, the frequency can be further decreased by dividing it by a programmable value. For very slow communications, this value can go beyond 255. Therefore, the divisor is stored in two separate bytes, the divisor latch registers DLL and DLM which contain the least, and most significant byte.

For error free communication, it is necessary that both the transmitting and receiving UART use the same time base. Default values have been defined which are commonly used. The table shows the most common values with the appropriate settings of the divisor latch bytes. Note that these values only hold for a PC compatible system where a clock frequency of 1.8432 MHz is used.

| Speed (bps) | Divisor | DLL  | DLM  |
| ----------: | ------: | :--: | ---- |
|      **50** |   2,304 | 0x00 | 0x09 |
|     **300** |     384 | 0x80 | 0x01 |
|   **1,200** |      96 | 0x60 | 0x00 |
|   **2,400** |      48 | 0x30 | 0x00 |
|   **4,800** |      24 | 0x18 | 0x00 |
|   **9,600** |      12 | 0x0C | 0x00 |
|  **19,200** |       6 | 0x06 | 0x00 |
|  **38,400** |       3 | 0x03 | 0x00 |
|  **57,600** |       2 | 0x02 | 0x00 |
| **115,200** |       1 | 0x01 | 0x00 |

## Reference

https://www.lammertbies.nl/comm/info/serial-uart