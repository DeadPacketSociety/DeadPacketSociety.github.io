---
layout: post
title: Rebuilding CAN bus traffic from NMEA2000 logs # Title
image: /assets/img/yacht-antenna-array.jpg # Add image post (optional, but encouraged)
date: 2021-06-23 # YYYY-MM-DD
description: Identifying CAN bus devices on a mixed network and rebuilding their CAN identifiers # Add post description (optional)
author: virusfriendly # virusfriendly or ladynikon
tag: [macphy, NMEA2000, J1939, CANbus]
---

# Summary
In this post I will cover how I identified and extracted CAN bus traffic from a NMEA2000 data log of a mixed technology network. This is an important step for untangling and reverse engineering CAN bus data, which I’ll go over in future posts.

# The Challenge
[HackTheMachine](https://hackthemachine.ai) is a maritime cyber contest hosted by the United States Navy to raise awareness in the computer security industry about challenges and threats facing maritime networks. During the 2020 and 2021 HackTheMachine competitions teams were given the output from a NMEA2000 logger of a network that included both NMEA2000 and CAN bus devices, and had to determine various aspects of the CAN bus devices. 

Though NMEA2000 is compatible with CAN bus, these protocols have different addressing schemes and their frames need to be interpreted differently.

# Separating Traffic
To rebuild CAN bus traffic from the NMEA2000 log we will need to understand the details of CAN bus and NMEA2000 frames and how they relate to each other in order to identify anomalous traffic and reconstruct them into CAN bus frames.

## CAN Bus Frame Reconstruction
NMEA2000 is not directly related to CAN bus but instead is based on the SAE J1939 protocol used in commercial vehicles such as trucking and construction. The SAE J1939 protocol is based on the CAN bus protocol used in consumer vehicles most people drive. As they share the same physical standards, all three protocols can operate on the same network. What makes them different is the addressing mechanism of their MAC (media access control) layer. As the addressing mechanism is located in the first 32 bits of the CAN bus frame, we can focus on that and ignore the rest of the frame for this post.

![Visual Summary of CAN bus, J1939, and NMEA2000](/assets/img/canbus-j1939-nmea2000.png)

### CAN bus Identifiers
CAN bus has a single address field called an identifier. This field both identifies the sender, message format, and the message priority with the lower ID having higher priority. The CAN bus ID field can be either 11 bits (CAN bus 2.0a) or 29 bits (CAN bus 2.0b). SAE J1939 and NMEA2000 only use the CAN bus 2.0b frames. While CAN bus 2.0a frames can operate on a network with CAN bus 2.0b frames, it is unlikely that the NMEA2000 data logger would record any CAN bus 2.0a frames.

### SAE J1939 PGN and Addresses
SAE J1939 splits up the CAN bus 2.0b’s 29 bit identifier field into the Message Priority, Parameter Group Number (PGN), Destination Address, and Source Address fields. The PGN number provides context on how to interpret the data field (typically 8 bytes) similar in how ICMP Data is based on the ICMP Type and Code. The PGN is formed by the Data Page, Extended Data Page, PDU Format, and PDU Specific fields.

While some messages need to be sent to a specific device, most do not require a destination address. A PDU Format value less than 240 (0xF0) indicates the message is intended for a specific device. When this occurs the PDU Specific field will contain the ID of that device, and the PGN is calculated with a PDU Specific value of 0 (0x00). PDU format values of 240 (0xF0) or more indicate a broadcast message, and the PGN is calculated with the PDU Specific value of the PDU Specific Field.

![Example PGN Calculations](/assets/img/example-pgn.png)

Note, a message can be sent to a specific identifier of 255 (0xFF) which is effectively a broadcast message, but the PGN would still be calculated with a PDU Specific value of 0 (0x00).

### NMEA2000 PGN and Addresses
NMEA2000 inherits the PGN and address features of SAE J1939 except for the Data Page and Extended Data Page fields. Instead it enlarges the PGN field by extending into the Data Page field, and marks the Extended Data Page field as a reserved field. Besides this slight change in PGN format, what makes NMEA2000 different is mainly due to redefining the PGN values.

We can reconstitute the CAN bus identifier using the NMEA2000 Priority, PGN, Source Address, and optional Destination Address. However, as bit 4 is reserved in the NMEA2000 frame, the log doesn’t provide us its value and so our CAN bus ID will always be incomplete. In general this isn’t a great loss of information, but it is something to keep in mind.

> For those interested, NMEA2000 device address assignment works similar to Automatic Private Internet Protocol Addressing (otherwise known as that annoying 169.154.0.0/16 address when your computer can’t reach the DHCP server). The NMEA2000 device selects an address value and asks the network if that address is already in use. If an address is already using that address, it’ll report “address claimed”. If the device doesn’t receive an “address claimed” message within a window of time, it will begin operating with that address. This form of self address procurement was also inherited from the SAE J1939 specification.

## CAN bus Traffic Identification
Depending on how much log data is collected, you can also determine CAN Bus devices based on traffic analysis. PGN values such as 59392, 59904, 60928, and 126208 are often seen in messages between NMEA2000 devices as they pertain to network address and control function (as explained above). If a device doesn’t participate in this traffic, then it (and its associated PGN value) is likely a misinterpreted CAN bus identifier.

Additionally by determining what NMEA2000 devices exist on the network, through a visual recon, and checking for the PGN values they transmit can help you determine which PGN values you shouldn’t see and are likely misinterpreted CAN bus identifiers. For interoperability, the PGN values any NMEA2000 device transmits or accepts are well documented and can be found either in the product brochure, technical specifications, or user manual. This is true even for proprietary PGNs, the data format will of course be an industry secret, but the PGN will be documented.

# Results
Examining the PGNs in the NMEA2000 log, I was able to identify three values (61184,65280,65281) via network analysis and confirmed by researching known device transmit PGN values. It appeared that there were seven devices based on the NMEA2000 source addresses. However, 15 CAN bus identifiers were revealed when these addresses and PGN values (along with the priority and destination fields) were converted.

![Converted CAN bus identifiers](/assets/img/converted-canbus-ids.png)

# Conclusion
Once the CAN bus frames are separated and reconstructed we are able to correctly organize the CAN bus data by the CAN bus identifier. This will help us while reverse engineering the data formats, which I will go over in a future post.

Source logs and code associated to this post can be found at [https://github.com/VirusFriendly/HackTheMachine-Notes](https://github.com/VirusFriendly/HackTheMachine-Notes)

Special thanks to Fathom5, Booz Allen Hamilton, and the United States Navy for providing the opportunity to gain hands-on experience with maritime networks, and to Ploppowaffles for review and feedback on this post.

