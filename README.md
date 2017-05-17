# serial-meter

Serial communications protocol for the TekPower TP4000ZC digital
multimeter.

Written by Mark Mason and Robin Garner.  Feel free to use and copy,
credit is appreciated.  mason at porklips dot org.

Data is transferred from the meter at 2400 baud, 8 data bits, one
stop bit, no parity.

Instead of sending ASCII text showing what's on the display, the
meter sends data representing the segments of the LCD display,
which requires some translation to turn into a usable value.

"Packets" representing single samples are sent at regular intervals
when the meter is in RS232 mode.  It is not necessary to poll for
samples.  The samples are sent at approximately 1 second intervals
on the volts, amps, and ohms scale.  Capacitance will take longer,
depending on the capacitance - up 10 15 seconds for 100uf.  In
addition, a single zero byte is sent when the meter is turned on.

A packet is 13 or 14 bytes.  The upper 4 bits contains a value
between 1 and 0xE (14).  This shows the position of that byte
within the packet.  They are sent in sequence, but the first byte
is not sent in some cases.

The lower four bits of each byte represent either the segments of
the digits on the LCD display or attributes teling what mode the
meter is in, such as mV, kohms, hz, etc.  The 2nd through 9th bytes
represent LCD segments, the 1st and 10th through 14th bytes give
attribute information.

For example, if the display says: "04.71 k ohms RS232 AUTO", the
following bytes would be sent:

  27 3D 42 57 69 75 80 95 A2 B0 C4 D0 E8

In this case, the first byte (1x) is not sent.  '2x' through '9x'
represent the LCD segments displaying "04.71", and Ax through Ex
indicate that the meter is in "k ohms RS232 AUTO" mode.

To decode this, discard the first four bits of each byte, since
they are just for packet framing.  This leaves us with 13 four bit
values:

  7 D 2 7 9 5 0 5 2 0 4 0 8

There are eight bits per digit, so four digits would be grouped
like this:

  7D 27 95 05

These numbers represent segments of the LCD display.  The segments
are:

   -        A
  | |     F   B
   -        G
  | |     E   C
.  _        D

The segments correspond to the following bits:

  B = 1
  G = 2
  C = 3
  D = 4
  A = 5
  F = 6
  E = 7
  Decimal = 8 (or the negative sign on the first digit)

In the example above, where "7D 27 95 05" represents "04.71", the
first digit is hex 7D.  This is, in binary,

  0111 1101

Bits 1, 3, 4, 5, 6, and 7 are set, resresenting LCD segments B, C,
D, A, F, and E, or:

   A       -
 F   B    | |
             
 E   C    | |
   D       -

Which is 0.

Simple enough?

The remaining bits, in bytes 1x (when present) and Ax through Ex
(that is, the bytes starting with 1, and A through E) give the
meter mode.  More than one bit can be set.  The bits are as
follows:

11 - Unknown
12 - AUTO
14 - DC
18 - AC
A1 - Diode test
A2 - Kilo (k)
A4 - Nano (n)
A8 - Micro (u)
B1 - Audible Alert (sound waves)
B2 - Mega (M)
B4 - %
B8 - Milli (m)
C1 - Hold
C2 - Rel Delta (triangle)
C4 - Ohms (omega)
C8 - Farads (F)
D1 - Unknown
D2 - Hertz (Hz)
D4 - Volts (V)
D8 - Amps (A)
E1 - Unknown
E2 - Unknown
E4 - Degrees celcius
E8 - Unknown
 
In the 4.71 k ohms example above, the attributes are A2, C4, and
E8, which indicates that the mode is kilo ohms, with the unknown
E8.
