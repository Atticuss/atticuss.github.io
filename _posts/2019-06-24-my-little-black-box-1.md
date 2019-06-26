---
layout: post
title: "My Little Black Box -- Creation"
date: 2019-06-25 00:00:00 -0700
tags: 
- odroid
- pine64
published: true
description: Running an at-home k8s cluster on top of four Pine64 Rock64 SBCs and a distributed NAS on two Odroid HC1 SBCs.
---

Kubernetes (or k8s, as all the cool kids call it) is all the rage these days. And while I'm perfectly happy learning a concept via toy examples and reading documentation, there really is something to be said for the hands-on-keyboard experience. As such, I decided to build a cheap-ish at-home k8s deployment using a cluster of SBCs. I've also been needing to get a proper NAS setup at home, instead of relying on a handful of external HDD for things.

## SBC Selection

When it comes to SBCs, many folk just default to the Raspberry Pi and call it a day. I was originally one of those folk until I decided to explore a bit more of what's out there. One of the most glaring weaknesses of the RasPi boards was the network throttling due to USB and ethernet sharing a bus. Another is the lack of RAM.

With the goal of resolving these two issues, I began looking into Odroid. I was intrigued by the [MC1 cluster](https://ameridroid.com/collections/single-board-computer/products/odroid-mc1-cluster) as the aluminum casing helps with heat dissipation. It's also on the cheaper side when you consider you don't need to buy four single cases or some sort of third-party cluster rack mount. However, the low RAM count (only 2GB per board) eventually turned me towards the 4GB variant of Pine64's [Rock64](https://ameridroid.com/products/rock64) board. If you plan on picking one of these up, you can get them a few bucks cheaper from Pine64 directly. But be warned: you'll have to wait 3-4 weeks for the production run in China to hit your doorstep.

For the NAS, the [Odroid HC1](https://ameridroid.com/collections/single-board-computer/products/odroid-hc1) was the obvious choice. As I only planned to run glusterfs, the 2GB of RAM was fine and the sled built to hold a 2.5" drive via SATA was far too convenient to pass up. As for the drives, I went with a pair of [2TB Seagate Barracudas](https://smile.amazon.com/gp/product/B01LX13P71/). Not exactly the best drives around, but having to use a 2.5" limits your options. It's worth noting that since I've started this project, the [HC2](https://ameridroid.com/products/odroid-hc2) has been released. At only a few bucks more, it's small bit beefier but also sports a sled that hold a 3.5" drive. Needless to say, I see a pair of HC2s and 4TB WD Reds in my future.

## Additional Parts

Beyond the boards themselves, there's a few other peripherals to pick up. First up, for power, most people pick up barrel plug wall wart. This wasn't exactly ideal, though, considering that I had planned to get four Rock64s and two HC1s. Instead, I picked up a [5V/30A power supply](https://smile.amazon.com/LETOUR-Supply-Converter-Adapter-Lighting/dp/B01HJA3OUG/ref=sr_1_3?keywords=letour+5v+30a&qid=1561505939&s=gateway&sr=8-3), the kind typically used for LEDs. Combined with some [barrel plug pigtails](https://www.monoprice.com/product?p_id=6880) and [2.1mm to 2.55 barrel plug converters](https://smile.amazon.com/CGTime-5-5x2-5mm-DC5-5x2-1mm-Connector-DC5-5x2-5mm/dp/B07L5GGW7Q), I could wire all my boards up while only consuming a single electrical outlet. The barrel plug converters were a bit pricy, but you could probably find something cheaper on Ali Express if you're willing to wait. The PSU also doesn't come with a cable, but you can just lop off the female end of any standard monitor/PC/server power cable, twist the wires, and clamp them down.

![Wiring up the PSU](/assets/images/blog/2019-06-24-my-little-black-box-1/psu-closeup.jpg)

I also planned to throw everything into a small rack, so found a 6U and shelf for cheap off Ebay. While the HC1s could stack and just sit on the shelf, I wanted something for Rock64s. While I could get individual cases for each board, I went with a slightly more elegant [3D printed rack](https://www.etsy.com/listing/544554171/sbc-storage-rack-rackmount-compatible) from Etsy, mounted with some M2.5 screws I had laying around. Once summer hit, I started noticing the boards were running hot, even with the AC on, so also picked up a pair of [USB-powered 80mm fans](https://www.amazon.com/gp/product/B002NVC1DS/) that I just slapped onto the back of the rack mount.

Last thing I needed was a few switches and several ethernet cables. The switches I already had laying around, including a small managed one which allows for port mirroring, but I did end up grabbing a bunch of 1' ethernet cables off Monoprice for cheap.

Overall, I ended up paying around $450 for everything. While not exactly cheap, it's still cheaper than the GPU upgrade I've been pining for.

![Final Product](/assets/images/blog/2019-06-24-my-little-black-box-1/little-black-box.jpg)

## Usecases

As I said, the main goal is to run k8s on top of the four Rock64s, with one of them acting as a master. I also planned to mount the NAS on the all the k8s nodes, using it as a volume mount for pods. This allowed me to just buy some small class 10 micro SDs for the OS without having to worry about containers that need lot sof storage. Though the availability of eMMC on the Rock64s would be nice if I ever need high speed IO. 

The first thing I implemented is a pihole instance with cloudflared set upstream. Pihole provides network-wide ad-blocking while using cloudflared allows for DNS-over-HTTPS. I then got ELK stood up, using Filebeat to forward the pihole logs. My latest deployment is a Py3 script that scrapes a bunch of metrics from Coinbase once per minute. Gotta use that NAS space for something, and the sooner I fill it up, the sooner I can justify some HC2 and WD Reds. 

I set up a Github repo with all my images and YAML files [here](https://github.com/atticuss/little-black-box), but I'll do some more detailed write-ups in the future on the details of each little project.