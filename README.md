# Telecrane Control (Flipper Zero)

A Flipper Zero app for Telecrane-style industrial crane radio remotes. It sniffs a
remote's serial off the air, saves it, and transmits forged command frames (2FSK,
around 848-851 MHz). Built against the ufbt / MNTM SDK.

## Status

Working end to end: the app sniffs a live remote, decodes its serial, saves it as
a new RC, and transmits frames the receiver acts on. Tested on two channels (ch8
and ch9); other channels are extrapolated from the measured channel step and have
not been verified on-air.

## Features

- **My RCs** - saved remotes. Pick one to open a hold-to-talk transmit screen. Hold
  OK on a command (start, stop, up, down, north, south, east, west, aux 1-4) to send
  frames at the real cadence; release to stop. On release the app sends a short idle
  tail like a real remote, except for STOP.
- **New RC from Serial** - type a serial and a name, save it.
- **Sniff RC** - hop the channels in receive, decode a live remote, extract its
  serial, and save it.
- Remotes persist across reboots in the app's config file.

## Protocol

2FSK, 18.9 kbaud, phase continuous, no checksum. Each frame is about 197 bits:

```
[0x55 preamble ~77 bits][sync 45BA85BA8][constant][24-bit ID][button mask][trailer]
```

The serial is `C` + a channel nibble (C0-CF) + 6 decimal digits. The 6 digits are 3
decimal pairs that become the 3 ID bytes (for example `C8123456` is channel 8 with
ID bytes 12, 34, 56).

Channel plan: ch8 at 848.934 MHz and ch9 at 850.918 MHz were measured directly; the
channel step is 1.9843 MHz and the other channels are extrapolated from it.

See [PROTOCOL.md](PROTOCOL.md) for the full writeup: the CC1101 register values, the
16-channel frequency table, the serial/addressing format, the bit-level frame layout,
the command masks, and the timing.

## Build

Install [ufbt](https://github.com/flipperdevices/flipperzero-ufbt), then:

```
ufbt
```

The fap lands in `dist/crane_remote.fap`.

## Install

With a Flipper connected over USB:

```
ufbt launch
```

This builds the app, uploads it, and launches it once. To have it show up under
**Apps > Sub-GHz > Telecrane Control**, copy `dist/crane_remote.fap` into the Flipper's
`/ext/apps/Sub-GHz/` folder with qFlipper or the ufbt storage helper.

## Debug logging

The app writes a per-session trace to `/ext/apps_data/crane_remote/debug.log`
(truncated at each start). Pull it with ufbt's storage helper:

```
python3 ~/.ufbt/current/scripts/storage.py -p <port> read /ext/apps_data/crane_remote/debug.log
```

Set `CRANE_SNIFF_DEBUG` to 1 at the top of `crane_remote.c` to add per-channel
capture statistics and a frame hex dump, useful when characterizing a new remote.

## Legal

For use on your own equipment, or equipment you are authorized to operate, only.
The transmit path uses frequencies that are licensed in most regions. You are
responsible for complying with the radio regulations where you operate.

## License

MIT. See [LICENSE](LICENSE).
