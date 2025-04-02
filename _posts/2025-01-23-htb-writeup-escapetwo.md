---
title: EscapeTwo &#x1F512;
description: La maquina EscapeTwo esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-01-23
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-escapetwo/escapetwo_logo.png
categories:
  - Hack The Box
  - Machines
tags:
  - windows
  - hack the box
  - tcp
  - dns
  - smb
  - kerberos
  - ldap
  - winrm

---
### Machine Information

As is common in real life Windows pentests, you will start this box with credentials for the following account: rose / KxEPkKe6R8su

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ ping -c 1 10.10.11.51
PING 10.10.11.51 (10.10.11.51) 56(84) bytes of data.
64 bytes from 10.10.11.51: icmp_seq=1 ttl=127 time=262 ms

--- 10.10.11.51 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 262.486/262.486/262.486/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.51 -n -Pn -oG nmap1
Host: 10.10.11.51 ()    Status: Up
Host: 10.10.11.51 ()    Ports: 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 1433/open/tcp//ms-sql-s///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 5985/open/tcp//wsman///, 9389/open/tcp//adws///, 47001/open/tcp//winrm///, 49664/open/tcp/////, 49665/open/tcp/////, 49666/open/tcp/////, 49667/open/tcp/////, 49685/open/tcp/////, 49686/open/tcp/////, 49687/open/tcp/////, 49692/open/tcp/////, 49716/open/tcp/////, 49725/open/tcp/////, 49793/open/tcp/////, 60313/open/tcp/////  Ignored State: filtered (65508)
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ sudo nmap -sCV -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,47001,49664,49665,49666,49667,49685,49686,49687,49692,49716,49725,49793,60313 10.10.11.51 -vvv -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-01-11 21:19:16Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.sequel.htb
| Issuer: commonName=sequel-DC01-CA/domainComponent=sequel
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-06-08T17:35:00
| Not valid after:  2025-06-08T17:35:00
| MD5:   09fd:3df4:9f58:da05:410d:e89e:7442:b6ff
| SHA-1: c3ac:8bfd:6132:ed77:2975:7f5e:6990:1ced:528e:aac5
| -----BEGIN CERTIFICATE-----
| MIIGJjCCBQ6gAwIBAgITVAAAAANDveocXlnSDQAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBGMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGc2VxdWVs
| MRcwFQYDVQQDEw5zZXF1ZWwtREMwMS1DQTAeFw0yNDA2MDgxNzM1MDBaFw0yNTA2
| MDgxNzM1MDBaMBoxGDAWBgNVBAMTD0RDMDEuc2VxdWVsLmh0YjCCASIwDQYJKoZI
| hvcNAQEBBQADggEPADCCAQoCggEBANRCnm8pZ86LZP3kAtl29rFgY5gEOEXSCZSm
| F6Ai+1vh6a8LrCRKMWtC8+Kla0PXgjTcGcmDawcI8h0BsaSH6sQVAD21ca5MQcv0
| xf+4TzrvAnp9H+pVHO1r42cLXBwq14Ak8dSueiOLgxoLKO1CDtKk+e8ZxQWf94Bp
| Vu8rnpImFT6IeDgACeBfb0hLzK2JJRT9ezZiUVxoTfMKKuy4IPFWcshW/1bQfEK0
| ExOcQZVaoCJzRPBUVTp/XGHEW9d6abW8h1UR+64qVfGexsrUKBfxKRsHuHTxa4ts
| +qUVJRbJkzlSgyKGMjhNfT3BPVwwP8HvErWvbsWKKPRkvMaPhU0CAwEAAaOCAzcw
| ggMzMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABs
| AGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQD
| AgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQME
| AgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglg
| hkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFNfVXsrpSahW
| xfdL4wxFDgtUztvRMB8GA1UdIwQYMBaAFMZBubbkDkfWBlps8YrGlP0a+7jDMIHI
| BgNVHR8EgcAwgb0wgbqggbeggbSGgbFsZGFwOi8vL0NOPXNlcXVlbC1EQzAxLUNB
| LENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERDPWh0Yj9jZXJ0aWZp
| Y2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0
| aW9uUG9pbnQwgb8GCCsGAQUFBwEBBIGyMIGvMIGsBggrBgEFBQcwAoaBn2xkYXA6
| Ly8vQ049c2VxdWVsLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERD
| PWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlv
| bkF1dGhvcml0eTA7BgNVHREENDAyoB8GCSsGAQQBgjcZAaASBBDjAT1NPPfwT4sa
| sNjnBqS3gg9EQzAxLnNlcXVlbC5odGIwTQYJKwYBBAGCNxkCBEAwPqA8BgorBgEE
| AYI3GQIBoC4ELFMtMS01LTIxLTU0ODY3MDM5Ny05NzI2ODc0ODQtMzQ5NjMzNTM3
| MC0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCBDjlZZbFac6RlhZ2BhLzvWmA1Xcyn
| jZmYF3aOXmmof1yyO/kxk81fStsu3gtZ94KmpkBwmd1QkSJCuT54fTxg17xDtA49
| QF7O4DPsFkeOM2ip8TAf8x5bGwH5tlZvNjllBCgSpCupZlNY8wKYnyKQDNwtWtgL
| UF4SbE9Q6JWA+Re5lPa6AoUr/sRzKxcPsAjK8kgquUA0spoDrxAqkADIRsHgBLGY
| +Wn+DXHctZtv8GcOwrfW5KkbkVykx8DSS2qH4y2+xbC3ZHjsKlVjoddkjEkrHku0
| 2iXZSIqShMXzXmLTW/G+LzqK3U3VTcKo0yUKqmLlKyZXzQ+kYVLqgOOX
|_-----END CERTIFICATE-----
|_ssl-date: 2025-01-11T21:21:03+00:00; 0s from scanner time.
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.sequel.htb
| Issuer: commonName=sequel-DC01-CA/domainComponent=sequel
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-06-08T17:35:00
| Not valid after:  2025-06-08T17:35:00
| MD5:   09fd:3df4:9f58:da05:410d:e89e:7442:b6ff
| SHA-1: c3ac:8bfd:6132:ed77:2975:7f5e:6990:1ced:528e:aac5
| -----BEGIN CERTIFICATE-----
| MIIGJjCCBQ6gAwIBAgITVAAAAANDveocXlnSDQAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBGMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGc2VxdWVs
| MRcwFQYDVQQDEw5zZXF1ZWwtREMwMS1DQTAeFw0yNDA2MDgxNzM1MDBaFw0yNTA2
| MDgxNzM1MDBaMBoxGDAWBgNVBAMTD0RDMDEuc2VxdWVsLmh0YjCCASIwDQYJKoZI
| hvcNAQEBBQADggEPADCCAQoCggEBANRCnm8pZ86LZP3kAtl29rFgY5gEOEXSCZSm
| F6Ai+1vh6a8LrCRKMWtC8+Kla0PXgjTcGcmDawcI8h0BsaSH6sQVAD21ca5MQcv0
| xf+4TzrvAnp9H+pVHO1r42cLXBwq14Ak8dSueiOLgxoLKO1CDtKk+e8ZxQWf94Bp
| Vu8rnpImFT6IeDgACeBfb0hLzK2JJRT9ezZiUVxoTfMKKuy4IPFWcshW/1bQfEK0
| ExOcQZVaoCJzRPBUVTp/XGHEW9d6abW8h1UR+64qVfGexsrUKBfxKRsHuHTxa4ts
| +qUVJRbJkzlSgyKGMjhNfT3BPVwwP8HvErWvbsWKKPRkvMaPhU0CAwEAAaOCAzcw
| ggMzMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABs
| AGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQD
| AgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQME
| AgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglg
| hkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFNfVXsrpSahW
| xfdL4wxFDgtUztvRMB8GA1UdIwQYMBaAFMZBubbkDkfWBlps8YrGlP0a+7jDMIHI
| BgNVHR8EgcAwgb0wgbqggbeggbSGgbFsZGFwOi8vL0NOPXNlcXVlbC1EQzAxLUNB
| LENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERDPWh0Yj9jZXJ0aWZp
| Y2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0
| aW9uUG9pbnQwgb8GCCsGAQUFBwEBBIGyMIGvMIGsBggrBgEFBQcwAoaBn2xkYXA6
| Ly8vQ049c2VxdWVsLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERD
| PWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlv
| bkF1dGhvcml0eTA7BgNVHREENDAyoB8GCSsGAQQBgjcZAaASBBDjAT1NPPfwT4sa
| sNjnBqS3gg9EQzAxLnNlcXVlbC5odGIwTQYJKwYBBAGCNxkCBEAwPqA8BgorBgEE
| AYI3GQIBoC4ELFMtMS01LTIxLTU0ODY3MDM5Ny05NzI2ODc0ODQtMzQ5NjMzNTM3
| MC0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCBDjlZZbFac6RlhZ2BhLzvWmA1Xcyn
| jZmYF3aOXmmof1yyO/kxk81fStsu3gtZ94KmpkBwmd1QkSJCuT54fTxg17xDtA49
| QF7O4DPsFkeOM2ip8TAf8x5bGwH5tlZvNjllBCgSpCupZlNY8wKYnyKQDNwtWtgL
| UF4SbE9Q6JWA+Re5lPa6AoUr/sRzKxcPsAjK8kgquUA0spoDrxAqkADIRsHgBLGY
| +Wn+DXHctZtv8GcOwrfW5KkbkVykx8DSS2qH4y2+xbC3ZHjsKlVjoddkjEkrHku0
| 2iXZSIqShMXzXmLTW/G+LzqK3U3VTcKo0yUKqmLlKyZXzQ+kYVLqgOOX
|_-----END CERTIFICATE-----
|_ssl-date: 2025-01-11T21:21:02+00:00; -1s from scanner time.
1433/tcp  open  ms-sql-s      syn-ack ttl 127 Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-info: 
|   10.10.11.51:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
|_ssl-date: 2025-01-11T21:21:03+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-01-11T19:02:26
| Not valid after:  2055-01-11T19:02:26
| MD5:   a839:345f:5b24:b589:a50b:6b29:9d36:494d
| SHA-1: b4ba:9346:9024:1dea:a1a4:f7fb:a894:ab00:4574:9bbc
| -----BEGIN CERTIFICATE-----
| MIIDADCCAeigAwIBAgIQG/jE+4JIv7hMg1GTNycudDANBgkqhkiG9w0BAQsFADA7
| MTkwNwYDVQQDHjAAUwBTAEwAXwBTAGUAbABmAF8AUwBpAGcAbgBlAGQAXwBGAGEA
| bABsAGIAYQBjAGswIBcNMjUwMTExMTkwMjI2WhgPMjA1NTAxMTExOTAyMjZaMDsx
| OTA3BgNVBAMeMABTAFMATABfAFMAZQBsAGYAXwBTAGkAZwBuAGUAZABfAEYAYQBs
| AGwAYgBhAGMAazCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJnIvlcE
| iuXvv1Smqusi+X6TNdp0spRuYVDWcdUJqVoqFmLm/4TfBn59vMcD/BwDLOT6qHIe
| 8JlrIjvlJ2Xx4YIHmLfu23Nx4vtRHuXbgeg9kUGcAMaI3Q2AAaxqf4LewPEoipz7
| g4pssZnfnoHizzWCky6g6vL+mMhH8I5BwOMzx2XPgZllrhRAnof0OeqUvAhPuWZJ
| GY1JRIuiohb6PaevWj7xUH1suKhKkwrpVvuPTxKk2bBlwIS6s8QkRzNRHsiU8fba
| 94a5FdODY91361stW+xXb2Z10SoKtXvfYNez7VLUI7kVMyW53MrW8mp82zvoKiuc
| bbPitZhKpRAluG0CAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAF7QmN3KzeQkIiI1p
| /OoLNcsM5X4pm2hj4PSnv1n0Dhv81vK0qGWi3WLlGGm7u6qyVFcDMW/ILkig5GXE
| xdUS2EQRvIZzn/hbWpAoovC/sqzALaRFp9OuU+xU51owYX4NL+kTa7twWEuSpvgy
| ua6v0eCZPpjJaYCWrD5aHFHSx5v0Q283QmSCODhF4vUr06G9C+GTfM1xD+xvgdcU
| iOU3Il8Ig4DKzy6E4AQiOM08tHxyR25zx2xWsKUGiiufEdp/ysA8QsAtz60g5DrI
| TV1CMgepyIN8Hx9wTrDpcH6nNRNREpOh/tHwuo/wNv7EiUUMBU49cnvpXboavPqY
| mu2Cww==
|_-----END CERTIFICATE-----
| ms-sql-ntlm-info: 
|   10.10.11.51:1433: 
|     Target_Name: SEQUEL
|     NetBIOS_Domain_Name: SEQUEL
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: DC01.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-01-11T21:21:03+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.sequel.htb
| Issuer: commonName=sequel-DC01-CA/domainComponent=sequel
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-06-08T17:35:00
| Not valid after:  2025-06-08T17:35:00
| MD5:   09fd:3df4:9f58:da05:410d:e89e:7442:b6ff
| SHA-1: c3ac:8bfd:6132:ed77:2975:7f5e:6990:1ced:528e:aac5
| -----BEGIN CERTIFICATE-----
| MIIGJjCCBQ6gAwIBAgITVAAAAANDveocXlnSDQAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBGMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGc2VxdWVs
| MRcwFQYDVQQDEw5zZXF1ZWwtREMwMS1DQTAeFw0yNDA2MDgxNzM1MDBaFw0yNTA2
| MDgxNzM1MDBaMBoxGDAWBgNVBAMTD0RDMDEuc2VxdWVsLmh0YjCCASIwDQYJKoZI
| hvcNAQEBBQADggEPADCCAQoCggEBANRCnm8pZ86LZP3kAtl29rFgY5gEOEXSCZSm
| F6Ai+1vh6a8LrCRKMWtC8+Kla0PXgjTcGcmDawcI8h0BsaSH6sQVAD21ca5MQcv0
| xf+4TzrvAnp9H+pVHO1r42cLXBwq14Ak8dSueiOLgxoLKO1CDtKk+e8ZxQWf94Bp
| Vu8rnpImFT6IeDgACeBfb0hLzK2JJRT9ezZiUVxoTfMKKuy4IPFWcshW/1bQfEK0
| ExOcQZVaoCJzRPBUVTp/XGHEW9d6abW8h1UR+64qVfGexsrUKBfxKRsHuHTxa4ts
| +qUVJRbJkzlSgyKGMjhNfT3BPVwwP8HvErWvbsWKKPRkvMaPhU0CAwEAAaOCAzcw
| ggMzMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABs
| AGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQD
| AgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQME
| AgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglg
| hkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFNfVXsrpSahW
| xfdL4wxFDgtUztvRMB8GA1UdIwQYMBaAFMZBubbkDkfWBlps8YrGlP0a+7jDMIHI
| BgNVHR8EgcAwgb0wgbqggbeggbSGgbFsZGFwOi8vL0NOPXNlcXVlbC1EQzAxLUNB
| LENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERDPWh0Yj9jZXJ0aWZp
| Y2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0
| aW9uUG9pbnQwgb8GCCsGAQUFBwEBBIGyMIGvMIGsBggrBgEFBQcwAoaBn2xkYXA6
| Ly8vQ049c2VxdWVsLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERD
| PWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlv
| bkF1dGhvcml0eTA7BgNVHREENDAyoB8GCSsGAQQBgjcZAaASBBDjAT1NPPfwT4sa
| sNjnBqS3gg9EQzAxLnNlcXVlbC5odGIwTQYJKwYBBAGCNxkCBEAwPqA8BgorBgEE
| AYI3GQIBoC4ELFMtMS01LTIxLTU0ODY3MDM5Ny05NzI2ODc0ODQtMzQ5NjMzNTM3
| MC0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCBDjlZZbFac6RlhZ2BhLzvWmA1Xcyn
| jZmYF3aOXmmof1yyO/kxk81fStsu3gtZ94KmpkBwmd1QkSJCuT54fTxg17xDtA49
| QF7O4DPsFkeOM2ip8TAf8x5bGwH5tlZvNjllBCgSpCupZlNY8wKYnyKQDNwtWtgL
| UF4SbE9Q6JWA+Re5lPa6AoUr/sRzKxcPsAjK8kgquUA0spoDrxAqkADIRsHgBLGY
| +Wn+DXHctZtv8GcOwrfW5KkbkVykx8DSS2qH4y2+xbC3ZHjsKlVjoddkjEkrHku0
| 2iXZSIqShMXzXmLTW/G+LzqK3U3VTcKo0yUKqmLlKyZXzQ+kYVLqgOOX
|_-----END CERTIFICATE-----
3269/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-01-11T21:21:02+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC01.sequel.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.sequel.htb
| Issuer: commonName=sequel-DC01-CA/domainComponent=sequel
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-06-08T17:35:00
| Not valid after:  2025-06-08T17:35:00
| MD5:   09fd:3df4:9f58:da05:410d:e89e:7442:b6ff
| SHA-1: c3ac:8bfd:6132:ed77:2975:7f5e:6990:1ced:528e:aac5
| -----BEGIN CERTIFICATE-----
| MIIGJjCCBQ6gAwIBAgITVAAAAANDveocXlnSDQAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBGMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGc2VxdWVs
| MRcwFQYDVQQDEw5zZXF1ZWwtREMwMS1DQTAeFw0yNDA2MDgxNzM1MDBaFw0yNTA2
| MDgxNzM1MDBaMBoxGDAWBgNVBAMTD0RDMDEuc2VxdWVsLmh0YjCCASIwDQYJKoZI
| hvcNAQEBBQADggEPADCCAQoCggEBANRCnm8pZ86LZP3kAtl29rFgY5gEOEXSCZSm
| F6Ai+1vh6a8LrCRKMWtC8+Kla0PXgjTcGcmDawcI8h0BsaSH6sQVAD21ca5MQcv0
| xf+4TzrvAnp9H+pVHO1r42cLXBwq14Ak8dSueiOLgxoLKO1CDtKk+e8ZxQWf94Bp
| Vu8rnpImFT6IeDgACeBfb0hLzK2JJRT9ezZiUVxoTfMKKuy4IPFWcshW/1bQfEK0
| ExOcQZVaoCJzRPBUVTp/XGHEW9d6abW8h1UR+64qVfGexsrUKBfxKRsHuHTxa4ts
| +qUVJRbJkzlSgyKGMjhNfT3BPVwwP8HvErWvbsWKKPRkvMaPhU0CAwEAAaOCAzcw
| ggMzMC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABs
| AGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQD
| AgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQME
| AgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglg
| hkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFNfVXsrpSahW
| xfdL4wxFDgtUztvRMB8GA1UdIwQYMBaAFMZBubbkDkfWBlps8YrGlP0a+7jDMIHI
| BgNVHR8EgcAwgb0wgbqggbeggbSGgbFsZGFwOi8vL0NOPXNlcXVlbC1EQzAxLUNB
| LENOPURDMDEsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERDPWh0Yj9jZXJ0aWZp
| Y2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0
| aW9uUG9pbnQwgb8GCCsGAQUFBwEBBIGyMIGvMIGsBggrBgEFBQcwAoaBn2xkYXA6
| Ly8vQ049c2VxdWVsLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNl
| cnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERD
| PWh0Yj9jQUNlcnRpZmljYXRlP2Jhc2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlv
| bkF1dGhvcml0eTA7BgNVHREENDAyoB8GCSsGAQQBgjcZAaASBBDjAT1NPPfwT4sa
| sNjnBqS3gg9EQzAxLnNlcXVlbC5odGIwTQYJKwYBBAGCNxkCBEAwPqA8BgorBgEE
| AYI3GQIBoC4ELFMtMS01LTIxLTU0ODY3MDM5Ny05NzI2ODc0ODQtMzQ5NjMzNTM3
| MC0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCBDjlZZbFac6RlhZ2BhLzvWmA1Xcyn
| jZmYF3aOXmmof1yyO/kxk81fStsu3gtZ94KmpkBwmd1QkSJCuT54fTxg17xDtA49
| QF7O4DPsFkeOM2ip8TAf8x5bGwH5tlZvNjllBCgSpCupZlNY8wKYnyKQDNwtWtgL
| UF4SbE9Q6JWA+Re5lPa6AoUr/sRzKxcPsAjK8kgquUA0spoDrxAqkADIRsHgBLGY
| +Wn+DXHctZtv8GcOwrfW5KkbkVykx8DSS2qH4y2+xbC3ZHjsKlVjoddkjEkrHku0
| 2iXZSIqShMXzXmLTW/G+LzqK3U3VTcKo0yUKqmLlKyZXzQ+kYVLqgOOX
|_-----END CERTIFICATE-----
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49685/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49686/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49687/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49692/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49716/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49725/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49793/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60313/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-01-11T21:20:26
|_  start_date: N/A
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 62012/tcp): CLEAN (Timeout)
|   Check 2 (port 24344/tcp): CLEAN (Timeout)
|   Check 3 (port 20179/udp): CLEAN (Timeout)
|   Check 4 (port 45881/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: mean: 0s, deviation: 0s, median: 0s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/642" target="_blank">Escapetwo Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }
