# GameSir Cyclone 2: firmware reverse engineering and an XInput bridge

Two related pieces of work on the GameSir Cyclone 2, a Bluetooth/2.4G game
controller built on a JieLi BR23 SoC (pi32v2 core).

**[`firmware/`](firmware/)** documents the reverse engineering: the image
layout, the instruction encodings, the timer subsystem, and four firmware
patches that add capabilities the stock firmware does not expose. It also
documents four rendering defects in a public Ghidra processor module, one of
which silently corrupts global addresses.

**[`bridge/`](bridge/)** is a Raspberry Pi that presents the pad to Windows as
an Xbox 360 Wireless Receiver, giving native XInput with working rumble and
battery. Nothing is installed on the PC. Getting output reports to flow at all
required patching the kernel's `f_fs` module.

The two halves are independent. The bridge works on stock firmware; the
firmware patches are useful without the bridge.

## Why I did any of this

I wanted my gaming PC to behave like a console, and I wanted Home Assistant to
be the thing that made it happen.

The PC is headless and lives in one room with everything else. I play from bed
over Moonlight most of the time. What I was after is the console experience:
pick up the controller and the machine wakes, put it down and everything powers
off on its own, without me thinking about any of it.

I tried the usual routes first and they were dead ends. Replacing the Windows
shell with a frontend is a reimage waiting to happen, and the one time people
report doing it on a machine that is also a workstation, the fix was a full OS
reinstall. GlosSI, which everyone still recommends, was archived in late 2025.
So the console feel had to come from outside Windows, which is where Home
Assistant and a Raspberry Pi came in.

That turned into a specific list of things the controller had to do, and each
one is why some piece of this repository exists:

| what I wanted | what it forced |
|---|---|
| controller turns on, PC wakes | a Pi in-line on USB that stays powered when the PC is off, sending Wake-on-LAN |
| PC sleeps, controller turns off | a firmware patch, because no GameSir pad has a host-driven power-off. The Cyclone 2 does have a writable inactivity-timeout register, so the power-off is indirect: write 1 minute when the PC sleeps, 20 when it wakes |
| battery in Home Assistant | the amber battery work, so the pad reports charge over Bluetooth and the Pi can publish it to MQTT and alert me before it dies mid-session |
| works in Steam Big Picture | the Xbox 360 receiver emulation, because Big Picture wants XInput and a DualShock identity was not good enough for me |
| rumble | the kernel patch, the hardest part of the whole project |

The Pi has to be independently powered or none of this works. It is fed from
the wall through a USB-C OTG splitter and passes gadget data through to the PC,
so it survives a full shutdown and can still wake the machine. My first setup
had it powered from the PC's USB port, so it died every time the PC did, which
defeats the point when its job is turning the PC back on.

None of the firmware work was the goal. I ended up in the firmware because the
features I wanted did not exist and there was no other way to find out why.

## Why the pad is interesting

It exposes six different personalities depending on a mode switch, each with its
own Bluetooth address, report format, and set of working features:

| mode | rate | rumble | battery | LED | audio | notes |
|---|---|---|---|---|---|---|
| amber, BT DInput | 481 Hz | yes | after patch | after patch | no | fastest mode |
| 2.4G dongle | 166 Hz, 230 after patch | yes | in report | yes | yes | 7.3x the payload |
| Switch Pro | 462 Hz | native | native | yes | no | loses analog triggers |
| PS4 / DS4 | not measured | yes | yes | yes | no | Steam reads it natively |
| BT XInput | not measured | yes | no | yes | no | |
| BT Steam | not measured | yes | no | yes | no | |

Audio only works over 2.4G. The SBC codec setup is gated on the mode variable,
so the headphone jack cannot work in any of the Bluetooth modes.

The "2.4GHz dongle" is itself a second BR23 speaking ordinary Bluetooth, so a
Raspberry Pi can stand in for it. That is how the LED, rumble and audio paths
were explored without a spare dongle.

Most of what follows is about closing the gaps in that table.

Because the Bluetooth address differs per mode, the pad can stay bonded to one
host in amber and another host in a different mode at the same time, and you
switch between them with the mode combo instead of re-pairing. That is
useful: the Pi keeps the amber bond for wake and battery, while the PC can
hold a direct bond in another mode for desk play.

## Disclaimer

This material is provided as is, without warranty of any kind, express or
implied. It is published for research, interoperability and educational
purposes.

**You use it entirely at your own risk. I accept no responsibility or liability
for any damage, data loss, bricked hardware, voided warranty, loss of function,
or any other harm arising from the use or misuse of anything in this
repository.**

Specifically, be aware that:

- Writing to a device's flash can render it unusable. The procedures here
  include verification gates and rollback images for a reason, and skipping
  them is how devices die.
- Modifying firmware will almost certainly void your warranty, and may breach
  the terms of sale or licence for your device.
- Values documented here were derived from one device on one firmware revision.
  A different revision may place things elsewhere, and applying an offset
  blindly can corrupt an image that was otherwise fine.
- Nothing here is legal advice. Laws covering reverse engineering, circumvention
  and modification differ by country, and it is your responsibility to know what
  applies to you.

Do not apply any of this to hardware you cannot afford to lose, and always keep
a verified stock dump before writing anything. See
[`firmware/README.md#flashing`](firmware/README.md#flashing).

## Credits

This work sits on top of other people's:

| project | author | used for |
|---|---|---|
| [ghidra-jieli](https://github.com/quarkslab/ghidra-jieli) | quarkslab, derived from Kagaimiq's original | pi32v2 disassembly |
| [ghidra-jieli](https://github.com/kagaimiq/ghidra-jieli) | Kagaimiq | the original processor module |
| [jl-misctools](https://github.com/kagaimiq/jl-misctools) | Kagaimiq | firmware decrypt/unpack, flashing over the loader |
| [jielie](https://github.com/kagaimiq/jielie) | Kagaimiq | JieLi architecture documentation |
| [fw-AC63_BT_SDK](https://github.com/Jieli-Tech/fw-AC63_BT_SDK) | Jieli-Tech | vendor SDK, used as cross-reference |
| [x360-research](https://github.com/InvoxiPlayGames/x360-research) | InvoxiPlayGames | Xbox 360 USB protocol and descriptor dumps |
| [MS-XUSBI](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-xusbi/) | Microsoft | the XUSB specification |
| [Ghidra](https://ghidra-sre.org/) | NSA | decompilation |

Decryption in particular is not original work here. Kagaimiq's tools do it, and
the notes in `firmware/` describe the cipher as a finding rather than shipping a
reimplementation.

## Licence

Three different kinds of thing live here and they cannot all carry the same
licence.

| what | licence | why |
|---|---|---|
| original code and documentation | [MIT](LICENSE) | mine to license |
| [`bridge/kernel/`](bridge/kernel/) | [GPL-2.0-only](bridge/kernel/LICENSE) | it modifies Linux kernel source and inherits its licence |
| firmware findings | none | facts are not copyrightable |
| vendor firmware | not mine | not included here, and not mine to license |

The kernel patch is the one that is not negotiable. It edits
`drivers/usb/gadget/function/f_fs.c`, so it is a derivative work of GPL-2.0-only
code, and even the diff carries kernel source in its context lines. It is kept
in its own directory with its own licence file so the boundary is unambiguous.

The userspace bridge is **not** derivative of the kernel. It uses UAPI headers
and syscalls, which is explicitly carved out by the kernel's own `COPYING` note,
which is why ordinary Linux applications are not all GPL. It stays MIT.

Files carry SPDX identifiers so the boundary is machine-readable:

```
SPDX-License-Identifier: GPL-2.0-only    the f_fs patch
SPDX-License-Identifier: MIT             everything else
```

Addresses, offsets, register names and instruction encodings are stated as
facts. Facts are not subject to copyright, so no licence is asserted over them
and none is needed.
