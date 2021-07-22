---
layout: post
title: Defining Unknown Ports in Nmap
image: /assets/img/dns-sleuth.jpg # Add image post. Remember images are in the /assets/img/ directory (optional, but encouraged)
date: 2021-07-21
description: Learn how Nmap defines ports and how to update its definitions # Add post description (optional)
author: virusfriendly
tag: [host, Nmap]
---
There are roughly a billion hosts on the internet, with each host having 65,536 TCP and 65,536 UDP ports. These metrics are relatively easy to track and look up. But ask how many application protocols there are, and you will never find the answer. As philosophical sounding the question may be, it reminds those who investigate networks that not only will our tools always be incomplete, but that their amount of incompleteness is unknown.

Nmap is an intensive port scanner that can provide useful information about a system or network we’re investigating. Like all network tools, it is  far from being all-knowing, but peeking under its hood we can use Nmap to discover the missing pieces and improve it by modifying the configuration. 

# What are unknown ports?

![Port Unknown](/assets/img/port-unknown.png)

When performing a basic Nmap scan of a host without any bells or whistles, we will come across at least one unknown port about 10% of the time (a guesstimate)

Nmap uses a file named nmap-services as a lookup table for port definitions. The file location depends on your system’s OS and configuration. On Ubuntu, it can most likely be found in `/usr/share/nmap`. The format and data is similar to /etc/services found on Unix-based systems, which is used by other Unix utilities for port and service name resolution. A key difference between these files however, is that the nmap-services file contains the percentages that the port has been seen among Internet wide scans. It is based on this data I made the prior guesstimate.

![nmap-services](/assets/img/nmap-services.png)

An unknown port is a port that is either not included in the file or defined as unknown in this services file. Of the possible 65,536 TCP or UDP ports, Nmap 7.60 has service names for 6385 TCP ports and 5637 UDP ports. The rest remain unknown because they are used by transient protocols, closed protocols, or custom assignments.

## Transient Protocols

A number of services are able to operate without a statically defined port number, and instead depend on another service manager or discovery method to communicate the end point.

In the last century these were mostly used by a class of protocols known as Remote Procedure Calls or RPCs. These protocols depend on a manager known as a port mapper. With Unix targets, if Nmap is able to access the port mapper, then Nmap will interrogate it for service names of all the RPC endpoints. Windows RPC endpoints can also be interrogated in a similar fashion, but requires the inclusion of an Nmap script.

> nmap <target> --script msrpc-enum

In recent decades a new class of protocols have sprung up known as ad-hoc protocols that utilize service discovery protocols such as Web Services Dynamic Discovery, Universal Plug And Play, and Multicast DNS. These protocols rely on the host advertising their services across the local network, rather than depending on a request method used by RPCs.

The nature of these ad-hoc protocols make it difficult for Nmap to determine the services. Once more Nmap scripts provide a helpful workaround.

> sudo nmap --script broadcast-wsdd-discover,broadcast-upnp-info

> nmap <target> -sV --script broadcast-upnp-info

## Closed Protocols

Officially, service names are registered to port numbers by the [Internet Assigned Numbers Authority](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt) (IANA). This works well for protocols which conform to open standards to be interoperable between different implementations, known as open protocols, but most protocols are only implemented and maintained by a single provider. These protocols, known as closed or proprietary protocols, forgo any official assignment.

Nmap does a good job documenting the ports occupied by more commonly used closed protocols, but it would be a sisyphean task to document all the protocols used across the entire Internet.

## Customized Port Assignments

Just because there is an official list of port assignments, this doesn’t prevent anyone from configuring their services on other ports. In practice most instances of a service will use its de facto port, and the few variants are too minor to record in nmap-services.

Additionally, because some port numbers are more memorable by humans than others, many services cluster around certain port numbers, such as 8888 or 1337. Each of these cases can result in unknown or misidentified services.

# Extending Nmap’s definitions

Each unknown port represents an opportunity to discover something new about your environment. If you have access to the system, then you can start your research by running `sudo netstat -anp` (or `netstat -anb` in windows). IoT devices are another story, but documentation may provide the answer. Worst case, capturing the traffic and reviewing it in Wireshark may be enlightening.

Once you’ve determined the application or protocol that’s using the port, make a copy of the nmap-services file and edit it to include the missing information. As you can see from the screenshot of the nmap-services file included above, the file is easy to humanly interpret and edit. You can leave the open-frequency percentage blank, or leave it as 0.

In Linux, you can store your copy of nmap-services in ~/.nmap and it’ll be used automatically. If you’re managing multiple environments (or using Windows) then place your copy of nmap-services in an aptly named folder or directory, and use --datadir to tell Nmap where to find it.

Remember, always be scanning and exploring the unknown. Special thanks to Ploppowaffles for review and feedback on this post.

