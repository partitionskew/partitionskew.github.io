+++
title = 'Computer Networking - Part 1: IP addresses'
summary = 'A quick primer on IP addresses, CIDR notation, and subnets'
tags = ["networking", "introduction", "ip address"]
date = 2024-09-07
showToc = true
draft = false
+++

## Introduction

In this series, I explain the fundamental concepts that underpin computer networking.

In part 1, we discuss what an IP address is. In [part 2](/posts/2024/07-sep/2024-09-14_networking_2), we'll talk about what happens behind the scenes when you enter a website like en.wikipedia.org in your web browser.

_Note: this article assumes you are familiar with the binary number system_

## Internet Protocol (IP) address

Every machine and device that you can reach on the public internet has a public IP address. A regular home address is needed for the postal office to be able to deliver your mail to the right person. Similarly, an IP address functions the same way to ensure that your data and requests make it to the correct web application.

An IP address is a numeric identifier consisting of 4 eight-bit numbers delimited by a period. For example, 0.0.0.0 and 255.255.255.255 are both valid IP addresses. As a reminder, an 8-bit binary number can represent any decimal number between 0 to 255 (inclusive)

If you do the math, this works out to 2^8 x 2^8 x 2^8 x 2^8 = 2^32 = ~4.3 billion unique IP addresses.

### Reserved IP addresses

Not every IP address is available for public use. Certain ranges of IP addresses are reserved for special use cases such as _localhost_ (127.0.0.1) which refers to the machine that an application is running on. Another reserved set of IP address ranges is the private IP address range which encompasses:
- 10.0.0.0 - 10.255.255.255
- 172.16.0.0 - 172.31.255.255
- 192.168.0.0 - 192.168.255.255

You can't directly reach a private IP address over the public internet. Private IP addresses are not unique. In my home network, I can have a mobile phone with a private IP address of 192.168.123.88. On your home network, maybe your smart fridge has the exact same private IP address. 

You might be wondering what the point of private IP addresses are at all. Unsurprisingly, private IP addresses are very useful. However, to fully leverage private IP addresses requires routers, modems, and something called network address translation (NAT). We'll cover this in another post.

### IPv4 vs. IPv6 addresses

The IP addresses we've discussed are actually IPv4 addresses. The Internet Protocol went through several iterations and the 4th version uses 32 bits to address devices on the network. However, there's over 15 billion mobile devices in the world let alone all the other types of devices that can surf the web. If there's only 4.3 billion unique IPv4 addresses, how can we make sure every network-connected device is reachable?

There are two solutions. One is to use private IP addresses and NAT. We'll discuss this in another post.

Another option is to use IPv6 addresses. Unlike IPv4, IPv6 allocates 128 bits to the address. Instead of 4 eight-bit digits (32 bits total), IPv6 uses 8 groups of 4 hexadecimals that are colon-delimited. Since each hexadecimal is 4-bits big, each IPv6 address therefore requires 128 bits (8 groups * 4 hexadecimals * 4 bits per hexadecimal = 128). 

IPv6 addresses range from 0:0:0:0:0:0:0:0 to ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff. In other words, there's a lot of IPv6 addresses and we're at no risk of running out any time soon.

> As a quick reminder, hexadecimal is a base-16 numbering system. Whereas the binary number system has two possible values for each digit (0 & 1) and decimal has 10 possible values (0 - 9), hexadecimal has 16 possible values (0 - 9 & a - f). Representing 16 possible values requires at least 4 bits (2^4 = 16). Ergo each hexadecimal has a corresponding 4-bit representation. 

## CIDR notation

As an aside, CIDR notation is a way to succinctly describe ranges of IP addresses. The way that it works is every CIDR block consists of two parts:
* the IP address prefix
* a number between 0 - 32 which indicates how many leading bits are fixed

For example, the IP address range 10.1.2.0 - 10.1.2.255 can be expressed as the CIDR block, 10.1.2.0/24. The way to interpret this is that for every single IP address in this range, the first 24 bits are always the same. They are always 10.1.2 (base-10) or 0001000.00000001.00000010 (base-2). The remaining 8 bits are variable. In total, this CIDR block describes a range of 2^8 = 256 IP addresses.

The number does not need to be a multiple of 8 although it's easier to reason about. For example, consider the following CIDR block, 192.168.0.0/22. Since the first 22 bits are fixed, this means the last 10 bits are variable. 2^10 means this CIDR block represents an address range of 1024 unique IPs.

To get the actual IP address range, let's convert the IPv4 address to its binary form:
- 11000000.10101000.00000000.00000000

The first 22 bits are fixed: 11000000.10101000.000000

The last 10 bits are variable and can range from a minimum of 00.00000000 to a maximum of 11.11111111

Hence, the IP address range denoted by the CIDR block, 192.168.0.0/22 is:
- 11000000.10101000.00000000.00000000 to 11000000.10101000.00000011.11111111 (binary)
- 192.168.0.0 to 192.168.3.255 (decimal)

## Subnets

Finally, we'll briefly discuss what a _subnet_ is. A subnet is simply a subset of a bigger network. If the entire internet consists of approximately 4.3 billion IP addresses, then a subnet is merely a smaller portion of it. For example, 192.168.0.0/16 is a subnet consisting exclusively of private IP addresses.

With CIDR blocks, you can easily carve out smaller subnets. Why might you want to do this? Imagine you're a business that has been granted a /16 subnet. This means you have exactly 65,536 IP addresses to work with. Since your organization consists of several teams, you may want to divide up the subnet into even smaller subnets to be distributed to each team. 

This way, a single team can't exhaust all the IP addresses and leave the other teams out to dry. For example, you may divide up your /16 subnet into two /17 subnets (32,768 IP addresses each) or 4 /18 subnets (16,384). 

Remember, binary works in powers of 2.

## References

### Wikipedia

* [Reserved IP addresses](https://en.wikipedia.org/wiki/Reserved_IP_addresses)


### DigitalOcean

* [Understanding IP addresses, subnets, and CIDR notation for networking](https://www.digitalocean.com/community/tutorials/understanding-ip-addresses-subnets-and-cidr-notation-for-networking#cidr-notation)
