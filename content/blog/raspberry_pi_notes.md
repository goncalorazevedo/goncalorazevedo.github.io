+++
title = "Raspberry Pi Notes"
draft = true
date = 2026-10-16
taxonomies.tags = [
  "raspberrypi", 
  "linux"
]
+++
Notes for personal reference.

First; do not(!) simply flash the image to the SD card, setup the image before plugging it in as well, for example, use `rpie-imager`: it lets you setup the hostname, add a WiFi connection, enable ssh and add your public key into most OS images. (avoid a tty with only wpa_supplicant)

Second; once connected find the IP address via the Mac addresses, list [here](https://udger.com/resources/mac-address-vendor-detail?name=raspberry_pi_foundation). If none show up it might be because it's not present in the arp table or the Pie might not be connected, if that is the case analyze the router directly.
```
> ifconfig
en0: ...
  inet 10.179.142.16 netmask 0xffffff00 broadcast 10.179.142.255
  
# subnet is 10.179.142.0/24
> for i in {1..254}; do ping -c1 -W100 10.179.142.$i >/dev/null & done 
> arp -an
? (10.179.142.16) at **************** on en0 ifscope permanent [ethernet]
? (10.179.142.161) at dc:a6:32:******** on en0 ifscope [ethernet] # << Raspberry Pi
? (10.179.142.178) at **************** on en0 ifscope [ethernet]
? ...
```

Third; install and enable `avahi-daemon` (if not already present in the image) so you get mDNS and you can resolve with `hostname.local`, if you need to get the IP it's very simple to do so from your computer.
```
> dns-sd -G v4 raspberrypi.local
DATE: ---Fri 16 Oct 2026---
21:21:54.476  ...STARTING...
Timestamp     A/R  Flags         IF  Hostname               Address                TTL
21:21:54.813  Add  2             17  raspberrypi.local.     10.179.142.161         120
```