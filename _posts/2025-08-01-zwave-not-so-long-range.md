---
layout: post
author: Al Calzone
title: "Z-Wave (not so) Long Range?!"
summary: |-
  Or: <i>Getting to the bottom of some surprising test results</i>
excerpt: |-
  Or: <i>Getting to the bottom of some surprising test results</i>
# image: chameleon-coding.jpeg
# feature: chameleon-coding.jpeg
comments: true
---

[Z-Wave Long Range](https://www.silabs.com/wireless/z-wave/z-wave-long-range-overview) (LR) is touted to be the next generation of Z-Wave, promising impressive ranges of over a mile in open air. We did some range tests before while working on our [upcoming Z-Wave adapter](https://www.youtube.com/watch?v=Z-pUkM6XIuA), and [so did others](https://drzwave.blog/2024/12/05/geographic-location-command-class-gps-coordinates-to-the-centimeter/), both with great results. In comparison, the advertised range for classic Z-Wave is around 100 meters in open air.

Of course, open air tests are somewhat artificial. Real devices are inside homes or commercial buildings, with other devices and sources of interference, which all have an impact on the achievable range. Z-Wave Long Range is also supposed to be more robust against that due to its different broad-spectrum modulation scheme.

So imagine my surprise when I recently participated in range tests in a smart hotel that had hundreds of existing Z-Wave devices, and found that **Z-Wave Classic beat Long Range by a significant margin** - using the same device!

Cue Leonardo DiCaprio:

> _You had my curiosity, but now you have my attention!_

## What sets Z-Wave Long Range apart?

The name is great for marketing, but it is a bit of a misnomer. Most of the "Long Range" actually comes from sheer transmit power. At first, LR was only available in the US, where the transmit power of classic Z-Wave devices was limited to -1 dBm. In comparison, Z-Wave LR could theoretically transmit at +30 dBm, but no devices for that kind of power exist yet. Current devices are limited to +20 dBm, which is still a significant increase over the classic Z-Wave transmit power - and range!
In addition, Z-Wave LR uses a different modulation scheme ([DSSS OQPSK](<https://en.wikipedia.org/wiki/Phase-shift_keying#Offset_QPSK_(OQPSK)>)), which is more robust against interference and is said to be equivalent to another 2 dB increase in transmit power.

In Europe, where Z-Wave LR is brand new, Z-Wave Classic was already allowed to transmit at +13 dBm - the same as Z-Wave LR. So, LR is not expected to go much further, except what the difference in modulation can achieve. In previous tests, we did reach far over a kilometer though, so there is still significant potential in the technology.

Another main difference though is that Z-Wave Classic always transmits at the same power, while Z-Wave LR includes an algorithm for automatically adjusting the transmit power. This is meant to avoid unnecessary power consumption and interference, while saving battery life.

Und da liegt der Hund begraben - as we Germans say. (_There's the rub!_)

## Dynamic transmit power

The specifications for this are vague - likely on purpose:

> The algorithm shall ensure that the minimum TX power is used and at the same time maintain a link budget between the source and destination that ensures a robust and error free communication.

The protocol deals with this by having the receiver of a message return its received signal strength and the measured noise floor to the sender. The sender can then adjust its transmit power accordingly, so that the link budget at the receiver is not unnecessarily high, but also not too low to cause communication errors:

![Signal Strength](/images/signal_strength.svg){: style="display: block; margin: 0 auto; max-height: 400px" }

This is essentially like talking in a public place. The higher the background noise around you, the worse you can hear the person you are talking to, and the louder they have to speak. If you are further away, the other person also has to speak louder, because the sound is attenuated by the distance between them and you.
When you can't hear the other person well, you ask them to speak louder (or say _"WHAT?"_). When they are louder than necessary, you tell them to lower their voice. If you stop answering, they repeat their last sentence a bit louder.

Sounds reasonable, right?

## Did you just talk back to me?!

Consider the example of two people talking over the phone. Person A is in a noisy bar and has trouble understanding anything the other person says, and person B is in a quiet room at home.
![Different link budget](/images/signal_strength_2.svg){: style="display: block; margin: 0 auto; max-height: 400px" }

For the argument's sake, let's assume that A's phone has a microphone with good noise suppression. It is then pretty intuitive in this situation that A can speak normally, while B has to speak louder for A to understand them. This is obviously true for both the conversation, and the feedback the listener gives the speaker about their volume.

Looking at the Zniffer traces from the range tests, I noticed something odd:
The acknowledgements for a frame from device A to device B were always returned with the same TX power as the original message - even if B had previously determined that it needs a completely different TX power. This can happen if both devices are subjected to different background noise levels or if they have antennas with vastly different RF performance.

What does this mean for our example? Let's assume that device A needs +14 dBm to reach device B, but it would pick up frames from device B just fine if B sends them at 0 dBm:

- B sends frame to A at 0 dBm.
- A acks the frame at 0 dBm. ACK does not reach B, because it would need +14 dBm to punch through the noise.
- B increases TX power to +3 dBm.
- A acks the frame at +3 dBm. ACK does not reach B, because it would need +14 dBm to punch through the noise.
- B increases TX power to +6 dBm.
- A acks the frame at +6 dBm. ACK does not reach B, because it would need +14 dBm to punch through the noise.
- ...

In the end, both A and B are now transmitting at +14 dBm, even though 0 dBm would be enough for B → A. Likewise, A sends frames to B at +14 dBm and B acknowledges them at +14 dBm, even though 0 dBm would be enough for A → B.

This is a waste of power and actually **fails to ensure** what the specification we quoted above requires: that the minimum TX power is used.

But this isn't even the weirdest part. If both devices just used unnecessarily high TX power, that would not explain why Z-Wave Classic performed better.

## I can't hear you!

Let's recall that the receiver of a frame sends back the signal strength of the incoming frame, and the noise floor it measured. In other words, it returns to the sender its current link budget, but as two distinct values. Most likely to leave some flexibility in the implementation. The sender can then calculate the link budget by subtracting those two values.

In the "open" source implementation maintained at the Z-Wave Alliance, the algorithm aims for a link budget of 6...10 dBm. If the link budget is lower, the TX power gets increased by 3 dB. If it is higher, the TX power is decreased by 3 dB. I can only assume that the Silabs implementation (that is widely used in millions of devices, including the ones we tested) aims for the same link budget, but it allows other increments than 3 dBm. Let's look at some Zniffer traces:

| Frame no. | TX power | Noise at sender | Noise at receiver | Signal strength | Link budget |
| --------- | -------- | --------------- | ----------------- | --------------- | ----------- |
| 1         | 20 dBm   | -89 dBm         | -81 dBm           | -73 dBm         | 8 dB        |
| 2         | 17 dBm   | -94 dBm         | -81 dBm           | -74 dBm         | 7 dB        |
| 3         | 14 dBm   | -95 dBm         | -77 dBm           | -73 dBm         | 4 dB        |
| 4         | 10 dBm   | ...             | ...               | ...             | ...         |

We start out at full blast (which is actually too high for Europe, but that's another story). This yields a link budget of 8 dB, which is within the target range.

The next frame is sent at **17 dBm**, 3 dBm less than the previous one.\
_Okay, maybe the Silabs implementation is a bit more aggressive in reducing the TX power..._\
This results in a link budget of 7 dB. Surely this is okay?

The third frame uses **14 dBm**, again 3 dBm less than the previous one.\
_Uhh, cutting it a bit close, I guess..._\
The link budget of this exchange is only 4 dBm, which is below the target range. So the TX power gets increased to...

...**10 dBm**?!

This continued a bit, all the way down to **6 dBm** TX power, with a link budget of around 2 dBm -- so barely nothing.

![WAT?](/images/side_eyeing_chloe.jpg)

Attentive readers might have noticed something about that table. We haven't talked about the noise at the sender at all. To be honest, I don't know what it is needed for, except if we wanted to give the receiver a hint about the transmit power it needs for its ACKs (which it always sends at the same power as the sender anyways). So why is the value even in that table?

| Frame no. | TX power | Noise at sender | Signal strength at receiver | Difference |
| --------- | -------- | --------------- | --------------------------- | ---------- |
| 1         | 20 dBm   | -89 dBm         | -73 dBm                     | 16 dB      |
| 2         | 17 dBm   | -94 dBm         | -74 dBm                     | 20 dB      |
| 3         | 14 dBm   | -95 dBm         | -73 dBm                     | 22 dB      |
| 4         | 10 dBm   | ...             | ...                         | ...        |

Ohhh...

...

![...the math ain't mathing](/images/math_aint_mathing.webp){: style="max-height: 300px" }

This does explain the behavior, but it just makes no sense.

> TL;DR: The dynamic power algorithm of Z-Wave LR is bugged, and possibly has been since its inception. When there is a big difference in background noise between the sender and the receiver, the sender is likely to reduce its TX power, even if the receiver has a less than desired link budget.
>
> This negatively impacts the range of Z-Wave LR, and can even lead to it performing worse than classic Z-Wave in some cases.

## Reproducing

I could have stopped here, but where would be the fun in that? It was time to give one of my experiments a spin:

[https://github.com/AlCalzone/zwave-rcp-firmware](https://github.com/AlCalzone/zwave-rcp-firmware)\
_A very experimental Z-Wave firmware, where everything except transmitting, receiving and configuring the radio is done on a host computer instead of on the Z-Wave chip._

I loaded this onto my development board and checked out a branch of Z-Wave JS, in which I am working on adding support for this firmware. Using this combination, I am able to see all Z-Wave traffic like with a Zniffer, but also **impersonate any device on the network**, including full control over what is being sent (within the limits of what is implemented at this point).

I defined a couple of test cases where I had my fake end device acknowledge frames with fake measurements for the signal strength and noise floor. Then I pinged the device a couple of times from a controller running official firmware and watched how the TX power developed over time:

- The original scenario: Noise floor at the receiver is higher than at the sender, signal strength close to the receiver's noise floor.
- The receiver reports a fixed +2 dBm link budget
- The receiver reports a fixed -2 dBm link budget
- The receiver only ACKs when the sender transmits at full power, then simulates a high noise floor
- The receiver only ACKs when the sender transmits at full power, then simulates a fixed 1 dBm link budget

In most cases, I could reproduce the original issue, although all of these cases should not have resulted in lowering the TX power. There were some edge cases though where the algorithm behaved as intended, which leads me to believe that the Silicon Labs algorithm uses some more complex heuristics than the "open" source implementation. Specifically, it seems that it is far more likely to increase the TX power when the ACKs are transmitted at full power - although I don't see a reason why that should even be taken into account.

## Conclusion

These findings have been reported to Silicon Labs, who were equally surprised by the results. They are currently investigating the issue, and I am hopeful that they will be able to fix it in a future release of the Simplicity SDK.

Nonetheless, the outcome is once again: we have to wait for some SDK release. If this part of the SDK were open source, the code could have been scrutinized by others, and we could be working around it for the short term. Instead, we are once again blessed by the magic of closed source software.

## Final words

How the hell does this not get noticed earlier? Z-Wave Long Range is 4 years old now!