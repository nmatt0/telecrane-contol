# Telecrane Control protocol

What a Telecrane-style remote puts on the air, as measured and as reproduced by
this app. Everything here is derived from captures of two physical remotes (on
channels 8 and 9) plus the transmit code in `crane_remote.c`. Treat the parts
marked *extrapolated* as unverified.

## Overview

- **Modulation:** 2-FSK, phase continuous.
- **Bit rate:** ~18.9 kbaud (about 53 us/bit).
- **Deviation:** ~8.7 kHz on channels below 9, ~9.5 kHz on 9 and above.
- **Framing:** no preamble/sync handling in the radio; the frame carries its own
  `0x55` preamble and a fixed 36-bit sync word. No whitening.
- **Integrity:** no checksum and no CRC. A frame is accepted on sync + structure
  alone.
- **Addressing:** each remote has a 24-bit ID (a serial) and lives on one of 16
  channels. The receiver is paired to one ID on one channel.

A command is a short frame repeated continuously at a fixed cadence for as long
as the operator holds the button. Releasing sends a brief idle tail (except for
STOP).

## Radio (CC1101)

The app programs the CC1101 from a 2-FSK async preset. The registers that define
the waveform:

| Register | Value | Meaning |
|----------|-------|---------|
| MDMCFG2  | 0x04  | 2-FSK, no preamble/sync inserted by the radio |
| MDMCFG3  | 0x7D  | `DRATE_M` = 125 |
| MDMCFG4  | 0xC9  | `CHANBW_E` = 3, `CHANBW_M` = 0 (RX BW ~101.6 kHz), `DRATE_E` = 9 |
| DEVIATN  | 0x23 / 0x24 | patched per channel (see below) |
| PKTCTRL0 | 0x32  | async serial, continuous, no whitening |
| IOCFG0   | 0x0D  | GDO0 = async serial data (used as sliced-data input on RX) |

Derived from those registers:

- **Data rate:** `DRATE = ((256 + DRATE_M) * 2^DRATE_E) / 2^28 * 26e6` =
  `(381 * 512) / 2^28 * 26e6` ≈ **18894 baud**.
- **RX bandwidth:** `26e6 / (8 * (4 + CHANBW_M) * 2^CHANBW_E)` =
  `26e6 / 256` ≈ **101.6 kHz**.
- **Deviation:** `f_dev = (8 + DEVIATN_M) * 2^DEVIATN_E * 26e6 / 2^17`.
  - `0x23` (M=3, E=2): `44 * 198.36` ≈ **8.73 kHz** (channels 0-8).
  - `0x24` (M=4, E=2): `48 * 198.36` ≈ **9.52 kHz** (channels 9-15).

## Channels

The channel number is the second character of the serial (`C` + nibble). The
plan was measured on channel 8 (848.934 MHz) and channel 9 (850.918 MHz); the
step between adjacent channels is **1.9842974 MHz**. The other 14 are
extrapolated from that step and have **not** been confirmed on-air.

```
freq(ch) = 848.9341155 MHz + (ch - 8) * 1.9842974 MHz
```

| Serial nibble | Channel | Frequency (MHz) | Deviation | Source |
|---------------|---------|-----------------|-----------|--------|
| C0 | 0  | 833.060 | 8.73 kHz | extrapolated |
| C1 | 1  | 835.044 | 8.73 kHz | extrapolated |
| C2 | 2  | 837.028 | 8.73 kHz | extrapolated |
| C3 | 3  | 839.013 | 8.73 kHz | extrapolated |
| C4 | 4  | 840.997 | 8.73 kHz | extrapolated |
| C5 | 5  | 842.981 | 8.73 kHz | extrapolated |
| C6 | 6  | 844.966 | 8.73 kHz | extrapolated |
| C7 | 7  | 846.950 | 8.73 kHz | extrapolated |
| C8 | 8  | 848.934 | 8.73 kHz | **measured** |
| C9 | 9  | 850.918 | 9.52 kHz | **measured** |
| CA | 10 | 852.903 | 9.52 kHz | extrapolated |
| CB | 11 | 854.887 | 9.52 kHz | extrapolated |
| CC | 12 | 856.871 | 9.52 kHz | extrapolated |
| CD | 13 | 858.856 | 9.52 kHz | extrapolated |
| CE | 14 | 860.840 | 9.52 kHz | extrapolated |
| CF | 15 | 862.824 | 9.52 kHz | extrapolated |

The split in deviation, frame period, and the 16-bit field base value all fall
at channel 9. The app picks the channel-8 profile for channels 0-8 and the
channel-9 profile for channels 9-15.

## Serial / addressing

A serial is 8 characters: `C` + a channel nibble + 6 decimal digits.

```
C 8 12 34 56
| |  \  \  \__ ID byte 2 (0-99)
| |   \  \____ ID byte 1 (0-99)
| |    \______ ID byte 0 (0-99)
| \___________ channel nibble, hex 0-F
\_____________ literal 'C'
```

The 6 digits are three decimal pairs. Each pair becomes one ID byte, so each ID
byte is limited to the range 0-99 (the decode step rejects any byte above 99,
which is how the sniffer tells a real frame from noise). The three bytes are the
24-bit ID transmitted in the frame.

Example: `C8123456` is channel 8, ID bytes `12 34 56` (`0x0C 0x22 0x38`).

## Frame structure

Bit 0 is the first bit of the sync word. The `0x55` preamble comes before it,
but its length varies with how much the demodulator recovers, so all offsets are
counted from the start of the sync word. From sync onward a frame is 120 bits:

| Bits | Field | Notes |
|------|-------|-------|
| before 0 | preamble | `0x55...` alternating pattern, ~77 bits, variable length |
| 0-35   | sync | 36-bit sync word `0x45BA85BA8` |
| 36-46  | gap zeros | 11 zero bits |
| 47     | marker | a single `1` bit |
| 48-50  | channel tag | varies by channel (ch8 = `000`, ch9 = `001`) |
| 51-74  | ID | 24-bit ID, MSB first, 3 bytes |
| 75-79  | button field | low 5 bits of the command mask |
| 80-95  | 16-bit field | `base + mask` (see Commands) |
| 96-119 | trailer | template tail |

Bits 36-50 are the 15-bit "gap" between the sync word and the ID: 11 zeros, the
marker, then 3 channel-dependent bits. The decoder verifies the 11 zeros and the
marker but skips the 3 channel bits (it already knows the channel from the
receive frequency).

The `base` for the 16-bit field is `0x0007` on channels 0-8 and `0x0006` on
channels 9-15. The transmitted value is `base + command_mask`.

## Commands

Each command is a 16-bit mask. It feeds the frame in two places: its low 5 bits
go into the button field at bits 75-79, and the full value is added to `base`
to form the 16-bit field at bits 80-95.

| Command | Mask |
|---------|--------|
| START | 0x0010 |
| STOP  | 0x0008 |
| UP    | 0x0004 |
| DOWN  | 0x0002 |
| NORTH | 0x2000 |
| SOUTH | 0x4000 |
| EAST  | 0x0001 |
| WEST  | 0x8000 |
| AUX 1 | 0x1010 |
| AUX 2 | 0x0808 |
| AUX 3 | 0x0404 |
| AUX 4 | 0x0202 |

These are the masks the app transmits. They were taken from the two remotes on
hand; other remote variants may map buttons differently.

## Timing

- **Frame period** (start of one frame to the next):
  - channels 0-8: ~19.94 ms
  - channels 9-15: ~30.02 ms
- **Hold:** frames repeat at the period above for as long as the button is held.
- **Idle tail:** on release the app sends ~50 idle frames (command mask 0), the
  way a real remote settles after a button. STOP is the exception: it sends no
  tail.

## Decoding (how the sniffer recovers a serial)

1. Hop to a channel, program its frequency and deviation, and receive in async
   mode with GDO0 as sliced data.
2. Capture level/duration edges and quantize them to bits at ~53 us each.
3. Slide a window looking for the 36-bit sync word `0x45BA85BA8`. That hit
   defines bit 0 for the rest of the frame.
4. On a sync hit, check the gap: the 11 zeros at bits 36-46 then the `1` marker
   at bit 47. Skip the 3 channel-dependent bits at 48-50.
5. Read bits 51-74 as 3 ID bytes. Reject the frame if any byte exceeds 99
   (it cannot be a valid decimal-pair ID, so it is noise).
6. Assemble the serial as `C` + the channel nibble (known from the receive
   frequency) + the three ID bytes as decimal pairs.

No checksum means step 5 (the 0-99 range test on all three bytes) is the only
structural filter beyond the sync word and the gap pattern.

## Known gaps

- Channels other than 8 and 9 are extrapolated from the measured step and have
  not been verified on-air.
- The 3 channel-tag bits at 48-50 are observed (`000` on ch8, `001` on ch9)
  but not fully characterized across all channels.
- The exact bit boundaries of the preamble and trailer are approximate; the
  sync word, gap, ID, and command fields are the parts that matter for
  transmit and decode.
- There is no checksum or rolling code. Capturing one frame is enough to clone
  a remote, which is the security-relevant takeaway.
