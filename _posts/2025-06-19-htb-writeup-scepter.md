---
title: Scepter &#x1F512;
description: La maquina Scepter esta activa. Este artículo se publicará para acceso público una vez que la maquina se retire, según la política de HackTheBox.
date: 2025-06-19
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-scepter/scepter_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - windows
  - hack_the_box

---
## Information Gathering

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ ping -c 1 10.10.11.65
PING 10.10.11.65 (10.10.11.65) 56(84) bytes of data.
64 bytes from 10.10.11.65: icmp_seq=1 ttl=127 time=1048 ms

--- 10.10.11.65 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1047.919/1047.919/1047.919/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ sudo nmap -p- --open -sS -T5 -vvv 10.10.11.65 -n -Pn -oG nmap1
Host: 10.10.11.65 ()    Status: Up
Host: 10.10.11.65 ()    Ports: 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 111/open/tcp//rpcbind///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 2049/open/tcp//nfs///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 9389/open/tcp//adws///, 47001/open/tcp//winrm///, 49664/open/tcp/////, 49665/open/tcp/////, 49666/open/tcp/////, 49669/open/tcp/////, 49673/open/tcp/////, 49688/open/tcp/////, 49689/open/tcp/////, 49690/open/tcp/////, 49691/open/tcp/////, 49705/open/tcp/////, 49716/open/tcp/////, 49719/open/tcp/////
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ sudo nmap -sCV -vvv -p53,88,111,135,139,445,464,593,636,2049,3268,3269,9389,47001,49664,49665,49666,49669,49673,49688,49689,49690,49691,49705,49716,49719 10.10.11.65 -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-06-09 23:23:25Z)
111/tcp   open  rpcbind       syn-ack ttl 127 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: scepter.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.scepter.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.scepter.htb
| Issuer: commonName=scepter-DC01-CA/domainComponent=scepter
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-11-01T03:22:33
| Not valid after:  2025-11-01T03:22:33
| MD5:   2af6:88f7:a6bf:ef50:9b84:3dc6:3df5:e018
| SHA-1: cd9a:97ee:25c8:00ba:1427:c259:02ed:6e0d:9a21:7fd9
| -----BEGIN CERTIFICATE-----
| MIIGLDCCBRSgAwIBAgITYgAAACHTgl9VBArXxgAAAAAAITANBgkqhkiG9w0BAQsF
| ADBIMRMwEQYKCZImiZPyLGQBGRYDaHRiMRcwFQYKCZImiZPyLGQBGRYHc2NlcHRl
| cjEYMBYGA1UEAxMPc2NlcHRlci1EQzAxLUNBMB4XDTI0MTEwMTAzMjIzM1oXDTI1
| MTEwMTAzMjIzM1owGzEZMBcGA1UEAxMQZGMwMS5zY2VwdGVyLmh0YjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBALpnNbJF0dXLfbmd6n3LpJlQDKdwZdVT
| JxqBS7Vz/LPj+ZpUA6JFTi31Jdy8qFqRF3HuBhsA5T+RPLGuhjoNAqMKqlWEcqOC
| A4VHl99hLPKB0mpqSTVKIXzvvU2Aa2Pc42gGY4nmpODO06an3XddKCMdQx2dPXK+
| /GUmsYPEszgoefAJLOaJ/ot23i1ffdcYE8c7xbi/ivUmLmOo6zQp/6FCRsJM4Ago
| OZ0mV9tLt7jfltrNBL+Iq8FWoiV59ciaOmNLNwIo+JqkPjTYJNSuSsiaeVNUtoY1
| yipUhhDOyX70wc48R20/So6PUOKnkGJ6ovrEQJCEpVBkic/eLlHaWbUCAwEAAaOC
| AzowggM2MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8A
| bABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/
| BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3
| DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjAL
| BglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFM+Zo2Ay
| sKIDhRmsELT8JvcQ5qJEMB8GA1UdIwQYMBaAFOuQVDjSpmyJasttTaS6dRVgFSfj
| MIHKBgNVHR8EgcIwgb8wgbyggbmggbaGgbNsZGFwOi8vL0NOPXNjZXB0ZXItREMw
| MS1DQSxDTj1kYzAxLENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxD
| Tj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPXNjZXB0ZXIsREM9aHRiP2Nl
| cnRpZmljYXRlUmV2b2NhdGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0
| cmlidXRpb25Qb2ludDCBwQYIKwYBBQUHAQEEgbQwgbEwga4GCCsGAQUFBzAChoGh
| bGRhcDovLy9DTj1zY2VwdGVyLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtl
| eSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2Nl
| cHRlcixEQz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRp
| ZmljYXRpb25BdXRob3JpdHkwPAYDVR0RBDUwM6AfBgkrBgEEAYI3GQGgEgQQuQyF
| jYzg20GS235CRJngkoIQZGMwMS5zY2VwdGVyLmh0YjBLBgkrBgEEAYI3GQIEPjA8
| oDoGCisGAQQBgjcZAgGgLAQqUy0xLTUtMjEtNzQ4Nzk1NDYtOTE2ODE4NDM0LTc0
| MDI5NTM2NS0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCKy5wPeTrqhyCr9gEglZ8K
| EKsHXZsfcQu35qHlaxyWxISCZ4CCDaD+MlTT6fnvw3oyF4Nd8ArI/QQwnqqPxxYk
| 72HoVo835fo0lP3FeDfnbYT6rUMrv4QVkeJossDwnOnrZuGPtfUEWxNg1O76D2kU
| gejyZzFgBcvaXAt/pEHVki2Zfdz7p1OAkbjP2cAsjFAAzdAZT1FpRdcL+s1PwZqd
| urydtAwyuvSqyzDYJgt4aj0kdyNoFexNK2meqw5DdYWnrDTcBLdN4v37kKtMm2w1
| 9X2shB2kglATgm0ULSz7jHVZNnACrxBBUsofMPVCvpsEBmfCb4zPo6a+oA0MjGsS
|_-----END CERTIFICATE-----
|_ssl-date: 2025-06-09T23:24:41+00:00; +7h59m59s from scanner time.
2049/tcp  open  nlockmgr      syn-ack ttl 127 1-4 (RPC #100021)
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: scepter.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.scepter.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.scepter.htb
| Issuer: commonName=scepter-DC01-CA/domainComponent=scepter
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-11-01T03:22:33
| Not valid after:  2025-11-01T03:22:33
| MD5:   2af6:88f7:a6bf:ef50:9b84:3dc6:3df5:e018
| SHA-1: cd9a:97ee:25c8:00ba:1427:c259:02ed:6e0d:9a21:7fd9
| -----BEGIN CERTIFICATE-----
| MIIGLDCCBRSgAwIBAgITYgAAACHTgl9VBArXxgAAAAAAITANBgkqhkiG9w0BAQsF
| ADBIMRMwEQYKCZImiZPyLGQBGRYDaHRiMRcwFQYKCZImiZPyLGQBGRYHc2NlcHRl
| cjEYMBYGA1UEAxMPc2NlcHRlci1EQzAxLUNBMB4XDTI0MTEwMTAzMjIzM1oXDTI1
| MTEwMTAzMjIzM1owGzEZMBcGA1UEAxMQZGMwMS5zY2VwdGVyLmh0YjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBALpnNbJF0dXLfbmd6n3LpJlQDKdwZdVT
| JxqBS7Vz/LPj+ZpUA6JFTi31Jdy8qFqRF3HuBhsA5T+RPLGuhjoNAqMKqlWEcqOC
| A4VHl99hLPKB0mpqSTVKIXzvvU2Aa2Pc42gGY4nmpODO06an3XddKCMdQx2dPXK+
| /GUmsYPEszgoefAJLOaJ/ot23i1ffdcYE8c7xbi/ivUmLmOo6zQp/6FCRsJM4Ago
| OZ0mV9tLt7jfltrNBL+Iq8FWoiV59ciaOmNLNwIo+JqkPjTYJNSuSsiaeVNUtoY1
| yipUhhDOyX70wc48R20/So6PUOKnkGJ6ovrEQJCEpVBkic/eLlHaWbUCAwEAAaOC
| AzowggM2MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8A
| bABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/
| BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3
| DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjAL
| BglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFM+Zo2Ay
| sKIDhRmsELT8JvcQ5qJEMB8GA1UdIwQYMBaAFOuQVDjSpmyJasttTaS6dRVgFSfj
| MIHKBgNVHR8EgcIwgb8wgbyggbmggbaGgbNsZGFwOi8vL0NOPXNjZXB0ZXItREMw
| MS1DQSxDTj1kYzAxLENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxD
| Tj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPXNjZXB0ZXIsREM9aHRiP2Nl
| cnRpZmljYXRlUmV2b2NhdGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0
| cmlidXRpb25Qb2ludDCBwQYIKwYBBQUHAQEEgbQwgbEwga4GCCsGAQUFBzAChoGh
| bGRhcDovLy9DTj1zY2VwdGVyLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtl
| eSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2Nl
| cHRlcixEQz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRp
| ZmljYXRpb25BdXRob3JpdHkwPAYDVR0RBDUwM6AfBgkrBgEEAYI3GQGgEgQQuQyF
| jYzg20GS235CRJngkoIQZGMwMS5zY2VwdGVyLmh0YjBLBgkrBgEEAYI3GQIEPjA8
| oDoGCisGAQQBgjcZAgGgLAQqUy0xLTUtMjEtNzQ4Nzk1NDYtOTE2ODE4NDM0LTc0
| MDI5NTM2NS0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCKy5wPeTrqhyCr9gEglZ8K
| EKsHXZsfcQu35qHlaxyWxISCZ4CCDaD+MlTT6fnvw3oyF4Nd8ArI/QQwnqqPxxYk
| 72HoVo835fo0lP3FeDfnbYT6rUMrv4QVkeJossDwnOnrZuGPtfUEWxNg1O76D2kU
| gejyZzFgBcvaXAt/pEHVki2Zfdz7p1OAkbjP2cAsjFAAzdAZT1FpRdcL+s1PwZqd
| urydtAwyuvSqyzDYJgt4aj0kdyNoFexNK2meqw5DdYWnrDTcBLdN4v37kKtMm2w1
| 9X2shB2kglATgm0ULSz7jHVZNnACrxBBUsofMPVCvpsEBmfCb4zPo6a+oA0MjGsS
|_-----END CERTIFICATE-----
|_ssl-date: 2025-06-09T23:24:42+00:00; +7h59m59s from scanner time.
3269/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: scepter.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.scepter.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.scepter.htb
| Issuer: commonName=scepter-DC01-CA/domainComponent=scepter
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-11-01T03:22:33
| Not valid after:  2025-11-01T03:22:33
| MD5:   2af6:88f7:a6bf:ef50:9b84:3dc6:3df5:e018
| SHA-1: cd9a:97ee:25c8:00ba:1427:c259:02ed:6e0d:9a21:7fd9
| -----BEGIN CERTIFICATE-----
| MIIGLDCCBRSgAwIBAgITYgAAACHTgl9VBArXxgAAAAAAITANBgkqhkiG9w0BAQsF
| ADBIMRMwEQYKCZImiZPyLGQBGRYDaHRiMRcwFQYKCZImiZPyLGQBGRYHc2NlcHRl
| cjEYMBYGA1UEAxMPc2NlcHRlci1EQzAxLUNBMB4XDTI0MTEwMTAzMjIzM1oXDTI1
| MTEwMTAzMjIzM1owGzEZMBcGA1UEAxMQZGMwMS5zY2VwdGVyLmh0YjCCASIwDQYJ
| KoZIhvcNAQEBBQADggEPADCCAQoCggEBALpnNbJF0dXLfbmd6n3LpJlQDKdwZdVT
| JxqBS7Vz/LPj+ZpUA6JFTi31Jdy8qFqRF3HuBhsA5T+RPLGuhjoNAqMKqlWEcqOC
| A4VHl99hLPKB0mpqSTVKIXzvvU2Aa2Pc42gGY4nmpODO06an3XddKCMdQx2dPXK+
| /GUmsYPEszgoefAJLOaJ/ot23i1ffdcYE8c7xbi/ivUmLmOo6zQp/6FCRsJM4Ago
| OZ0mV9tLt7jfltrNBL+Iq8FWoiV59ciaOmNLNwIo+JqkPjTYJNSuSsiaeVNUtoY1
| yipUhhDOyX70wc48R20/So6PUOKnkGJ6ovrEQJCEpVBkic/eLlHaWbUCAwEAAaOC
| AzowggM2MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8A
| bABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/
| BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3
| DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjAL
| BglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFM+Zo2Ay
| sKIDhRmsELT8JvcQ5qJEMB8GA1UdIwQYMBaAFOuQVDjSpmyJasttTaS6dRVgFSfj
| MIHKBgNVHR8EgcIwgb8wgbyggbmggbaGgbNsZGFwOi8vL0NOPXNjZXB0ZXItREMw
| MS1DQSxDTj1kYzAxLENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxD
| Tj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPXNjZXB0ZXIsREM9aHRiP2Nl
| cnRpZmljYXRlUmV2b2NhdGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0
| cmlidXRpb25Qb2ludDCBwQYIKwYBBQUHAQEEgbQwgbEwga4GCCsGAQUFBzAChoGh
| bGRhcDovLy9DTj1zY2VwdGVyLURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtl
| eSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9c2Nl
| cHRlcixEQz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRp
| ZmljYXRpb25BdXRob3JpdHkwPAYDVR0RBDUwM6AfBgkrBgEEAYI3GQGgEgQQuQyF
| jYzg20GS235CRJngkoIQZGMwMS5zY2VwdGVyLmh0YjBLBgkrBgEEAYI3GQIEPjA8
| oDoGCisGAQQBgjcZAgGgLAQqUy0xLTUtMjEtNzQ4Nzk1NDYtOTE2ODE4NDM0LTc0
| MDI5NTM2NS0xMDAwMA0GCSqGSIb3DQEBCwUAA4IBAQCKy5wPeTrqhyCr9gEglZ8K
| EKsHXZsfcQu35qHlaxyWxISCZ4CCDaD+MlTT6fnvw3oyF4Nd8ArI/QQwnqqPxxYk
| 72HoVo835fo0lP3FeDfnbYT6rUMrv4QVkeJossDwnOnrZuGPtfUEWxNg1O76D2kU
| gejyZzFgBcvaXAt/pEHVki2Zfdz7p1OAkbjP2cAsjFAAzdAZT1FpRdcL+s1PwZqd
| urydtAwyuvSqyzDYJgt4aj0kdyNoFexNK2meqw5DdYWnrDTcBLdN4v37kKtMm2w1
| 9X2shB2kglATgm0ULSz7jHVZNnACrxBBUsofMPVCvpsEBmfCb4zPo6a+oA0MjGsS
|_-----END CERTIFICATE-----
|_ssl-date: 2025-06-09T23:24:40+00:00; +7h59m58s from scanner time.
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49669/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49673/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49688/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49689/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49690/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49691/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49705/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49716/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49719/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-06-09T23:24:33
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h59m58s, deviation: 0s, median: 7h59m58s
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 18337/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 36463/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 13248/udp): CLEAN (Failed to receive data)
|   Check 4 (port 54214/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/657" target="_blank">Scepter Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

> Una máquina puede estar activa o retirada. Retirada, significa que la máquina no cuenta para los puntos de temporada.
{: .prompt-tip }

