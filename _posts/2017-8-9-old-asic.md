---
layout: post
title: The Secret GAMECART Register — Spelunking in Atari STE Silicon
---

![Inside the STE GLUE/MMU chip](/images/GAMECART.png "Inside the STE GLUE/MMC chip")

<!--break-->

Deep inside the silicon of the Atari STE's GLUE/MMU chip, there is a register
that nobody seems to have documented. Writing a single bit to address 0xFF9000
activates a signal called GAMECART — and nobody knows quite what it was for.

Here's how I found it.

## Background: the dream of a better MMU

As a long-time Atari ST owner I've spent more hours than I should admit with
emulators, clones, and retrocomputing rabbit holes. So when I stumbled across the
news that several of Atari's custom ASIC designs had been
[uncovered and converted to PDF schematics by Christian Zietz](https://www.chzsoft.de/asic-web/),
I was immediately excited. Then I downloaded the files, browsed through them briefly,
and almost immediately forgot about them.

What brought me back was a [discussion on the Atari ST and STe Users Facebook group](https://www.facebook.com/groups/133161394213/permalink/10155613110164214/).
[Chris Swinson](https://exxosnews.blogspot.dk/) had shared an old design from
[Atarihacks](http://atari4ever.free.fr/hardware/zip/16mbram.zip) that attempted to
hack in support for more than the ST's maximum 4MB of RAM, by generating missing
DRAM signals from pins 23 and 22 on the CPU. Nobody who'd tried building it had
gotten it to work, but it seemed tantalizingly close — mostly a timing problem.

The deeper issue, though, is that the CPU isn't the only thing that needs to
address DRAM. The Blitter has the address pins for it, but the DMA and Shifter
generate no address signals at all. The MMU has internal registers for managing
the video buffer and DMA transfers — and since those registers are buried inside
the chip, there's no way to add the missing address bits without replacing the
MMU entirely.

Which brought me back to those schematics.

## Reading the silicon

In a moment of hubris, I decided I'd translate the entire GLUE/MMU design into
Verilog, find a suitable CPLD, and build a functional drop-in clone. Following
[Chris' overclocking work](https://exxosnews.blogspot.dk/2017/07/ste-booster-powered-by-altera-test.html),
such a chip could even be enhanced to support faster clocks and the full 24-bit
address space of the M68000.

The small problem: I had never touched an HDL in my life. I was learning Verilog
at the same time as learning to read 1980s hand-drawn chip schematics. I do like
jumping in at the deep end.

So I started small. The STE's combined GLUE/MMU ASIC was the only version I had
access to. Page 2 of the schematics turned out to be address decoding — relatively
tractable. Tracing back from the pads, I confirmed that the /RAM pin activates for
addresses 0x000008 to 0x3FFFFF. The hand-drawn logic is extremely low level —
chains of AND, OR, XOR gates, buffers, inverters, and curiously long strings of
NOT gates (buffering? deliberate delay?). But it was readable.

## The ROM map gets strange

Next I turned to the /ROMx pads, and immediately noticed something odd: the STE's
GST MCU had *more* /ROMx select signals than the original ST GLUE chip — despite
the STE actually using *fewer* ROM select pins in practice.

The original ST used three pairs of 32KB ROMs for 192KB of TOS. Later models
consolidated these with larger chips and TTL logic. The STE moved the ROM base
from 0xFC0000 to 0xE00000, making room for a 256KB ROM on /ROM2. The /ROM0 and
/ROM1 pins went unconnected — but the schematics revealed they still map to
0xE80000–0xEBFFFF and 0xE40000–0xE7FFFF respectively, meaning the STE could
theoretically address 768KB of ROM if those pins were brought out.

Then there were the new /ROM5 and /ROM6 pins, which I traced to a 512KB region
at 0xD00000–0xD7FFFF. Probing that range was apparently
[already known](http://info-coach.fr/atari/hardware/STE-HW.php) from hardware
experiments, but nobody had explained why those pins were added.

## GAMECART?!

![cartridge or game cart?](/images/ROM3-4.png "cartridge or game cart?")

This is where it got interesting. The decoding for /ROM3 and /ROM4 — the
cartridge port selects used on both ST and STE — had extra logic I didn't expect.
The first part behaved normally, mapping to the traditional 0xFB0000 and 0xFA0000
cartridge address spaces. But there was a second path, controlled by a signal
named GAMECART, that could remap those pins to 0xD80000–0xDFFFFF instead —
combining with the /ROM5 and /ROM6 region to make a full megabyte of
previously mysterious address space suddenly usable.

The name alone raises questions. Did Atari plan a larger game cartridge for the
STE? Were they considering a game console variant of the hardware? I traced the
GAMECART signal back to page 10 of the schematics — the section visible in the
image at the top of this article. It originates from a flip flop, activated by
writing to bit 8 at address 0xFF9000.

As far as I can tell, this has never been publicly documented. If anyone owns an
STE, it would be fascinating to test: write byte 1 to 0xFF9000 and see whether
any connected cartridge relocates to 0xD80000.

## Where this leaves things

I've written Verilog modules for the address decoding and some of the clock
signal generation uncovered above, and simulation confirms the signals activate
correctly. Whether I'll ever translate the entire design is another question —
Chris reminded me that mature FPGA implementations of the ST already exist, most
notably the [Suska project](https://www.experiment-s.de/en/gallery/), which was
built by replacing the original ST motherboard's chips piece by piece.

But this kind of schematic archaeology has value beyond any single project.
Undocumented registers like GAMECART are exactly what emulators miss — and
finding them in the original silicon is the only reliable way to know they exist.

A SUPER-MMU in a CPLD is still on the table. It'll just require better soldering
skills than I currently have, and a willingness to work with chips that come in
packages seemingly designed to be impossible. One step at a time.

## Update

Literally minutes after [sharing this article](https://www.facebook.com/groups/133161394213/permalink/10155663972899214/)
on the [Atari ST and STe users Facebook group](https://www.facebook.com/groups/133161394213/),
Troed Sångberg pointed out [this thread on atari-forum.com](http://atari-forum.com/viewtopic.php?f=15&t=32054)
posted by Christian Zietz himself, discussing the very same GAMECART register and its implications.