---

layout: post
author: Al Calzone
title: "Doubling the Z-Wave range by changing a single bit"
summary: |-
  Or: <i>Why an open source SDK would be a good idea</i>
excerpt: |-
  Or: <i>Why an open source SDK would be a good idea</i>
# image: chameleon-coding.jpeg
# feature: chameleon-coding.jpeg
comments: true

---

What if I told you that you could double the range of your Z-Wave device with a single line of code like this one?

```c
bool doubleRange = true;
```

This is exactly what I did! The story is a bit more complicated, so let's start from the beginning.

## A bug in Simplicity SDK

_Shocking, right?!_

As mentioned before, I'm working on firmware for the Nabu Casa Z-Wave controller. This firmware is based on Simplicity SDK, which is the preferred one for 800 series Z-Wave devices. Mainly because it comes with several much needed stability fixes, and support for EU Long Range.

While doing so, I stumbled across a powerlevel bug, where even on chips that support +20 dBm transmit power, attempting to change the power level through the Serial API to a value above +14 dBm results in an error instead. This is especially weird, since Simplicity SDK 2024.12.1 explicitly sets the default powerlevel to +20 dBm on those chips. But once you try to change it, you're suddenly limited to +14 dBm, **halving the achievable range**.

At the time of writing, Silicon Labs have promised a release containing the fix within a couple of weeks. But given this was first reported 8 months ago, and SDK versions available for certification often lag a bit behind, I wanted to get the ball rolling by fixing this in existing firmware.

So let's dig into it...

## Source code only gets us so far

The `Serial API Setup` command is responsible for configuring the RF region and powerlevel among other things. In the NCP controller sample application, which most controller firmwares are based on, this command is handled in `cmds_management.c`. To set the powerlevel above +14 dBm, the 16-bit version of the `SetTxPower` command is used, which first retrieves the max. supported TX power, to ensure that the new value is within the supported range.

```c
iTxPowerMaxSupported = GetMaxSupportedTxPower();
```

This function sends a command to the Z-Wave stack running in the background, and waits for the response:
```c
// Wait for protocol to handle command
SZwaveCommandStatusPackage result = { 0 };
if (GetCommandResponse(&result, EZWAVECOMMANDSTATUS_ZW_GET_TX_POWER_MAX_SUPPORTED))
{
  return result.Content.GetTxPowerMaximumSupported.tx_power_max_supported;
}
```

Sadly, searching for `EZWAVECOMMANDSTATUS_ZW_GET_TX_POWER_MAX_SUPPORTED` in the source code yields no other results. Once again, this happens because Silicon Labs helpfully pre-compiles parts of the Z-Wave stack into binary blobs and ships those instead of providing the source code.

## Whipping out the red dragon

We've been here [before]({% link _posts/2025-02-13-adventures-with-zwave-bootloaders.md %}). Like last time, we managed to get around it by using [Ghidra](https://github.com/NationalSecurityAgency/ghidra?tab=readme-ov-file#ghidra-software-reverse-engineering-framework). Open the `.out` file for our firmware, wait a minute, and suddenly we have semi-readable C code for everything, including the closed source "magic".

The exchange of commands with the background task is done using FreeRTOS's queues, so to find where the command is handled we can search for usages of `xQueueReceive`. We soon end up in a function called `ProtocolInterfaceAppCommandHandler`. The handler for this specific command calls `zpal_radio_get_maximum_tx_power` to determine the powerlevel limit, which essentially does this:
```c
if (m_TxPowerMode == ZW_RADIO_TX_POWER_MODE_20DBM /* 1 */) {
  return 200;
}
else { /* 0 */
  return 140;
}
```

_Pretty sophisticated, huh?_

Another code search yields no other results for `m_TxPowerMode`, aside from its declaration in the `.bss` section. Essentially this means that `m_TxPowerMode` will always be 0 at runtime, which means the max. TX power will always be +14 dBm, no matter the chip. Given that this used to work on Gecko SDK (Simplicity SDK's predecessor), I can only speculate that someone forgot to initialize this variable when splitting out the new SDK.

## Fine, I'll initialize it myself!

My first thought was that I could modify the value of the variable in the linker script without touching the binaries or source code. However, there seems to be no way to initialize variables if they are not initialized (or zeroed) in the C code, causing them to get placed in the `.bss` section.

And because the SDK contains no source code for these parts, there are also no (public) header files that could be used to reference the existing declaration from code - or simply to initialize the variable with the correct value in the first place.

However, the toolchain is nice enough to spit out a `.map` file which contains the memory addresses of all symbols in the binary, including `m_TxPowerMode`.
In our case, this happens to be address `0x20009503`:

```
.bss.m_TxPowerMode
    0x20009503  0x1 C:\Users\...\libzpal_EFR32ZG23.a(ZW_RadioGecko.c.obj)
```

With that knowledge, at every startup, we can execute a slightly more "sophisticated" version of
```c
m_TxPowerMode = ZW_RADIO_TX_POWER_MODE_20DBM;
```

by using a pointer, and two magic values:

```c
int main(void)
{
  // Fix incorrect max. TX Power on Simplicity SDK
  // Equivalent to m_TxPowerMode = ZW_RADIO_TX_POWER_MODE_20DBM;
  uint8_t* m_TxPowerMode = (uint8_t*)0x20009503;
  *m_TxPowerMode = 1;
  // ...
}
```

And suddenly, everything works again. We just needed to flip a single bit (not clickbait)!

## Closing remarks

A true open source SDK would have made this a lot easier. This article is pretty condensed, but during the investigation I took a bunch of wrong turns that cost me a couple of hours. I can only imagine other engineers have gone down a similar path.

Simply having access to the source code would have allowed to find and fix or work around the root cause in a matter of minutes.
Add another couple of minutes for a bug report to Silicon Labs and they could have fixed it for everyone in the next release.
Instead, we had to wait 8 months and pending!

At least now there's some competition on the horizon, so maybe this will change in the future. One can only hope!