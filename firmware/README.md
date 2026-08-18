# Cyclone 2 firmware: reverse engineering notes

GameSir Cyclone 2. JieLi BR23 SoC (AC635N/AC695N family), pi32v2 core, 1 MB
flash. The 2.4 GHz dongle is a second BR23 with its own 512 KB flash, its own
key, and its own firmware, covered below.

These notes cover getting the firmware off the device, recovering the
encryption key, producing a usable decompile, and six firmware patches that
are currently running on hardware. They also document four rendering defects in
the public pi32v2 Ghidra processor module, one of which silently corrupts a
class of global addresses.

No firmware images or decompiler output are included. Apply the offsets and
values here to a dump of your own device.

## Contents

- [Methodology](#methodology)
- [Getting the firmware off the device](#getting-the-firmware-off-the-device)
- [Recovering the encryption key](#recovering-the-encryption-key)
- [Getting a usable decompile](#getting-a-usable-decompile)
- [Image layout: two segments, not one](#image-layout-two-segments-not-one)
- [Mode map and report format](#mode-map-and-report-format)
- [The dongle firmware](#the-dongle-firmware)
- [The 2.4G link is Bluetooth](#the-24g-link-is-bluetooth)
- [Address mapping](#address-mapping)
- [A SLEIGH bug: byte-load addresses are wrong](#a-sleigh-bug-byte-load-addresses-are-wrong)
- [Instruction encodings](#instruction-encodings)
- [Timer subsystem](#timer-subsystem)
- [The rate investigation](#the-rate-investigation)
- [Where the ceiling actually is](#where-the-ceiling-actually-is)
- [The patches](#the-patches)
- [Flashing](#flashing)
- [Things that did not work](#things-that-did-not-work)
- [Open challenge: the 0xe53f instruction](#open-challenge-the-0xe53f-instruction)

---

## Methodology

The offsets below only matter if you own this controller. The process
generalises, so it goes first. Every item is something that actually cost time
here.

### Verify the tool before trusting its output

The pi32v2 SLEIGH module renders byte-load addresses incorrectly (see
[below](#a-sleigh-bug-byte-load-addresses-are-wrong)). Every global read off a
byte load was wrong by a nibble position, which is why xref hunting for globals
kept failing.

Decompiler output is a hypothesis. When a finding matters, confirm it against
the raw encoding, and where possible against the running device.

### Make sure your harness can produce a positive result

The most expensive mistake in this project was a long run of careful negative
experiments performed on an apparatus incapable of returning a positive.

On the bridge side, every USB descriptor variation was tested against a gadget
framework that never delivered an output packet under *any* configuration.
Endpoint sizes, intervals, descriptor types, interface counts: correctly
tested, correctly negative, entirely meaningless. What broke the deadlock was
not another variation but a control experiment: *has this path ever carried a
single byte?*

So before trusting a negative result, check that the setup is capable of
producing a positive one.

### Beware metrics that look principled

Two examples from this project, both of which sent work in the wrong direction:

**Entropy does not identify the right decryption key here.** On a region with a
known-correct root, an entropy sweep ranked the true key at **7.8439 against a
median of 7.8536**, i.e. below median, and put garbage keys on top. The payload
is compressed, so a correct decryption still reads about 7.8. Zero-count, the
number of `0x00` bytes after decryption, ranks the true root first.

**Crib hit count does not identify it either.** A wrong key produced 1975
consistent hits where the correct one produced fewer. Structural markers, such
as recognisable strings and all-zero blocks, are the discriminator.

Validate any scoring metric against a region whose answer you already know
before trusting it on a region you do not.

### Measure under load, not at idle

2.4G idles at 3.75 ms between reports and runs at 2.50 ms under stick motion.
Any latency figure from a resting controller measures the idle path. Amber
happens to be flat, but you only learn that by measuring both.

### Print sample counts, because a dying link fakes a win

Raising a keepalive interval produced a visibly better gap histogram. It was
killing the link: fewer samples, and the survivors were the good ones. Any
metric over a variable-size sample needs its sample size printed beside it.

### Distrust a summary statistic that disagrees with the raw data

My own measurement script reported 304 Hz for a run whose mean gap was 4.36 ms.
Both cannot be true, since `1000 / 4.36` is 229. It was splitting a capture into
two timed phases, but anything arriving after the split still landed in the
second bucket while the rate was computed against the shorter phase length, so
roughly 24 seconds of gaps got divided by 18 seconds. That inflates by exactly
the ratio you would expect.

The mean and the percentiles were right the whole time. When a rate disagrees
with `1000 / mean`, the mean is the one to believe.

### Compute the physical ceiling before optimizing

Effort went into raising the 2.4G report rate before anyone established what
the transport could carry. Once report sizes were measured on the wire, the
answer fell out of air time: 2.4G was already at 85% of its physical ceiling.
See [Where the ceiling actually is](#where-the-ceiling-actually-is).

### Find the computation, not the symbol

Loads through a base register plus offset produce no Ghidra xrefs, so searching
for references to a global finds nothing even when dozens of sites use it.
Battery was solved by locating the ADC averaging function and reading outward,
after xref hunting had failed repeatedly.

### Locate patch sites by unique byte pattern, not address arithmetic

An off-by-2 in the address mapping survived a long time because it usually
landed on the right byte anyway. Patch builders here search for a pattern and
assert uniqueness:

```
pattern 03 ff 04 20 5b 06   ->  exactly 1 occurrence  ->  safe to patch
```

If the pattern is not unique the build fails rather than guessing.

### Verify the encoding, not just the byte

One patch generator matched the wrong byte of a 2-byte instruction. Twenty
edits "verified" clean because in those instructions the register field
happened to hold the value being searched for. Applied, it would have changed
destination registers throughout.

Decode what you are about to change, and confirm the operand at that offset is
the operand you think it is.

### Confirm you are patching the site that serves your case

The first attempt at raising the 2.4G report rate changed an in-flight gate and
did nothing. The reason was not that the gate was the wrong *kind* of
constraint. There are **six** in-flight gates in this firmware; exactly one
serves 2.4G and the other five serve the Bluetooth modes. The patch had landed
on a Bluetooth gate, past the point where the 2.4G branch ends, so it was never
testing what it appeared to test.

A null result from the wrong site looks exactly like a null result from the
right one, so it is worth confirming the site serves the case you are testing
before drawing conclusions from it.

### Gate anything destructive

1. dump the live device first and record the hash
2. build the patch from *that dump*, never from a pristine image
3. validate the live image's own CRCs with your own implementation, which
   proves your checksum code matches the bootloader before you write anything
4. read back the target sectors and compare against the dump
5. write
6. read back and compare against the intended result
7. keep revert sectors cut from the same dump

---

## Getting the firmware off the device

The pad exposes a vendor HID report that drops it into the BR23 boot loader.

```
send HID report   0x0F 0x17 0x55 0x88   to the pad's hidraw node
  -> re-enumerates as 4c4a:2342 "BR23UBOOT1.00"
  -> /dev/sg0 appears, SCSI-transported loader protocol
```

From there [jl-uboot-tool](https://github.com/kagaimiq/jl-misctools) reads and
writes flash.

### Why this is hard to brick

The mask ROM's serial download path is permanent and cannot be overwritten, and
none of the patches here touch `uboot.boot`. A bad app image means reflashing
1 MB rather than a dead device, provided you kept the stock dump.

### What was dumped

| image | size | notes |
|---|---|---|
| controller flash | 1 MB | flash id `0xeb6014` |
| dongle flash | 512 KB | flash id `0x856013`, identity `GS_C2_Dongle` at `0x10` / `0x1010` |
| mask ROM | 10 KB | via the loader |
| RAM in loader mode | | not what it looks like, see below |

### The RAM dump is the loader, not the app

`ram0` / `ram1` captured in loader mode contain the **UBOOT loader**, which
speaks a different protocol entirely. They are not a snapshot of the running
application, so anything you conclude from them about the running firmware
will be wrong.

Relatedly, `0x01E00120` reads as zeros in loader mode. Link addresses only
exist while the app is running, so this is expected and is not evidence that
anything is missing.

### The dongle is a different target

The dongle is a second chip with its own firmware and its own flash. It does
**not** accept the controller's `0f 17 55 88` loader-entry report, and it is
only the command target while the pad is asleep, which is easy to get backwards
if you assume it behaves like the pad.

---

## Recovering the encryption key

The application image is encrypted with JieLi's "ENC" cipher: an LFSR
(CRC16-CCITT-like) whose key is re-derived every 32 bytes from the block
address.

```
block_key = chipkey ^ ((offset - BASE) >> 2)      # re-derived every 32 bytes
for each byte:  out = in ^ (k & 0xff)
                k   = ((k << 1) ^ (0x1021 if k & 0x8000 else 0)) & 0xffff
```

| chip | chipkey | BASE |
|---|---|---|
| controller | `0xFFFF` | `0x4020` |
| dongle | `0xFFFF` | `0x11000` |

Quarkslab published a teardown of a *different* JieLi chip, the AC6958, where
the equivalent constant was `0x170f`. The BR23 parameters were not published
anywhere and were recovered here.

### Why you will see `0xeff7` quoted instead

`0xeff7` for the controller and `0xbbff` for the dongle turn up as "root keys",
sometimes with the conclusion that the key is not family-wide. Both are the same
scheme written the wrong way round:

```
0x4020 >> 2 = 0x1008          0xFFFF ^ 0x1008 = 0xEFF7
0x11000 >> 2 = 0x4400         0xFFFF ^ 0x4400 = 0xBBFF
```

Both chips use the same chipkey. What differs is the base offset. Using
`0xeff7` against absolute offsets matches the real scheme only where the
subtraction does not borrow, so you get roughly a third of the image decrypted
cleanly and the rest as noise that reads convincingly like compressed data. The
dongle behaves the same way with `0xbbff`.

Use the base-relative form and it decrypts cleanly throughout.

### The crib-drag

Index all 65536 possible keystreams by a 16-byte window, then look for
ciphertext whose XOR against a *known* plaintext is a valid keystream.
Zero-padding regions are the productive crib.

**The trap that invalidates the whole method:** the LFSR is linear, so the XOR
of two keystreams is itself a valid keystream. Cribbing ciphertext against
ciphertext therefore matches unconditionally and yields contradictory keys.
Only crib against data you can prove is plaintext.

**Hit count is not the discriminator.** A wrong candidate produced 1975
consistent hits, more than the correct key did. Confirm structurally instead:

- 74 blocks decrypt to all-zeros
- recognisable strings appear: `app_area_head`, `cfg_tool.bin`, `PRCT`, `CODE`
- FreeRTOS symbols: `xQueueGenericSend`, `vTaskStartScheduler`, `prvDeleteTCB`
- chip identifiers: `AC695X_*`, `JL-BR22`, `br22xx`
- full debug log strings

### Use the published tools

None of this needs reimplementing. Kagaimiq's tools handle it:

```sh
python jl-misctools/firmware/fwunpack_newfw.py --dirname out cyclone2_fw.bin
```

The account above is here to explain the parameters and the traps, not to
replace the tool.

---

## Getting a usable decompile

| tool | source |
|---|---|
| Ghidra | [ghidra-sre.org](https://ghidra-sre.org/) |
| pi32v2 processor module | [quarkslab/ghidra-jieli](https://github.com/quarkslab/ghidra-jieli), derived from [kagaimiq/ghidra-jieli](https://github.com/kagaimiq/ghidra-jieli) |
| architecture reference | [kagaimiq/jielie](https://github.com/kagaimiq/jielie) |
| vendor SDK, cross-reference | [Jieli-Tech/fw-AC63_BT_SDK](https://github.com/Jieli-Tech/fw-AC63_BT_SDK) |

pi32v2 is derived from Analog Devices Blackfin. Ghidra 12 uses PyGhidra rather
than Jython, which matters if you are adapting older scripts.

The import must create **two** memory blocks or most of the image will not
resolve. See [the next section](#image-layout-two-segments-not-one).

### Coverage reached

```
45%  ->  83% coverage, 137811 instructions, 1720 functions
segment 2 accounts for 103 of those functions, about 200 KB
```

### Ghidra coverage numbers are cumulative

Ghidra projects persist code units between script runs, so coverage figures
accumulate silently across attempts. This nearly produced a false "the
controller is 76% code" conclusion. Re-import into a fresh program before
quoting any coverage number, or explicitly clear code units first.

### Run length beats coverage percentage

For deciding whether a region is code, the better test is how many instructions
decode consecutively from an arbitrary seed. Real code sustains long runs that
end on control flow. Non-code dies within one to three instructions. Coverage
percentage tells you how much the disassembler *attempted*, not how much of it
was right.

---

## Image layout: two segments, not one

`app.bin` is **two link segments**:

```
seg1_flash   app.bin 0x00000..0x66E34   linked at 0x01E00120
seg2_sram    app.bin 0x66E34..0x721AD   linked at 0x02000000   (0xB379 bytes)
```

The obvious thing to do is import one flat block at `0x01E00120`, and that is
wrong. Every call into segment 2 then renders as an unresolved external, which
makes it look as though a large part of the firmware lives in mask ROM you
cannot dump. It does not; it is sitting in the file you already have.

### Verified against hardware

`app.bin 0x66E38` onward is byte-for-byte identical to a live `memread` of SRAM
`0x000004` in loader mode. The address space aliases, so `0x00000000`,
`0x02000000` and `0x04000000` return the same bytes.

### The alias trap this creates

```
0x01e6864c - 0x01E00120 = 0x6852C   =  segment 2's offset for that function
```

so `FUN_01e6864c` in single-block decompiles is the same thing as
`0x020016F8`, the 2 ms joypad callback, under two different names.

### Addresses not worth chasing

`0x01c7xxxx` / `0x01c8xxxx` are all `malloc`, `free` and `memcpy`. `memread`
returns zeros there in loader mode, and nothing in the timer path leaves
`app.bin`.

---

## Mode map and report format

The mode selector is `DAT_0000a0b0`, and almost every interesting branch in the
firmware tests it:

| value | mode |
|---|---|
| 1 | 2.4G dongle |
| 2 | Bluetooth XInput |
| 3 | Bluetooth DInput, the amber mode |
| 4 | Bluetooth Switch Pro |
| 5 | Bluetooth PS4 |
| 6 | Bluetooth Steam |

The pad also uses a **different Bluetooth address per mode**, so a host paired
in one mode does not see it in another. On this unit the amber address ends
`...F8`, Switch Pro `...FA`, and the 2.4G pairing mode `...F5`.

That has a practical consequence worth knowing: the pad can hold a bond with one
host in amber and a different host in another mode at the same time, and you
choose between them with the mode combo rather than by re-pairing.

The amber report is report `0x07`, 11 bytes, and entirely standard HID:

```
07 | 80 80 80 80 | 0f | 00 00 | 00 00 | 00
id |  X  Y  Z  Rz | hat| 16 buttons | L2 R2 | pad
```

Four axes at 0-255 centred on `0x80`, a 4-bit hat, 16 buttons, and two analog
triggers. Byte 10 is the padding byte that the battery patch later reclaims.

The IMU is an LSM6DS3. Audio is 2.4G only by construction: SBC encode and decode
initialisation is gated on `DAT_0000a0b0 == 1`, so the headphone jack cannot
work over Bluetooth in any mode, patched or not.

---

## The dongle firmware

The 2.4G dongle is a second BR23 with its own firmware, and it is worth dumping
even if you only care about the pad. It is the reference implementation of the
protocol the pad speaks: the 25-byte output block layout came from
reverse-engineering the dongle's own report sender (`FUN_01e0c26a`, which uses
that layout for report IDs `0xb1` through `0xe2`), not from guessing at captures.

| | controller | dongle |
|---|---|---|
| flash id | `0xeb6014`, 1 MB | `0x856013`, 512 KB |
| identity string | `GS_C2_ADC_DEVICE` | `GS_C2_Dongle` |
| link base | `0x01E00120` | `0x01E000C0` |
| cipher base | `0x4020` | `0x11000` |
| encrypted payload | | `0x010000` to `0x06052f` |

### Getting it off

The dongle does not accept the pad's `0f 17 55 88` loader-entry report. It is
only the command target **while the pad is asleep**, showing PID `0575`. While
the pad is linked at `0x100b`, the same command reaches the pad instead.

That is a race, not a state you can check and then act on: the pad can wake
between your check and your send. Poll and fire from one script rather than
two commands.

### The firmware is interleaved

This is the part that stops a naive dump from disassembling. The image is
stored as alternating 4 KB pages: code pages interleaved with high-entropy
protection pages. De-interleaving them into two streams separates the code from
the noise, and disassembly coverage improves substantially once you do.

That structure matches the `CODE` and `PRCT` entries in the file table, which is
what makes the interpretation more than a guess.

If you dump this chip and it looks like high-entropy noise that decrypts
correctly but will not disassemble, de-interleave before concluding anything
about the cipher. The apparent "packing" here is largely a page-layout problem
rather than compression.

I am stating this one qualitatively on purpose. My notes carry specific entropy
and coverage figures for the two streams, but they disagree with each other
across two passes and I no longer have the split files to re-measure, so the
shape of the finding is solid while the exact numbers are not.

### JLFS entry layout

Both chips use the same file table, and the entries are checkable rather than
guessable:

```
0x00  hdr_crc  (2)
0x02  data_crc (2)
0x04  offset   (4)
0x08  size     (4)
0x0C  attr     (1)      bit 0x40 = compressed, only ever set on uboot.boot
0x0D  reserved (1)
0x0E  index    (2)
0x10  name     (16)
```

`hdr_crc` is CRC16-CCITT with init `0x0000` over bytes `0x02` to `0x20`, so you
can validate an entry instead of assuming your parse is right. That check is
what caught an earlier wrong reading of the field layout.

---

## The 2.4G link is Bluetooth

The dongle is sold as a "2.4GHz wireless receiver" and it is a second BR23
running ordinary Bluetooth BR/EDR. There is no proprietary radio protocol. That
means a Raspberry Pi with a normal Bluetooth adapter can take its place, which
is how the LED, rumble and audio paths were explored without owning a second
dongle.

Three things make it awkward, and all three have to be handled together.

### It bypasses SDP

The dongle does not do service discovery. It connects straight to the L2CAP HID
PSMs, `0x11` for control and `0x13` for interrupt, which is exactly what its own
firmware does. Its L2CAP stack is hand-built rather than a standard one.

This is why BlueZ cannot be used in the normal way. BlueZ does SDP first, the
pad offers nothing in this mode, and BlueZ gives up before opening any channel.
You have to open the PSMs yourself over raw L2CAP.

### The pad only accepts addresses it already knows

The pad keeps a table of bonded dongle addresses in flash, around `0x0f712a`,
each entry a 4-byte prefix followed by a BD_ADDR:

```
0x0f712a:  20 25 60 00 | D0:56:81:xx:xx:xx
0x0f7134:  4c 25 60 00 | D0:56:81:xx:xx:xx      <- the dongle's own address
```

A connection from anything else is refused. So the adapter has to present one of
those addresses. Read the table off your own pad; the values are specific to
each unit and its dongle, which is why they are not reproduced here. Then set
the adapter's address with the vendor HCI command, bytes in reverse order:

```sh
sudo hcitool cmd 0x3f 0x0001 <b6> <b5> <b4> <b3> <b2> <b1>
sudo hciconfig hci0 reset
```

### Output rides in a 25-byte block

Motors, LED and audio volume all travel in one 25-byte block inside report
`0x11`, with the motors at block offsets 17 and 18.

The block is checksummed, and the checksum is plain `zlib.crc32` over bytes 0
through 22, low byte in 23 and high in 24. That is to say, **with** the final
XOR:

```python
crc = zlib.crc32(block[0:23]) & 0xFFFF
block[23] = crc & 0xFF
block[24] = (crc >> 8) & 0xFF
```

Getting that wrong is quiet and expensive. Nothing downstream runs until a block
passes CRC, so an inverted checksum gates the LED *and* the report rate at once,
and the pad simply behaves as though it is idle rather than reporting an error.
Fixing it took the link from 10 Hz to 166 Hz and lit the LED in the same change.

`block[14]` selects which report personality the pad sends. `0x00` gives the
full report including LED, and is the one worth using. A real dongle runs at
roughly 200 Hz.

Wrong bytes elsewhere in the block are not ignored either: the pad drops the
link rather than discarding the packet, and then powers off after a couple of
minutes with no host.

---

## Address mapping

```
file offset  = address - 0x01E00120
flash offset = app.bin offset + 0x4140
immediates   = instruction address + 2
```

Raw flash is encrypted, so instruction patterns are findable only in the
decrypted image, never in a flash dump. Patch builders decrypt, locate, edit,
and re-encrypt only the changed bytes. Recover the keystream as
`orig_decrypted XOR orig_cipher` so any live dump can be handled.

`0x01E0011E` is a tempting value for the base and it is wrong by two. It lands
on the intended byte often enough that a patch built against it can work, which
is what makes it worth stating.

---

## A SLEIGH bug: byte-load addresses are wrong

This invalidates a class of `DAT_0000eXXX` / `uRam0000eXXX` symbols.

For the 4-byte byte-access form through the global base register (`50ee` load,
`52ee` store), the **load** form inserts a spurious `0` nibble into the
displacement, so a real `0x3a` prints as `0x30a`. The store form renders its
displacement correctly (it has a different problem, see
[below](#three-more-rendering-defects)).

### Conversion rule

Read a byte-load symbol `0xe810 + 0xH0L` as `0xe810 + 0xHL`.

### Verified alias table

```
eb1a -> e84a    eb1b -> e84b    eb1c -> e84c    eb1d -> e84d
ef1d -> e88d    e91b -> e82b    e91c -> e82c    e91d -> e82d
ec14 -> e854    ec15 -> e855
```

### Demonstrated by construction

The clearest evidence is synthesised instructions assembled by hand and fed
through the module. Identical bit fields, load versus store:

```
50ee 08 01     lb.z r0,[r0 + 0x108]      load,  displacement should be 0x18
50ee 08 02     lb.z r0,[r0 + 0x208]      load,  should be 0x28
50ee 00 0f     lb.z r0,[r0 + 0xf00]      load,  should be 0xf0
52ee 5c 01     sb   r0,[r5 + 0x1c]       store, correct
52ee 17 25     sb   r2,[r1 + 0x57]       store, correct
```

The store decodes displacement as `(b3 & 0xf) << 4 | (b2 & 0xf)`, which is
right. The load reads the same bits as a **12-bit** field where the store reads
**8-bit**, so the high nibble lands one position too far left.

The firmware demonstrates it internally as well. `FUN_01e27e62` writes the
advertising data length to `+0x17` with an `sb`, then reads that same field back
to log it, and the read renders as `+0x107`.

It also shows up in real code. In `FUN_01e36e0c` one counter is written by a
short-form store:

```
01e36e52:  db45        sb r3,[r5+0x5]    with r5 = r4+0x38 = 0xe848   -> 0xE84D
```

and read back through the 4-byte load form:

```
01e36e36:  50ee4d23    renders as [r4 + 0x30d]  -> "0xeb1d"
```

The same variable, rendered two different ways depending on how it was
accessed.

Practical impact: `DAT_0000e91b`, the in-flight send counter that appears
throughout the joypad task, is really **`0xE82B`**.

Halfword (`50ed` / `51ed`) and word forms are unaffected.

This is worth reporting upstream. It is a silent correctness bug, not a display
quirk.

### Three more rendering defects

Found alongside the address aliasing. All three affect how much you can trust a
decompile of this architecture:

| form | defect |
|---|---|
| 4-byte store (`52ee`) | source register is masked to 3 bits, so `r8`-`r15` render as `r0`-`r7`. Roughly 197 sites carry the form |
| `sb eregC,[++eregB = eregA]` | the stored register and the addend are transposed |
| indexed halfword loads with `imm1619=8` | the `<<1` shift is omitted, under-computing the address |

The store defect is the nastiest of the three, because a wrong source register
reads as plausible code rather than an obviously broken address.

### Instruction pairing, and the leading underscore

This one is not a defect, it is the core being dual-issue, and misreading it
will cost you a patch.

An instruction printed with a **leading underscore** is the second half of a
pair. It commits the value produced by the *previous* instruction, not the value
printed on its own line:

```
01e44ee2:  0027   lw  r0,[sp+0x1c]      loads L2
01e44ee4:  80d4   clr r0_r1
01e44ee6:  b842   _sb r0,[r3+0x2]       stores what the lw loaded, not the cleared 0
01e44ee8:  b843   sb  r0,[r3+0x3]       stores the cleared 0
```

Read naively, both stores appear to write the same register and the first looks
redundant. They write different values.

It is worth being precise about this, because it is tempting to call it delayed
writeback and reason about it as a pipeline hazard. It is neither. The processor
module handles it and marks it, and the `.slaspec` says so directly: for a
prefix of 6 the instruction is parallelised and the next one executes first.

The practical consequence for patching: **pairing changes ordering but does not
create slots.** If you are trying to fit extra work into a run of instructions,
the pairing buys you no room.

---

## Instruction encodings

**`mov rX,#imm8`**

```
0x2000 | ((imm & 0x1F) << 8) | 0x40 | ((imm >> 5) << 3) | reg
```

**`je rX,#imm,target`** (4-byte form): byte2 is the displacement, byte3 is
`imm * 2`.

**`add [rX+off],#imm`** (4-byte form)

```
AA eb II RR      byte0 = 0xC0 + off/4,  byte2 = immediate,  byte3 = reg << 4
```

**Conditional branch word 1**

```
word1 = (imm << 9) | disp9        disp9 = (target - (addr+4)) / 2
```

**`call`** encodes as jaddr23, so targets never appear as literals and a byte
search for them finds nothing. To enumerate call sites, decode every offset
where `b[1] == 0xEA` and `0x80 <= b[0] <= 0xBF`. That is how the timer walker's
single caller was found after Ghidra reported no references at all.

`pi32asm.py` assembles the subset needed for patching. Having an independent
encoder is worth the effort: the processor module's own README notes that
pi32v2 coverage is incomplete.

---

## Timer subsystem

```
TIMER3 (PRD = 24000, 2 ms)
  -> ISR at 0x01E4C5DA
    -> walker FUN_01e00856          the only call site, found via a jaddr23 scan
      -> 2 ms joypad callback at 0x020016F8
         posts the joypad event and advances roughly 200 counters
```

### Durations are hardware-clocked

`usr_timer_add` (real implementation `0x01e006f4`) stores
`deadline = clock() + msec` at entry offset `0x14`. `usr_timer_modify`
(`FUN_01e15a26`) does the same. The walker compares
`puVar6[5] <= (raw_hw_counter + 1) >> 1`, reading a hardware timer pair at
`0x10F0F0` / `0x10F0F4` fresh each pass. It is not a tick accumulator.

**Therefore changing TIMER3 does not skew any timer's duration.** A 20 ms timer
stays 20 ms and gains resolution. The natural objection, that sleep timeout and
battery sampling and LED animation would drift, is refuted.

The counters inside the 2 ms callback are a different matter entirely, and that
is what makes the tick change hard. See
[Things that did not work](#things-that-did-not-work).

### List geometry

15 slots at `0xE4E0`, stride `0x20`, bounds-checked in `usr_timer_del`
(`0xe4df < p && p < 0xe6c0`). Two priority lists headed at `0xF18C` / `0xF194`,
computed as `0xE810 + 0x97C + i*8`, which is why they never appear as xrefs.

| offset | field |
|---|---|
| `+0x08` | callback |
| `+0x0c` | argument |
| `+0x14` | deadline |
| `+0x18` | period (24-bit) |
| `+0x1c` | id |
| `+0x1f` | in-use |

### Resolution floor

A timer entry's `msec` value was changed from 2 to 1 at app.bin `0x38415` and
flashed. It changed nothing measurable. The clock is finer than 2 ms, but the
list is only walked when the ISR fires, so `now+1` and `now+2` are first
satisfied on the same service pass. **2 ms is the effective floor for a timer
request**, and anything below rounds back up to it.

---

## The rate investigation

Amber runs at roughly 480 Hz. 2.4G ran at 166 Hz. Working out why took three
wrong answers first.

### What the 166 Hz figure actually is

One thing to be clear about before the numbers below: **166 Hz was measured
against a 2.4G dongle emulator running on the Pi**, not against GameSir's own
dongle, which does about 200 Hz.

The emulator had been stuck at 10 Hz until an inverted checksum was fixed. The
block checksum is plain `zlib.crc32`, that is, **with** the final XOR:

```python
crc = zlib.crc32(block[0:23]) & 0xFFFF
block[23] = crc & 0xFF
block[24] = (crc >> 8) & 0xFF
```

Setting `block[14] = 0x00` then took the rate from 10 Hz to 166 Hz. One
inverted checksum had been gating both the report rate and the LED, because
neither path runs until a block passes CRC.

### Wrong answer 1: the in-flight gate, patched at the wrong site

The first patch raised an in-flight threshold and did nothing. The reason is
more interesting than "wrong kind of constraint": there are **six** in-flight
gates in this firmware. Exactly one serves 2.4G; the other five serve the
Bluetooth modes. The patch landed on app.bin `0x448B2`, a Bluetooth gate past
where the 2.4G branch ends, so it never tested what it appeared to test.

### The finding: a 4:1 send divider, 2.4G only

Inside `FUN_01e433b8`:

```c
if (DAT_0000a0b0 == '\x01') {                    // 2.4G ONLY
  ...
    if (((DAT_0000e91b < 2) && (((DAT_0000eb70 ^ 1) & 1) == 0))
        && (3 < _DAT_0000e8e2)) {                // needs >= 4
      _DAT_0000e8e2 = 0;                         // divide by 4, then reset
```

Every other send gate in the file has only the in-flight test and no divider.
A search for the divider global returns exactly two hits, both on those lines,
so the gating is unambiguous.

(`DAT_0000e91b` is the in-flight counter, really `0xE82B`, per the
[SLEIGH bug](#a-sleigh-bug-byte-load-addresses-are-wrong).)

### The patch

```
instruction   0x01e44328   03 ff 04 20 5b 06    jb r2,#0x4, 0x01e44fe4
immediate     app.bin 0x4420A     0x04 -> 0x01
flash         0x04834A            sector 0x048000
```

The 6-byte pattern is unique in the image, so the site cannot be misidentified.
Build output from the actual flash:

```
app.bin CRC 0x259B -> 0xFD6E      JLFS entry cksum 0xCC3E -> 0xA172
changed 5 decrypted bytes: 0x4040 0x4041 0x4042 0x4043 0x4834a
```

### Measured

Raw tool output, post-patch:

```
rate 230.1 Hz   mean 4.36 ms   p50 3.75   p95 7.50   p99 8.76   max 28.75
gaps > 10 ms: 27 (0.65%)
```

Against the pre-patch baseline:

|  | pre-patch | post-patch |
|---|---|---|
| rate | 166 Hz | **230 Hz** |
| mean gap | 6.05 ms | **4.36 ms** |
| p50 | 6.25 ms | **3.75 ms** |
| p95 | 8.75 ms | 7.50 ms |
| p99 | 11.2 ms | **8.76 ms** |
| gaps > 10 ms | 2.4% | **0.65%** |

p50 moved off 6.25 ms, where it had sat through every previous attempt.

### What binds it now

Removing a 4:1 divider should have taken 6.25 ms to 1.56 ms. Landing at
3.75 ms, still on the 1.25 ms Bluetooth slot-pair quantum, means something else
binds. It is the **2.4G in-flight gate**: `ja r0,#0x1` on counter `0xE82B`,
which permits a window of 2 outstanding reports. That is the gate the first
patch was aiming at and missed.

---

## Where the ceiling actually is

Air time, from report sizes measured on the wire rather than assumed.

```
2.4G    avg report 109.5 B   (104 B and 120 B dominate)   37042 B/s
amber   report      11.0 B   (single 0x07 format)          5071 B/s
```

2.4G carries **7.3x** amber's payload. At 625 us per Bluetooth slot:

| mode | payload | packet | slots + ack | min period | ceiling | actual | efficiency |
|---|---|---|---|---|---|---|---|
| 2.4G | 109.5 B | DH3 | 3 + 1 | 2.50 ms | 400 Hz | 338 Hz | **85%** |
| amber | 11.0 B | DH1 | 1 + 1 | 1.25 ms | 800 Hz | 461 Hz | 58% |

2.4G is at 85% of what the transport can physically deliver. The remaining gap
is not a firmware problem.

### The amber tick ceiling

There is a tempting argument that a 2 ms task tick cannot be the amber limit,
because 40% of measured gaps come in at 1.25 ms, below the supposed floor.

That does not survive contact with the arithmetic. A producer running at
1.992 ms sampled onto a 1.25 ms grid
necessarily yields 40.6% one-slot gaps, since `T/Q = 1.5936`. So the measurement
is what the tick model predicts rather than evidence against it, and I spent a
while chasing it the wrong way before working that out.

Confirmed three ways: a producer-period fit of 1.992 ms (502 Hz) predicting
40.4% against 40.5% measured; `P(1 -> 1)` of 1.5% where random loss would
require 40%; and flashing the timer request from 2 ms to 1 ms, which changed
nothing.

Measured amber on current firmware: **481.2 Hz**, with gaps falling at
1.25 ms 40.3% of the time, 2.50 ms 54.1%, and 3.75 ms 5.2%. No stalls.

---

## The patches

Six are currently flashed and running:

| patch | what it does |
|---|---|
| poweroff | adds a power-off command the stock firmware does not expose |
| divider | removes the 2.4G 4:1 send divider, 166 -> 230 Hz |
| in-flight | raises an in-flight threshold, but on a Bluetooth gate rather than the 2.4G one, so it changes nothing useful |
| amber LED | gives amber mode a working RGB path |
| amber battery | makes the battery byte visible to the host |
| mode LED | recolours amber's mode LED to match Steam's player colour |

### Power-off

The one I actually set out to build. I wanted the pad to switch itself off when
the PC went to sleep, driven from Home Assistant.

**No native remote power-off exists.** Every sleep primitive in the firmware
(`FUN_01e3a91e`, `FUN_01e36c8c`, `FUN_01e3f5d8`) is gated behind the physical
button, the charge and battery scanner, or the internal idle counter, and the
HID command handler never calls any of them, so there is no command you can
send that would do it.

The patch adds one, reusing the pad's own sleep sequence rather than inventing
anything:

```
handler   0x1e2fa96      command (0x0F, 0x16), an inert config slot
before    movz r2,#0x1d4c ... 46 bytes, ending in a pop at 0x1e2fac2
after     call FUN_01e3f5d8 ; pop {pc,r7,r6,r5,r4}
```

`FUN_01e3f5d8` is the idle-sleep event poster, so the patched command posts the
same event the pad raises on its own idle timeout. Command `0x20` begins at
`0x1e2fac4`, immediately after the handler's `pop`, so overwriting the first 6
bytes leaves the remainder as dead code and touches nothing else.

Ten bytes differ from stock in total: 6 code bytes, plus 4 for `data_crc` and
`hdr_crc` so boot-time integrity checks pass.

### The command table, and where to add your own

The power-off patch works because the HID command handler dispatches through a
jump table with unused entries in it. The table is at `0x01e2f8e4` (file offset
`0x2f7c4`), 32 halfword entries indexed by command byte.

Several slots are inert: `0x11`, `0x13`, `0x15`, and `0x18` through `0x1f`.
Power-off took `0x16`, which leaves roughly ten free for anything else you want
to add. Each is a handler that currently does nothing useful, so overwriting one
costs no existing functionality.

Things that would fit: enter-pairing on demand, a seconds-granularity sleep
nudge to get under the 1 minute register floor, a locate-my-controller that
rumbles and flashes the LEDs, or custom telemetry.

### Sleep timeout, which needs no patch at all

Try this before flashing anything. The pad exposes a **writable inactivity
timeout** over its vendor HID interface, and it gets most of the way to the same
result with no firmware modification.

```
SLEEP_TIMEOUT  register 0x0273   minutes, 0 = never
PICKUP_WAKE    register 0x0272
```

Write 1 minute when the PC sleeps and 20 when it wakes, and the pad powers off
shortly after you stop using it. Two things to know:

- **The minimum is 1 minute**, since the register is in minutes. Ten seconds is
  not reachable.
- **The timeout is evaluated against already-elapsed idle time**, not restarted
  on write. A pad that has been idle for a while powers off almost immediately;
  worst case is a full minute.

The vendor protocol is only live in **XInput mode**. In PS4 or Switch mode the
pad enumerates as Sony or Nintendo and the vendor interface does not exist.

Two more things about that interface, both of which cost time to learn:

**Battery only streams while the pad is being heartbeated.** Send `0x0F 0xF2`
every half second or so and the `0x12` report starts carrying charge state,
battery at byte 36 and the charge flag at byte 35. Stop the heartbeat and the
data stops with it, which looks exactly like the pad not supporting battery over
this interface.

**Register writes go to the LED bank**, `0x20`, and the useful registers are
`0x0273` for sleep timeout, `0x0272` for pickup-wake, and `0x026d` for the
audio-reactive mode. The pad reports as PID `0x100B` when it is linked and
presenting the gamepad.

### Amber LED

Amber is the fastest mode and has no LED path at all in stock firmware. One
instruction opens one, and rumble arrives through the same handler.

```
addr    0x01E3B734      app.bin 0x3B614      flash 0x03F754     sector 0x3F000
before  80 f8 f4 0a     jne r0,#0x5,0x01e3b920    (return unless mode == 5)
after   80 fc f4 04     jbe r0,#0x2,0x01e3b920    (return only if mode <= 2)
```

`FUN_01e3b6e0` is the HID output-report handler, and its structure is an
if/else-if chain rather than a switch: it diverts 2.4G first, then *returns* for
anything that is not DS4, so amber reached nothing. Relaxing that guard lets
amber fall into the DS4 body, which decodes a standard DualShock 4 Bluetooth
output report. No CRC and no 25-byte block, unlike the 2.4G path.

Only the opcode (`f8` to `fc`) and the immediate (5 to 2) change; the
displacement is unchanged. Both encodings exist verbatim elsewhere in the image,
which is a useful check that the assembler produces real forms.

Side effect: modes 4 and 6 also fall through. Harmless, since nothing sends them
a DS4-shaped output report.

Driving it, plain `hidraw` write, no emulator:

```
11 C0 20 F3 00 00 <rumbleR> <rumbleL> <R> <G> <B> ...    (32 bytes)
```

| byte | destination |
|---|---|
| `payload[2]` | flags, bit 0 enables rumble |
| `payload[5]` | `0xE86D` |
| `payload[6]` | `0xE86C` |
| `payload[7..9]` | `0xE8B0` / `0xE8B1` / `0xE8B2` = R, G, B |
| | `0xE81C = 1` latches the override |

Report `0x21` works identically. An all-zero RGB triplet is rejected by a guard,
so pure black is not settable this way.

One address to avoid: `0xEA8B..0xEA8D` looks like the host-settable LED colour
and is not. It is the base colour of the battery-level indicator animation, a
5x3 struct at `0xEA85..0xEA93` feeding the LED output triplets at
`0xF266/F269/F26C/F26F/F272`. Its only writer is the battery-level selector, so
nothing a host sends will ever reach it.

### Mode LED colour

Amber's mode LED is yellow in stock firmware. This changes it, and the technique
is more interesting than the result: rather than finding space for new code, it
**retargets a jump-table entry at an existing branch**.

Mode colours are set through a `tbb` jump table. Most entries are a single
instruction, because most modes set one channel. The default branch is the
exception, since white needs all three:

```
01e4199a  mov r4,#0xff     R
01e4199c  mov r3,#0xff     G
01e4199e  mov r2,#0xff     B
01e419a0  goto tail
```

So point amber at that branch, then recolour it:

| # | app.bin | before | after | effect |
|---|---|---|---|---|
| 1 | `0x41850` | `1f` | `16` | amber's table entry now targets the default branch |
| 2 | `0x4187A` | `7c 3f` | `5c 22` | `mov r4,#0xff` becomes `mov r4,#0x62` (R) |
| 3 | `0x4187C` | `7b 3f` | `43 20` | `mov r3,#0xff` becomes `mov r3,#0x00` (G) |

`0x16` is `(0x01e4199a - 0x01e4196e) / 2`, the table's offset encoding for that
branch. The result is R `0x62`, G `0x00`, B `0xff`, which is `#6200FF`. That is
not an arbitrary purple: it is the player colour Steam assigns this pad, so the
firmware LED and the Steam UI agree.

All three edits are size-neutral, so nothing moves and no other offset in the
function changes.

This is also the place the "verify the encoding" rule earned itself. Two of the
three edits use encodings lifted verbatim from elsewhere in the same function,
but `mov r4,#0x62` did not appear anywhere, so the immediate layout had to be
solved and then checked against 15 known instructions before anything was
written.

### Amber battery

**Byte 10 of the amber report was already carrying the battery value.** It read
`0x64` (100) in every report and was declared as padding. Nothing needed to
compute it or move it.

The fix is the **HID descriptor** alone: declare byte 10 as a Battery Strength
usage with Logical Maximum 100. Linux then creates a `power_supply` entry, which
is what Steam and SDL read natively.

Two things will catch you out. **The descriptor exists twice in the image**, and patching one copy is a silent
no-op, so both need the change.

**BlueZ caches the HID descriptor.** After flashing, the change is invisible
until the cached record is cleared and the device re-paired. This is what made a
working patch look like a failure. Remove the device's stored directory and pair
again.

The accompanying report-builder change was **unnecessary and harmful**. See
[Things that did not work](#things-that-did-not-work).

### Battery, at source level

Found by locating the ADC averaging function; the loads are base-register plus
offset and produce no xrefs.

| global | meaning |
|---|---|
| `0xA0B2` (u8) | battery percent, 0-100 |
| `0xA0B3` (u8) | battery level, 0-4 |
| `0xE91A` (u16) | latest millivolts |
| `0xEC14` (u32) | 15-sample averaged mV |
| `0xE84A` (u8) | Switch/Pro battery + connection nibble |
| `0xEC08` / `0xEC04` | charging / charge-full flags |

Percent formula, from `FUN_01e36d14`:

```c
pct = (mV * 100 - 0x4F588) / 0x352      // 324488, 850
    // 0% at 3245 mV, 100% at 4095 mV, clamped
```

Level thresholds `0x5a/0x4b/0x32/0x19`: >= 90 gives 4, >= 75 gives 3, >= 50
gives 2, >= 25 gives 1, else 0. Charge-full forces 100 / 4.

The ADC path runs in **every** mode: `FUN_01e36e0c` is called unconditionally
from the joypad task's periodic case, before any mode test. Only the transport
differs.

| mode | how battery reaches the host |
|---|---|
| Pro / Switch | report `0x30` byte 2, already exposed by `hid_nintendo` |
| 2.4G | millivolts, big-endian, at capture offsets 68 and 69. No patch needed |
| amber | byte 10, once the descriptor declares it |
| DS4 / XInput / Steam | not carried |

Input report offset 72 is a trap. It reads 70 decimal, which looks like a
battery percentage, but it is a hardcoded `0x46` and never moves no matter how
long you charge the pad. I lost time to that one.

---

## Flashing

Start by confirming you are talking to the device you think you are. The chip
carries a plaintext product-id string at flash `0x1000 + 0x10`, up to
16 bytes, readable in the loader:

```
GS_C2_ADC_DEVICE     the controller
GS_C2_Dongle         the 2.4G dongle
```

This is worth using as a gate because it comes off the flash itself rather than
from a USB descriptor, so it cannot be spoofed by whatever the device is
currently pretending to be, and it does not depend on firmware version. It reads
the same on pad firmware 3.26 and 3.52, and on dongle firmware 1.19 and 1.21.
The two chips take completely different images, so writing one to the other is
the obvious way to lose a device.

```
1. dump the live device, record the hash
2. build from that dump, not from a pristine image
3. validate the live image's CRCs with your own jl_crc16
4. read back target sectors, compare against the dump
5. write
6. read back, compare against the intended image
7. keep revert sectors cut from the same dump
```

A real build log, for shape:

```
live vs pristine: 1688 decrypted bytes differ, 7 inside the app region
located divider @flash 0x048348 (app.bin 0x44208) : 03 ff 04 20 5b 06
patched 0x04 -> 0x01 at flash 0x04834a
app.bin CRC 0x259B -> 0xFD6E      JLFS entry cksum 0xCC3E -> 0xA172
```

Rollback is a sector write of the saved originals:

```sh
jluboottool.py --device /dev/sg0 "write 0x004000 revert_sector_004000.bin"
jluboottool.py --device /dev/sg0 "write 0x048000 revert_sector_048000.bin"
```

### JLFS checksums

The entry checksum is `jl_crc16` over `entry+2 .. entry+0x20`, and application
length comes from `entry+8`, not `entry+0x0C`. See
[JLFS entry layout](#jlfs-entry-layout) for the full structure. Getting either
wrong produces an image that fails its own checksum, which the gate in step 3
catches before anything reaches the device.

---

## Things that did not work

Each of these was a plausible theory that tested clean and was still wrong.

### The 1 ms tick (flashed, then reverted)

TIMER3 `PRD 24000 -> 12000` at app.bin `0x38436` (`c0 5d` -> `e0 2e`) is one
halfword. The problem is everything else in the same callback.

The report producer and the counter bank are the same function, so doubling its
rate doubles its counters:

| class | sites | required action |
|---|---|---|
| millisecond (`+2`) | 114 | must become `+1` |
| millisecond (`+4`) | 1 | must become `+2` |
| raw tick (`+1`) | 83 | rate doubles, every threshold needs individual review |

The full patch came to 116 edits, 115 of them landing in a single sector. It was
built, byte-verified, flashed with all gates passing, and reverted. The firmware
afterwards was byte-identical to before the attempt.

For a **1.333 ms** tick there is no integer increment that works at all, so
every millisecond-based timer in the pad would run 50% fast permanently with no
way to compensate.

### The timer request change (flashed, no effect)

`msec 2 -> 1` at app.bin `0x38415`. Measured identical afterwards, because the
timer list is only walked when the ISR fires and requests below 2 ms round back
up to it.

### The first rate patch (flashed, wrong site)

Landed on a Bluetooth in-flight gate rather than the 2.4G one, so it produced a
null result that looked exactly like the gate not mattering.

### The amber battery report-builder patch (froze the pad)

Replacing `clr r0_r1` with a load of the battery value into `r0` froze amber
completely: link up, no input at all.

`clr r0_r1` clears a register **pair**. The replacement loaded `r0` and left
`r1` uninitialised, breaking the report builder. Reverted, full recovery, amber
back at 480 Hz.

It was also unnecessary, since byte 10 already carried the value and only the
descriptor needed changing.

### Entropy sweeps for key recovery

Ranked the known-correct root below median. Covered under
[Methodology](#beware-metrics-that-look-principled).

---

## Open challenge: the 0xe53f instruction

The pi32v2 processor module does not decode the 4-byte instruction `0xe53f`
(word `0xe53f`, bytes `3f e5`, class prefix `111`).

```
525 sites total
340 within 0x600 of a function entry  ->  code, not data
```

Upstream confirms it is genuinely unknown: the module's own TODO lists
"remaining missing instructions" for pi32v2.

Decoding it and writing a SLEIGH rule would complete coverage of the image and
would be a contribution to the processor module itself.

Two things not to expect from it. The undecoded remainder is not all blocked by
`0xe53f`; a good part of it is genuine data, string tables and constants. And it
does not hold the idle counter, which came out of the decoded portion anyway
(`0xe8d8`, threshold `DAT_00014efd`, sleep flag `0xe843`, event poster
`FUN_01e3f5d8`).

So it is completeness and an upstream fix rather than a hidden secret, but it is
a well-scoped problem if that sort of thing appeals to you.
