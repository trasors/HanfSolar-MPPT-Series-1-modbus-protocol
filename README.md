# ModBus protocol for cheap Aliexpress MPPT charge controller

I bought a cheap MPPT controller with bluetooth from Aliexpress. It looks to be the following model, but can also be seen branded as Mexxsun, Lexron etc.
http://www.hanfsolar.com/products_detail/productId=104.html

A great review can be found at https://www.youtube.com/watch?v=1wyP2qlu8AI

I wanted to be able to query the controller over Bluetooth for logging and graphing purposes, however the modbus register documentation for this device is none existant (or not easily found).

Below are the results of my findings. Please note, whilst these findings are useful for my purpose, they are based on observation and not conclusive, so use/treat with caution.

# Hardware
The bluetooth device in the controller is a DX-BT18 device. The device name is BT18 and the default password is 1234. There is a microswitch on the module that probably boots it into AT mode (not tested)

# Methodology
In order to get an idea of the protocol, I installed the Android app from the manufacturer website, enabled developer mode on the phone and switched on bluetooth debugging. I then began using the app to query the controller. Once I had gone through enough options, I saved the debug log (via ADB) and imported into Wireshark. It was fairly clear from Wireshark that the protocol used is similar to Modbus TCP.

## Further testing
More detailed testing was done using Termite (https://www.compuphase.com/software_termite.htm) because it easy allowed HEX messages to be sent. If using Windows, use the 2nd com port presented by the BT18 device (baud rate is unimportant)

# Packet format

## Request
| Transaction ID | Protocol ID | Length | Unit ID | Function code | Address | Number of registers |
| --- | --- | --- | --- | --- | --- | --- |
| 2 bytes | 2 bytes | 2 bytes | 1 byte | 1 byte | 2 bytes | 2 bytes |
| 00 01 | 00 00 | 00 06 | 01 | 03 | 00 00 | 00 01 |

## Response
| Transaction ID | Protocol ID | Length | Unit ID | Function code | Length in bytes | Data |
| --- | --- | --- | --- | --- | --- | --- |
| 2 bytes | 2 bytes | 2 bytes | 1 byte | 1 byte | 1 byte | n bytes |
| 00 01 | 00 00 | e.g. 00 05 | 01 | 03 | e.g. 02 | e.g. 00 08 |

# Modbus registers 

| Name | Address | Number of registers | Notes |
| --- | --- | --- | --- |
| Solar voltage (V) | 0x0001 | 0x0001 | 0x076c = 1900 = 19.0V |
| Battery voltage (V) | 0x0005 | 0x0001 | 0x0538 = 1320 = 13.2V |
| Temperature (C) | 0x0006 | 0x0001 | 0x0015 = 21 = 21C |
| Load (A) | 0x0008 | 0x0001 | 0x00aa = 170 = 1.7A |
| Load (on/off) | 0x0036 | 0x0001 | 0x0000 = on, 0x0008=off |
| Boost duration | 0x0020 | 0x0001 |  0x0078 (120ms) |
| Equalize duration	| 0x0021 | 0x0001 | 0x0078 (120ms) |
| Temp compensation	| 0x0022 | 0x0001 | 0xfffc = -4 |
| Charging limit | 0x0044 | 0x0001 | 0x0640 (16.00) |
| Discharging limit | 0x0045 | 0x0001 | 0x0384 (9.00) |
| Over volt disconnect | 0x0034 | 0x0001 | 0x0640 (16.00) |
| Over volt reconnect | 0x0035 | 0x0001 | 0x060e (15.50) | 
| Low volt disconnect | 0x0023 | 0x0001 | 0x0438 (10.80) |
| Low volt reconnect | 0x0024 | 0x0001 | 0x04ec (12.60) |
| Equalize charging | 0x001f | 0x0001 | 0x05b4 (14.60) |
| Boost charging | 0x001e | 0x0001 | 0x05a0 (14.40) |
| Float charging volt | 0x001d | 0x0001 | 0x0564 (13.80) |
| Boost recon charge | 0x0043 | 0x0001| 0x04ec (12.60) |


# RAW dump of registers
You can get a complete dump of registers by reading 123 registers from address 0000 e.g.

## Command
```
0x00010000000601030000007b
```

## Output
```
00 01 00 00 00 f9 01 03 f6 00 00 01  .....??..??...
21 00 00 00 00 00 00 04 d5 00 17 00  !.......??...
00 00 04 00 01 00 32 00 00 00 19 00  ......2.....
00 00 00 00 00 00 00 00 00 00 00 00  ............
00 00 00 00 00 00 00 00 00 00 00 00  ............
00 00 00 00 00 00 00 05 64 05 a0 05  ........d.??.
b4 00 78 00 78 ff fc 04 38 04 ec 00  ??.x.x????.8.??.
00 00 00 00 00 00 3c 00 00 00 00 00  ......<.....
00 00 00 00 00 01 f4 02 58 00 1e 00  ......??.X...
1e 00 00 00 00 06 40 06 0e 00 00 00  ......@.....
00 00 0c 00 14 00 14 00 00 00 00 00  ............
00 00 00 00 00 00 00 00 00 00 00 04  ............
ec 06 40 03 84 00 0c 00 0f 00 73 00  ??.@.???.....s.
65 00 69 00 72 00 65 00 53 00 20 00  e.i.r.e.S. .
31 00 54 00 50 00 50 00 4d 00 00 00  1.T.P.P.M...
00 00 00 00 00 00 00 00 00 00 00 00  ............
00 00 00 00 00 00 00 00 00 79 20 6f  .........y o
67 6f 6c 68 6e 65 63 20 54 6e 64 20  golhnec Tnd 
61 63 65 65 6e 63 69 20 53 65 69 6e  aceenci Sein
66 48 61 6e 20 68 61 57 75 00 00 00  fHan haWu...
64 05 a0 03 c0 00 00 00 00 00 00 00  d.??.??.......
00 00 00
```

# More RAW commands

Note, strings returned byte swapped and in reverse order

## Device model
```
0x000100000006010300480006
0x0001000000060103004e0006
->
MPPT1 Series
```

## Device version
```
0x000100000006010300460002
->
1.2 1.5
```

## Device manufacturer
```
0x000100000006010300600006
0x000100000006010300660006
0x0001000000060103006c0006
-> 
```
