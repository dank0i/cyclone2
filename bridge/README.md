# Cyclone 2 to Xbox 360 Wireless Receiver bridge

A Raspberry Pi presents a Bluetooth game controller to Windows as an Xbox 360
Wireless Receiver. The result is native XInput with working rumble and battery,
with nothing installed on the PC: no ViGEmBus, no driver, no user-space agent.

Built for a GameSir Cyclone 2 in its Bluetooth HID mode, but the Xbox side is
generic. Any pad you can read from `hidraw` can be mapped into it.

```
Cyclone 2  --Bluetooth HID-->  Pi  --USB gadget-->  Windows (xusb22 / XInput)
```

Measured: 400 Hz end to end, USB high speed, 125 us endpoint interval.

## Status

| feature | state |
|---|---|
| XInput input | working |
| rumble | working |
| battery level | working (4 levels, which is all XInput has) |
| survives reboot | yes, via systemd and a persisted Bluetooth bond |
| Steam battery display | working, via XInput NiMH type |

---

## The finding that makes this work

Emulating an Xbox 360 receiver needs two things at once:

1. the USB configuration must carry Microsoft's **vendor-specific descriptor**,
   or `xusb22` will not create a controller
2. the device must **receive OUT packets**, or there is no rumble and no LED

No stock Linux USB gadget framework does both.

| framework | carries the vendor descriptor | delivers OUT |
|---|---|---|
| gadgetfs | yes | **no** |
| functionfs | **no**, rejects it with `EINVAL` | yes |
| raw-gadget | yes | **no**, on this hardware |
| configfs `f_hid` | n/a, HID only | yes |

raw-gadget is the one to watch out for. It accepts input indefinitely,
`xusb22` binds, the
controller appears in Windows, `XInputGetState` returns success with a climbing
packet counter, and Steam enumerates it. Meanwhile `EP_READ` on the OUT endpoint
blocks forever and never returns a byte, with no errors, while the endpoint is
enabled and its address is correct.

That combination sends you looking at Windows, at `xusb22`, at the descriptor,
or at the Xbox security handshake. The fault is none of those, it is the gadget
framework underneath.

### How to tell quickly

Windows sends the **LED/player-slot command unprompted** when a 360 controller
connects, before any game runs. If you see no OUT traffic at all, the pipe was
never usable. A policy or focus-related filter would still let the LED command
through.

```
00 00 08 40    <- Set LED, sent by Windows at connect
```

---

## The kernel patch

functionfs delivers OUT correctly and refuses the vendor descriptor. That
refusal is one `default` case.

`ffs_do_single_desc()` in `drivers/usb/gadget/function/f_fs.c` rejects any
descriptor type it does not model, and one rejection kills the **entire**
descriptor set. Xbox type `0x21` falls into the `USB_TYPE_CLASS | 0x01` case and
dies there because our interface class is `0xFF` rather than HID, CCID or DFU.
Type `0x22` reaches `default`.

The patch passes both through, scoped to vendor-class interfaces so no standard
class parsing changes:

```c
	if (*current_class == 0xFF &&
	    (_ds->bDescriptorType == 0x21 || _ds->bDescriptorType == 0x22)) {
		pr_vdebug("vendor passthrough descriptor: %d\n",
			  _ds->bDescriptorType);
		return length;
	}

	/* Parse descriptor depending on type. */
	switch (_ds->bDescriptorType) {
```

Build out of tree against the running kernel. `f_fs.c` needs only `u_fs.h`,
`u_os_desc.h` and `configfs.h` from the tree; `u_f.h` no longer exists and its
contents are in `include/linux/usb/func_utils.h`, which ships in the headers
package.

```sh
make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo cp usb_f_fs.ko /lib/modules/$(uname -r)/updates/
sudo depmod -a
```

Installing to `updates/` makes it take precedence over the stock module at boot.

**A kernel update silently reverts this** and rumble stops. Rebuild and
`depmod -a` after upgrades.

---

## Descriptors

The vendor descriptor form must match the interface protocol. They are not
interchangeable: the `0x21` form uses 1-byte report sizes and `0x22` uses 2-byte
ones.

| `bInterfaceProtocol` | descriptor | length |
|---|---|---|
| `0x01` wired pad | `0x21` | 17 bytes |
| `0x81` adapter | `0x22` | 20 bytes |

The adapter descriptor, verified byte for byte against a real 045E:0291 dump:

```
14 22 00 01 13 81 1D 00 17 01 02 08 13 01 0C 00 0C 01 02 08
```

Decoded, it is entirely about endpoints and report sizes:

| offset | field | value |
|---|---|---|
| 0 | bLength | `0x14` (20) |
| 1 | bDescriptorType | `0x22` adapter |
| 4 | wReports | `0x8113` = EP `0x81` IN, 3 reports |
| 6, 8, 10 | report sizes | 29, 23, 2 bytes |
| 12 | wReports | `0x0113` = EP `0x01` OUT, 3 reports |
| 14, 16, 18 | report sizes | **12**, 12, 2 bytes |

Interface and endpoints:

```
interface   09 04 00 00 02 FF 5D 81 00       class FF, subclass 5D, protocol 81
EP IN       07 05 81 03 20 00 01             32 bytes
EP OUT      07 05 01 03 20 00 08             32 bytes
```

Note the endpoint packet size is 32 while the largest OUT report is 12. Those
are independent, and the spec says so explicitly. Sizing the endpoint to the
report is a mistake.

### Hardware limits

- The real receiver is a **full-speed** device with `bMaxPacketSize0 = 8`.
  A high-speed gadget must use 64; copying the 8 makes the host refuse
  enumeration outright.
- `bInterval` means different things at the two speeds. Copying a full-speed
  device's values onto a high-speed gadget gives you the wrong period.
- A faithful 4-slot receiver is impossible on `dwc2`: it has 8 endpoints and a
  real receiver needs 16.

---

## Protocol

All observed live. Headers are 5 bytes; most packets append data.

### Inbound (device to host)

| packet | bytes |
|---|---|
| presence | `08 80`, exactly 2 bytes |
| announce | `00 0f 00 f0 ...`, 29 bytes |
| input | `00 01 00 f0 <20-byte report>`, 29 bytes |
| battery / state | `00 00 00 <b0> <b1>` |

SDL creates the joystick only when a 2-byte `08 8x` arrives **after** it opens
the device, so presence must be heartbeated. A single announce at configure time
is missed, and the symptom is that XInput works while Steam sees nothing.

### Outbound (host to device)

| packet | meaning |
|---|---|
| `08 00 0f c0 ...` | controller status request |
| `00 00 00 40 ...` | **battery status request** |
| `00 00 02 80 ...` | capabilities request |
| `00 00 08 4x` | set LED, `0x40 \| led_type` |
| `00 01 0f c0 00 <strong> <weak>` | rumble |

---

## Battery

Windows **asks** for battery. It is not enough to push it on a timer; answer
`00 00 00 40` when it arrives.

The reply is a State packet whose byte 1 is bit-packed:

```c
// byte 0:  vibrationLevel:2, headset:1, chatpad:1, always_0x1:4
// byte 1:  unknown:1, batteryType:2, onlyMic:1, powerState:2, batteryLevel:2
```

So:

```c
p[3] = 0x10;                            /* high nibble must be 0x1 */
p[4] = (level << 6) | (0 << 1);         /* level bits 6-7, type bits 1-2 */
```

The type value matters more than it looks, and the reason is only visible in
the driver. See [Battery type, and why Steam ignored it](#battery-type-and-why-steam-ignored-it).

### A mistake that is easy to make here

Sending `0xA0 | level` looks reasonable and is wrong. `0xA0` is `0b10100000`,
so bits 6-7 are already `0b10` = 2 = MEDIUM, and the level you compute lands in
low bits that are ignored. The battery is pinned to MEDIUM permanently.

It is a hard one to spot, because a value is plainly being sent, and forcing
every level from 0 to 3 changes nothing. That reads as "Windows ignores battery"
rather than "I am sending a constant".

### Battery type, and why Steam ignored it

For a while this bridge reported a correct battery *level* to Windows while
Steam showed no battery at all. The two facts are connected, and the link is a
mapping inside `xusb22.sys` that no public implementation documents.

Steam reads XInput battery through SDL, and SDL gates on the **type** before it
looks at the level:

```c
case BATTERY_TYPE_UNKNOWN:      state = SDL_POWERSTATE_UNKNOWN;
...
if (state == ON_BATTERY || CHARGING) percent = ...;  else percent = -1;
```

`UNKNOWN` means Steam discards the battery entirely, whatever level you send.
And Windows was reporting `type = 255`.

Decompiling `xusb22.sys` (10.0.26100.8972) shows why. The State packet decoder
takes type from byte 4 bits 1-2, and then a second function maps that wire
value before handing it to the API:

| wire value (byte 4 bits 1-2) | XInput `BatteryType` |
|---|---|
| 0 | 3, NIMH |
| 1 | 2, ALKALINE |
| 2 | 0xFF, UNKNOWN |
| 3 | 0xFF, UNKNOWN |

Two of the four values are invalid, and the "obvious" NiMH constant from the
public header (`3`) is one of them. Send **0** for NiMH. With that, Windows
reports `type=3, level=n`, SDL treats it as a real battery, and Steam shows a
percentage and fires its low-battery notice.

Two other things worth knowing from the same investigation:

- SDL's hidapi driver for the 360 receiver, which parses `00 00 00 13 <level>`
  on a 0-255 scale, is **not** the path Windows uses. `xusb22` owns the USB
  interface, so Steam on Windows goes through XInput and gets the 4-level
  scale. That hidapi path is Linux and macOS.
- Sending that `0x13` packet *as well* makes things worse on Windows: `xusb22`
  parses it as a second State packet and it clobbers the level from the first.
  Verified by suppressing it, which restored the level.

---

## Bluetooth: making the pairing survive a reboot

Two separate problems, and they will both bite anyone bridging a BR/EDR HID
device.

### 1. HID rejected as `!bonded`

```
profiles/input/device.c:hidp_add_connection()
    Rejected connection from !bonded device
```

The pad opens its HID control channel before BlueZ has committed the bond, so
the profile is refused and the pairing then unwinds. `ClassicBondedOnly`
defaults to true in modern BlueZ.

```ini
# /etc/bluetooth/input.conf
[General]
ClassicBondedOnly=false
```

### 2. The link key is generated and then thrown away

Symptom: pairing succeeds, `Paired: yes`, HID attaches, everything works, and
after a reboot the device is gone and the pad will not reconnect.

The cause is visible only in the HCI trace:

```
> HCI Event: IO Capability Request
      Authentication: No Bonding - MITM not required (0x00)     <- us
> HCI Event: IO Capability Response
      Authentication: No Bonding - MITM not required (0x00)     <- the pad
> HCI Event: Simple Pairing Complete
> HCI Event: Link Key Notification                              <- key exists
  ...
  new_link_key ... store_hint 0                                 <- discard it
```

A key **is** generated every time. The kernel marks it non-persistent because
the pairing negotiated No Bonding, and BlueZ honours that. Per spec, such a key
is session-only.

The fix is not about who initiates. `hci_persistent_key()` also returns true on
**your own** requested auth type, independent of what the peer asks for:

```c
/* Local side had dedicated bonding as requirement */
if (conn->auth_type == 0x02 || conn->auth_type == 0x03) return true;
```

BlueZ derives that auth type partly from the registered agent's IO capability.
What I measured: with no agent registered the trace shows `No Bonding (0x00)`,
and with a `KeyboardDisplay` agent it shows `Dedicated Bonding - MITM required
(0x03)`, which satisfies the check above and persists.

I have not traced exactly why the no-agent case ends up at `0x00` rather than
one of the dedicated-bonding values, so treat the agent as an empirical
requirement here rather than a fully explained one.

### Working recipe

```sh
# 1. become unconnectable, so the pad cannot page us first.
#    It always wins that race, which forces the pairing to arrive incoming,
#    which is what produces No Bonding.
sudo hciconfig hci0 noscan

# 2. clear any stale record
bluetoothctl remove "$MAC"
sudo rm -rf /var/lib/bluetooth/*/"$MAC"

# 3. one bluetoothctl session, with an agent that has input capability
{
  echo "agent KeyboardDisplay"; sleep 1
  echo "default-agent";         sleep 1
  echo "scan on";               sleep 60      # put the pad in pairing mode now
  echo "pair $MAC";             sleep 20
  echo "trust $MAC";            sleep 3
  echo "quit"
} | bluetoothctl

# 4. restore and connect
sudo hciconfig hci0 piscan
bluetoothctl connect "$MAC"
```

The resulting trace:

```
Authentication: Dedicated Bonding - MITM required (0x03)     <- us
Authentication: No Bonding - MITM not required (0x00)        <- the pad
Link Key Notification
```

The pad still refuses to bond, and the key persists anyway. Verify with a
`[LinkKey]` section in `/var/lib/bluetooth/<adapter>/<device>/info`, then reboot
and confirm the pad reconnects on its own.

### Dead ends

- `btmgmt pair` fails with `Already Connected (0x09)`, because the ACL is
  already up by the time it runs
- plain `bluetoothctl pair` with no agent registered pairs successfully and
  still does not persist
- pairing with page scan left enabled always loses the initiator race

---

## Debugging notes

Errors that mean something other than what they say.

**`org.bluez.Error.Failed br-connection-create-socket`** is not a local socket
problem. The HCI trace showed `Connect Complete -> Status: Page Timeout (0x04)`.
The peer simply did not answer.

**`Connected: yes` with no `hidraw` node** is not a connection problem. The ACL
is up and the HID profile was refused. Check `journalctl -u bluetooth`, because
`bluetoothctl` will not tell you.

**`EP_ENABLE` handles are not `EPS_INFO` indices.** They index the *enabled*
endpoint list. If you enable IN then OUT, you get handles 0 and 1, which may
coincidentally match the `EPS_INFO` positions of `ep1in` and `ep1out` and make a
bogus verification look like proof.

**`RUN` failing with `EBUSY` when the UDC is free.** In my own program this was
argument parsing: a `--wired` flag was being read as the positional UDC driver
name, so `INIT` bound to a driver that did not exist. Worth checking before you
conclude anything about the mode you were trying to test, because every result
from that path is void.

**`dwc2_hsotg_ep_stop_xfr: timeout DOEPCTL.EPDisable`** in dmesg means the
controller was wedged by tearing down functionfs while the host was still
attached. It binds afterwards but never enumerates. A reboot clears it.

---

## Service

The UDC must be bound **after** the program has written its descriptors.
functionfs materialises the endpoint files only once it has accepted the set, so
`ep1` appearing is the real readiness signal:

```ini
ExecStartPre=/path/setup-ffs.sh          # builds the gadget, does NOT bind
ExecStart=/path/x360w_ffs
ExecStartPost=/bin/sh -c 'for i in $(seq 1 40); do [ -e /dev/ffs-x360w/ep1 ] && break; sleep 0.25; done; echo <udc> > /sys/kernel/config/usb_gadget/x360w/UDC'
```

Only one gadget can own the UDC, so any other gadget service must be disabled
rather than merely stopped.

---

## Latency

```
pad -> Pi     400 Hz, bounded by the pad
Pi -> Windows USB high speed, 125 us endpoint interval
```

Send on arrival rather than on a timer. An early version polled a 2 ms
`usleep` loop while the endpoint was ready every 125 us, adding up to 2 ms of
pure queuing delay. A condition variable signalled by the reader thread removes
it. This does not change the report rate, which is pad-bound; it changes how
long each report waits.

---

## Credits

This builds directly on other people's work.

| project | author | used for |
|---|---|---|
| [x360-research](https://github.com/InvoxiPlayGames/x360-research) | InvoxiPlayGames | Xbox 360 USB protocol, real receiver descriptor dumps |
| [MS-XUSBI](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-xusbi/) | Microsoft | the XUSB specification, report tables and descriptor forms |
| [Linux kernel](https://www.kernel.org/) `f_fs` | kernel contributors | the functionfs gadget this patches |
| [BlueZ](http://www.bluez.org/) | BlueZ contributors | the Bluetooth stack, and `hci_persistent_key` semantics |
| [ViGEmBus](https://github.com/nefarius/ViGEmBus) | nefarius | reference XUSB descriptors |
| [tinyusb-xinput](https://github.com/fluffymadness/tinyusb-xinput) | fluffymadness | reference XInput descriptors |
| [ArduinoXInput](https://github.com/dmadison/ArduinoXInput) | dmadison | OUT report parsing reference |
| [GP2040-CE](https://github.com/OpenStickCommunity/GP2040-CE) | OpenStickCommunity | working XInput rumble precedent |
| [OGX-Mini](https://github.com/wiredopposite/OGX-Mini) | wiredopposite | working XInput rumble precedent |

The last five matter for a specific reason: their working wired implementations
are what established that XSM3 authentication is **not** required on Windows,
and that 32-byte endpoints are the norm. Without those precedents, the silent
OUT failure here would have been very hard to distinguish from an authentication
requirement.

The kernel patch is a modification of `drivers/usb/gadget/function/f_fs.c` and
is derivative of GPL-2.0 kernel source. It is offered under the same terms.
