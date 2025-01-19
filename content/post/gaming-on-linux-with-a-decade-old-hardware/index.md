---
title: Gaming on Linux with a decade old hardware
description: Fedora Linux for gaming with very old hardware
slug: gaming-on-linux-with-a-decade-old-hardware
date: 2024-01-21 00:00:00+0000
#image: cover.jpg
categories:
    - linux
tags:
    - linux
    - gaming
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# A retrospective 

Ten years ago I have used my exact same workstation hardware to try my first linux desktop experience.
My hardware is: 
- Intel Xeon CPU E3-1231 v3
- Nvidia GTX 970
  - broke twice, but one time within warranty
  - replaced it with a GTX 1050Ti
- 16 GB DDR3 RAM
As you can tell, that is not really a powerul hardware nowadays anymore.

Back then I was using Ubuntu in dual boot with Windows 7. As I was not sure, if Linux would actually work as I hoped it to work, I didn't dare to completely wipe my Windows. Fortunately, I haven't done it as my Linux desktop experience was quite tedious. The biggest pain points I was facing were: 
- I was not that familiar with Linux itself (at least compared to now) 
- Installing the correct hardware drivers was a nightmare 
- Most games didn't offer native Linux support thus you had to rely heavily on [wine](https://www.winehq.org/)
- Almost no native gaming clients
- I could not get some games working on linux
- Looking at forums you occasionally found threads about people complaining to be banned, because they were playing on a non supported operating system 

# Why now? 
As time has passed, I was evaluating if I want to migrate my Windows 10 to Windows 11. After checking the [system requirements for Windows 11](https://www.microsoft.com/en-us/windows/windows-11-specifications), I found out that I my PC lacking on the following parts: 
- Convert Master Boot Record (MBR) to GUID Partition Table (GPT) to enable UEFI boot
- No Trusted Platform Module (TPM)
- Unsupported processor

While converting the MBR to GPT was a software issue and can be [solved with this guide](https://www.windowscentral.com/how-convert-mbr-disk-gpt-move-bios-uefi-windows-10), the remaining items are hardware issues. There might be the possibility to work around this, but I didn't like the direction Windows was aiming for anyway. Consequently, I have decided to give Linux another chance. 

# The Linux Experience 

## Which Distro? 

One of the first questions which come up, when you think about Linux is, which distribution to use? Given that I work for [Red Hat](https://www.redhat.com/en), I have decided to use [Fedora Workstation](https://fedoraproject.org/). This is mostly due to the [Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux) like behavior.

## Usability

## Are all problems solved now?
Let me elaborate on the problems mentioned earlier. 

### Drivers
Per default Fedora was shipped with the [NVIDIA nouveau](https://nouveau.freedesktop.org/) drivers. Those drivers worked well for me on native Linux games. However, they performed very poorly on games which were relying on Wine. Yet again I had to install the GPU drivers from NVIDIA. To my surprise this was a very easy task. The software center had automatically detected which third party driver my system would need and I could install them very comfortably via the GUI.
![](hardware_drivers.png)

### Gaming clients
Unfortunately some gaming clients like the [Blizzard Battle.net](https://www.blizzard.com/en-us/) or [GOG Galaxy](https://www.gog.com/wishlist/galaxy/release_the_gog_galaxy_client_for_linux) do not offer native Linux support. Consequently you will need to take a look at other solutions.
Currently there are differnet gaming clients available. The most common ones, besides [Steam](https://steamcommunity.com/), are [Heroic Games Launcher](https://heroicgameslauncher.com/) and [Lutris](https://lutris.net/). Both of those clients have their pros and cons. Lutris allows you to connect to the Battle.net
![](lutris_battle_net.png) but does not offer cloud save files for the GOG Galaxy. While the Heroic Games Launcher does support the cloud saves, you cannot use it for the Blizzard games. Consequently, there is no one size fits all solution.

... to be continued ...