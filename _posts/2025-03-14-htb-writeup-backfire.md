---
title: Backfire &#x1F512;
description: La maquina Backfire esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-03-14
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-backfire/backfire_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - linux
  - hack the box
  - tcp
  - ssh
  - http
  - https

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ ping -c 1 10.10.11.49
PING 10.10.11.49 (10.10.11.49) 56(84) bytes of data.
64 bytes from 10.10.11.49: icmp_seq=1 ttl=63 time=177 ms

--- 10.10.11.49 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 177.029/177.029/177.029/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.49 -n -Pn -oG nmap1
Host: 10.10.11.49 ()    Status: Up
Host: 10.10.11.49 ()    Ports: 22/open/tcp//ssh///, 443/open/tcp//https///, 8000/open/tcp//http-alt///
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/backfire:-$ sudo nmap -sCV -p22,443,8000 -vvv 10.10.11.49 -oN nmap2
PORT     STATE SERVICE  REASON         VERSION
22/tcp   open  ssh      syn-ack ttl 63 OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey: 
|   256 7d:6b:ba:b6:25:48:77:ac:3a:a2:ef:ae:f5:1d:98:c4 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJuxaL9aCVxiQGLRxQPezW3dkgouskvb/BcBJR16VYjHElq7F8C2ByzUTNr0OMeiwft8X5vJaD9GBqoEul4D1QE=
|   256 be:f3:27:9e:c6:d6:29:27:7b:98:18:91:4e:97:25:99 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA2oT7Hn4aUiSdg4vO9rJIbVSVKcOVKozd838ZStpwj8
443/tcp  open  ssl/http syn-ack ttl 63 nginx 1.22.1
|_http-server-header: nginx/1.22.1
| http-methods: 
|_  Supported Methods: GET
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
|_http-title: 404 Not Found
| ssl-cert: Subject: commonName=127.0.0.1/organizationName=LTD/stateOrProvinceName=/countryName=US/postalCode=2166/streetAddress=/localityName=
| Subject Alternative Name: IP Address:127.0.0.1
| Issuer: commonName=127.0.0.1/organizationName=LTD/stateOrProvinceName=/countryName=US/postalCode=2166/streetAddress=/localityName=
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-10-04T10:01:35
| Not valid after:  2027-10-04T10:01:35
| MD5:   1045:c8ca:2888:1dcb:7921:0d4b:81c6:41cb
| SHA-1: 8352:0976:4835:0fe4:fafc:2c68:bd07:f840:8658:aa65
| -----BEGIN CERTIFICATE-----
| MIIDuzCCAqOgAwIBAgIRAO7Ik+Hd1u4PgVElHC2C2oAwDQYJKoZIhvcNAQELBQAw
| XzELMAkGA1UEBhMCVVMxCTAHBgNVBAgTADEJMAcGA1UEBxMAMQkwBwYDVQQJEwAx
| DTALBgNVBBETBDIxNjYxDDAKBgNVBAoTA0xURDESMBAGA1UEAxMJMTI3LjAuMC4x
| MB4XDTI0MTAwNDEwMDEzNVoXDTI3MTAwNDEwMDEzNVowXzELMAkGA1UEBhMCVVMx
| CTAHBgNVBAgTADEJMAcGA1UEBxMAMQkwBwYDVQQJEwAxDTALBgNVBBETBDIxNjYx
| DDAKBgNVBAoTA0xURDESMBAGA1UEAxMJMTI3LjAuMC4xMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEAz6YS73tjhd1KVFsNtUfXzS0XjCkt11uL6TprYKVf
| Wjgs8RhmjEjWcQEJkHDcCjH5I/rlmqdCLdj2aBuZpRGRBs00mgPwko2EscyeqoWS
| usi5R7QNjZih+7p486kq3rJfxSSAsr/ym6tjxKwVyXxyiE0+e002Kozyge7CW9YM
| RyEUA3N6Je8jz9YtIh5gnmSJorF700zMJWW8gxGmKRGDwAGegzQNNTWTPDHclC4u
| JEdbj7hk4nxkLwBFaYjgbVW2pHrjUXJBELInsPFveQLD77lfkThLgwFERKzeQQ2y
| 4mJijD6HQEiAPCdZKjJG/vEZapDJc00hLn3ggB3R19v1aQIDAQABo3IwcDAOBgNV
| HQ8BAf8EBAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1Ud
| EwEB/wQFMAMBAf8wHQYDVR0OBBYEFKgcqcbeYNgRnjRe+we2+Ley6leOMA8GA1Ud
| EQQIMAaHBH8AAAEwDQYJKoZIhvcNAQELBQADggEBADSdO/WATVHu1XpM0Geotz+O
| c2UkAD4io8P69V8SU5/ptVfZsMxYCf5QoriDPLPGIwgd1EL6ghNrEu0wxLFEF+xE
| piKglcwF8Hbaz7kSx+E80XdBsXoUghrwEGI/Y00BsmGT/GQ4bu4OLftAbCYu/pwd
| QVYaIXj3m7rdfSIDPKuDpk9n2Hs5HuKrsHXi02wQYANTdSa/UGYd2bf9jYnteM75
| K26iQ9QaSV9ATzk8vV1dp5NtDXsBnninufiw49Rt597DA0ErZkuawSRX4wZfvNVU
| 2hbOYe33/zj/77mmWtW3gBGoUMt6ajARs+2dBiJNX5nZp31w9nElr5pXkDzQJkM=
|_-----END CERTIFICATE-----
8000/tcp open  http     syn-ack ttl 63 nginx 1.22.1
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-title: Index of /
|_http-server-header: nginx/1.22.1
| http-ls: Volume /
| SIZE  TIME               FILENAME
| 1559  17-Dec-2024 12:31  disable_tls.patch
| 875   17-Dec-2024 12:34  havoc.yaotl
|_
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


> <a href="https://www.hackthebox.com/achievement/machine/1521382/643" target="_blank">Backfire Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }