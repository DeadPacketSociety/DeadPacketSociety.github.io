---
layout: post
title: Reverse Engineering Unknown CAN bus Protocols # Title
image: /assets/img/circuit_repair.jpg # Add image post. Remember images are in the /assets/img/ directory (optional, but encouraged)
date: 2021-07-31
description: # Add post description (optional)
author: virusfriendly # virusfriendly or ladynikon
tag: [macphy, CANbus] # Common tags are [macphy, network, host, crypto]
---

# Summary
In this post I will cover how I determine data structures in unknown CAN bus protocols. I’ll then be able to contrast these data structures to identify separate protocols from each other and perform traffic analysis in a future post to determine meaning behind the data structures.

# The Challenge
[HackTheMachine](https://hackthemachine.ai) is a maritime cyber contest hosted by the United States Navy to raise awareness in the computer security industry about challenges and threats facing maritime networks. During the 2020 and 2021 HackTheMachine competitions, teams were given the output from a NMEA2000 logger of a network of devices using a proprietary CAN bus protocol and had to determine various aspects of the CAN bus devices.

# Byte Entropy Analysis
In a [previous post](http://deadpacketsociety.net/Rebuilding-CANbus-traffic-from-NMEA2000-logs/), I demonstrated how I extracted CAN bus traffic from a NMEA2000 log. Now I can create a script to correlate traffic by CAN bus ID, and count the variety of values in each byte of data for us to analyze.

CAN bus frames can have up to 8 bytes of data. Each byte should have a minimum of 1 value seen, and a maximum of 256. I have the script output these values in padded hex for easy viewing.

Using the following value counts as an example:
`91 08 02 01 9b 08 75 71`

Given enough samples, we can use [Benford’s Law (the first digit law)](https://youtu.be/XXjlR2OK1kM) to determine the size of a data field. Leading digits have less variety or entropy than the trailing digits in a value, with leading zeros having no entropy.

The first digit had 145 (0x91) different values and the values to the right of it have decreasing entropy until we reach the 4th byte which only had one value. This strongly indicates that the value is little endian, and 32bit. Using this technique we can see that the next field is likely a 16 bit value.

The last two bytes are more troubling, as there’s less than a 5% difference between them. These could be two 8 bit values, one 16 bit value, or a variety of other data formats. The technique I’m using here works well for identifying byte aligned integers but not bit fields, floating point values, or non-byte aligned values. It’s alright to assume byte aligned values for now, as I will analyze these values in a future post.

As the last two bytes each have a count greater than 16 (0x10), I continue to assume that they are an integer value, and assume that they are a 16bit value. If the counts for these bytes were less than 16, then I would suspect that they are bit flags. Additionally, I could have used Benford’s Law on a bit level to determine if these values had a sign bit that was always off, this would improve detecting smaller field sizes.

## Similarities with SAE J1939
Proprietary protocols are a black box, which is often assumed that the vendor created their own protocol, but nothing prevents the vendor from taking inspiration from open standards. I know I’m making a big assumption here, but this protocol seems very similar to SAE J1939.

SAE J1939 data have multiple fields called signals. These signals are also little-endian, except for ascii data. Both of these protocol features match the proprietary one. However, I don't think the protocol is entirely SAE J1939, as we don't see the typical address reservation messages from our analysis of SAE J1939 in the previous post.

## Analyzing Constants
In the example, the 4th byte only has one value which could be zero, but it could be a single byte non-zero constant. Additionally, SAE J1939 uses special values to indicate meaning. A signal containing all 0xFF bytes means that the device doesn’t support that signal. While a signal containing all 0xFE bytes indicates an error.

I need a way to indicate to slip into my byte counts that it’s a constant and the type of constant without making a mess of the output. To avoid any confusion I can’t use any symbol that could be a hexadecimal digit, so I’m left with G-Z.

* ZZ indicating Zero byte
* XX indicating Unsupported Signal
* XE indicating Error
* YY indicates any other constant values.

I can also use this method to indicate a byte that has had all 256 values without messing with the output format.

* GG indicating 256 values

# Results
The following is an output of my script, along with my manual demarcation of signals. Notice that five pairs of CAN bus IDs have the same signal arrangement (or SPN in SAE J1939 jargon). This is expected as the CAN bus devices in question have redundant controls.

![Protocol field output](/assets/img/htm-can-protocols.png)

# Conclusion
Had I made the assumption that these CAN bus devices were using a single unified protocol, it would have been impossible to conclude any meaning from the data. By examining the data per device, I was able to make some educated guesses on each device’s signal formats. However, I still don’t know the meaning behind these signals, or what each device is yet, but knowing these signal formats will allow me to perform traffic analysis in a future post.

Source logs and code associated to this post can be found at [https://github.com/VirusFriendly/HackTheMachine-Notes](https://github.com/VirusFriendly/HackTheMachine-Notes)

Special thanks to Fathom5, Booz Allen Hamilton, and the United States Navy for providing the opportunity to gain hands-on experience with maritime networks, and to Ploppowaffles for review and feedback on this post.

Additionally, I would like to thank Calc who gave me my first view of a car’s CAN bus traffic as viewed through an entropy analysis tool he developed. While this work is not directly derived from his, I would often think back to that demonstration while working on this competition.

