---
title: Multiroom Audio with Librespot and Snapcast
date: 2025-02-20
categories:
  - Multiroom Audio
tags:
  - Multiroom Audio
published: False
draft: False
---


[Snapcast](https://github.com/badaix/snapcast) is an open-source synchronous multiroom client-server audio player and [Librespot](https://github.com/librespot-org/librespot) is an open source client library for Spotify - a perfect combination to create a free multiroom audio solution. Librespot works as a spotify connect receiver and can be controlled by regular Spotify apps, while Snapcast serves time synchronized audio to snapcast clients over the network (wifi).

<!-- more -->

While Librespot and Snapcast support several backends and sources, respectively, this article focuses on using Alsa backend.(1)
{ .annotate }

1.  :man_raising_hand: Though `pipe` backend works, snapcast appears to
    intermittently crash when using `pipe`

## Virtual Sound Card

As shown in Snapcast's documentation, we'll use the `snd-aloop` kernel module to create a virtual sound card.

```bash
sudo modprobe snd-aloop
```

To load this kernel module at boot, add the `snd-aloop` to `/etc/modules`

```bash
echo snd-aloop | sudo tee -a /etc/modules
```

The following output shows the sound cards before loading the module

```bash
red@avani:~$ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: ALC283 Analog [ALC283 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 3: HDMI 0 [HDMI 0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 7: HDMI 1 [HDMI 1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 8: HDMI 2 [HDMI 2]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 9: HDMI 3 [HDMI 3]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: PCH [HDA Intel PCH], device 10: HDMI 4 [HDMI 4]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

Output of `aplay -l` after rebooting the host
```bash
red@avani:~$ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: Loopback [Loopback], device 0: Loopback PCM [Loopback PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 0: Loopback [Loopback], device 1: Loopback PCM [Loopback PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 1: PCH [HDA Intel PCH], device 0: ALC283 Analog [ALC283 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 3: HDMI 0 [HDMI 0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 7: HDMI 1 [HDMI 1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 8: HDMI 2 [HDMI 2]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 9: HDMI 3 [HDMI 3]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 10: HDMI 4 [HDMI 4]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
red@avani:~$
```

!!! note
    The Loopback card might show up with a different id when loading the kernel moduel using the `modprobe snd-aloop` command as opposed to having the module load at boot.

Notice the card number, device number and the sub-device numbers on the entries labelled Loopback. They form the hardware id in the format `cardnumber,devicenumber,subdevicenumber, (Ex. hw:0,0,5)`. Card number indicates the hardware ID the OS recognized, and the two devices (with id `0` and `1`) form a loop. Audio sent to device `0` on a sub-device `(Ex. hw:0,0,5)` will be delivered (looped) on device `1` on the same sub-device `(Ex. hw:0,1,5)`.

## Librespot 

Though linux package managers include librespot, they are usually not up to date. Librepot can be compiled with required features. Install the OS related dependecies as mentioned in librespot [github](https://github.com/librespot-org/librespot) as well as Rust to compile the software.

Check Version
```bash
librespot --version
```

### Config

The Alsa sound card will be passed device to librespot along with a few other parameters such as player name, bitrate, etc. Following are some common parameters passed. Store these parameters in a config file `/etc/librespot/librespot.conf`

```bash

cat <<EOF | sudo tee /etc/librespot/librespot.conf

name=Home-Multiroom #Name of the spotify player
backend=alsa #Since we are using alsa in this example
device=hw:0,0,0 # Id of the cardnumber,devicenumber,subdevicenumber
bitrate=320 
initial_volume=80
device_type=avr #Device type indicator when it shows up in Spotify players list
cache_dir=/tmp/spotify/

EOF
```

### Librespot systemd service file

Setup librespot to run as a systemd service

```bash

cat <<EOF | sudo tee /etc/systemd/system/librespot.service

[Unit]
Description=Librespot Spotify Client
After=network.target
Wants=snapserver.service

[Service]
EnvironmentFile=/etc/librespot/librespot.conf

ExecStart=/usr/bin/librespot \
    --name ${name} \
    --backend ${backend} \
    --device ${device} \
    --bitrate ${bitrate} \
    --system-cache ${cache_dir} \
    --cache ${cache_dir} \
    --initial-volume ${initial_volume} \
    --device-type ${device_type} \

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

EOF
```

### Run librespot

Run `sudo systemctl daemon-reload` to load the new service file and then start the service with `sudo systemctl enable librespot --now`. View the status of librespot using `systemctl status librespot` and use `journalctl -u librespot` to view the librespot's systemd service logs, if needed.


## Snapserver 

Download and install snapserver packages for the distribution from [releases](https://github.com/badaix/snapcast/releases) page.

### Config

The Snapcast server receives audio on the other device of the virtual sound card. Similar to librespot, snapserver will take the hardware id, `hw:0,1,0` in this case.

```bash

cat <<EOF | sudo tee /etc/snapserver.conf

[stream]
source = alsa:///?name=multiroom&device=hw:0,1,0&sampleformat=44100:16:2

[http]
enabled = true
bind_to_address = 0.0.0.0
port = 1780
doc_root = /usr/share/snapserver/snapweb

[tcp]
enabled = true
bind_to_address = 0.0.0.0
port = 1705

EOF
```

In addition, the builtin snapweb can be reached using port `1780 (http) or 1788 (https)`.

<div style="display: flex; justify-content: center;">
    <a href="snapweb.png" class="image-popup"><img src="snapweb.png" alt="snapweb" title="snapweb" width="800" height="300"></a>
</div>

### Run snapserver

Start the service with `sudo systemctl enable snapserver --now`. View the status of librespot using `systemctl status snapserver` and use `journalctl -u snapserver` to view the snapserver's systemd service logs, if needed.



