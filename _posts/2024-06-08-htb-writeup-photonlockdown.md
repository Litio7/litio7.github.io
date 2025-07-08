---
title: Photon Lockdown
description: We've located the adversary's location and must now secure access to their Optical Network Terminal to disable their internet connection. Fortunately, we've obtained a copy of the device's firmware, which is suspected to contain hardcoded credentials. Can you extract the password from it?
date: 2024-06-08
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-challenges/hardware_logo.png
categories:
  - Hack The Box
  - Challenges
tags:
  - hack the box
  - hardware
  - data leaks
  - data leak exploitation

---
## Data Leak Exploitation

Se extraen los archivos del firmware.

```terminal
/home/kali/Documents/htb/challenges/photonlockdown:-$ unzip photon_lockdown.zip
fwu_ver
hw_ver
rootfs
```

El archivo rootfs es identificado como un sistema de archivos SquashFS.

```terminal
/home/kali/Documents/htb/challenges/photonlockdown:-$ file rootfs
rootfs: Squashfs filesystem, little endian, version 4.0, zlib compressed, 10936182 bytes, 910 inodes, blocksize: 131072 bytes, created: Sun Oct  1 07:02:43 2023
```

Descomprimo el sistema de archivos. Y busco cualquier referencia a "HTB" dentro.

```terminal
/home/kali/Documents/htb/challenges/photonlockdown:-$ sudo unsquashfs -d root rootfs

/home/kali/Documents/htb/challenges/photonlockdown/root:-$ grep -rl 'HTB'
bin/ip
bin/tc
etc/config_default.xml
```

Por ultimo, encuentro la flag dentro del archivo `config_default.xml`.

```terminal
/home/kali/Documents/htb/challenges/photonlockdown:-$ grep -i 'HTB' etc/config_default.xml
< Value Name="SUSER_PASSWORD" Value="HTB{N0w_Y0u_C4n_L0g1n}"/>
```

> <a href="https://www.hackthebox.com/achievement/challenge/1521382/548" target="_blank">Photon Lockdown Challenge from Hack The Box has been Pwned</a>
{: .prompt-tip }