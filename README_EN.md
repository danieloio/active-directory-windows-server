# 04 — Active Directory with Windows Server 2022

![Windows Server](https://img.shields.io/badge/Windows_Server-2022-blue?logo=windows)
![Active Directory](https://img.shields.io/badge/Active_Directory-empresa.local-0078D4)
![VMware](https://img.shields.io/badge/VMware-Workstation_17-607078?logo=vmware)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

Deployment of an Active Directory domain on Windows Server 2022 in VMware Workstation. The project includes promoting the server to a domain controller, creating Organizational Units (OUs) per site, users and security groups per department, Group Policy Objects (GPOs) applied per OU, and a Windows 11 Pro client joined to the domain.

---

## Table of Contents

- [Infrastructure](#infrastructure)
- [Domain](#domain)
- [Configuration](#configuration)
- [Users and groups](#users-and-groups)
- [Group Policy Objects (GPOs)](#group-policy-objects-gpos)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

---

## Infrastructure

| Parameter | SRV-DC-01 | PC-CLIENT-01 |
|---|---|---|
| OS | Windows Server 2022 Standard Evaluation | Windows 11 Pro x64 |
| RAM | 4 GB | 4 GB |
| Disk | 60 GB NVMe | 60 GB NVMe |
| CPU | 2 cores | 2 cores |
| Network | LAN Segment 1 + NAT | LAN Segment 1 |
| IP | 192.168.100.1 | 192.168.100.10 |
| Hypervisor | VMware Workstation 17 Player | VMware Workstation 17 Player |

### Network configuration — SRV-DC-01

![SRV VMware Network](screenshots/03-srv-vmware-network.png)
![SRV VMware Options](screenshots/04-srv-vmware-options.png)

### Network configuration — PC-CLIENT-01

![CLI VMware Network](screenshots/01-cli-vmware-network.png)
![CLI VMware Options](screenshots/02-cli-vmware-options.png)

---

## Domain

| Parameter | Value |
|---|---|
| Domain name | empresa.local |
| Domain controller | SRV-DC-01.empresa.local |
| Forest functional level | Windows Server 2016 |
| Domain functional level | Windows Server 2016 |
| Integrated DNS | Yes — 192.168.100.1 |

---

## Configuration

### 1. Static IP on SRV-DC-01

```
IP:      192.168.100.1
Mask:    255.255.255.0
Gateway: (empty)
DNS:     127.0.0.1
```

### 2. AD DS installation and domain controller promotion

```powershell
# Install the role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to domain controller
Install-ADDSForest `
  -DomainName "empresa.local" `
  -DomainNetbiosName "EMPRESA" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -Force:$true
```

### 3. Static IP on PC-CLIENT-01

```
IP:      192.168.100.10
Mask:    255.255.255.0
Gateway: 192.168.100.1
DNS:     192.168.100.1
```

> DNS must point to the server because Active Directory relies on integrated DNS to resolve empresa.local.

### 4. Joining PC-CLIENT-01 to the domain

```
Start → System → Advanced system settings → Computer Name → Change
Computer name: PC-CLIENT-01
Member of Domain: empresa.local
Credentials: Administrator / (server password)
```

---

## Users and groups

### OU structure

![AD OU Structure](screenshots/05-ad-ou-structure.png)

```
empresa.local
├── OU-MADRID
│   ├── OU-Equipos
│   ├── OU-Usuarios
│   │   ├── Jose Garcia Lopez     (josegalo)
│   │   └── Laura Martinez Ruiz   (lamaru)
│   ├── GRP-BaseDatos
│   ├── GRP-Redes
│   ├── GRP-Sistemas
│   └── GRP-Soporte
├── OU-BCN
│   ├── OU-Equipo
│   └── OU-Usuarios
│       ├── Alberto Gonzalez Gil  (agongil)
│       └── Ana Lopez Fernandez   (alofer)
└── OU-VLC
    ├── OU-Equipos
    └── OU-Usuarios
        ├── Miguel Torres Alba    (mitoral)
        └── Sofia Ramos Vega      (soramve)
```

### Users in OU-MADRID

![OU Users Madrid](screenshots/06-ou-users-madrid.png)

### Security groups

![Security Groups](screenshots/07-security-groups.png)

| Group | Type | Scope | Members |
|---|---|---|---|
| GRP-Sistemas | Security | Global | jose.garcia |
| GRP-Redes | Security | Global | laura.martinez, miguel.torres |
| GRP-Soporte | Security | Global | alberto.gonzalez, sofia.ramos |
| GRP-BaseDatos | Security | Global | ana.lopez |

---

## Group Policy Objects (GPOs)

![GPO Management](screenshots/08-gpo-management.png)

| GPO | Applied to | Type | Effect |
|---|---|---|---|
| GPO-Fondo-Corporativo | empresa.local | Enforced | Corporate desktop wallpaper |
| GPO-Politica-Contrasenas | empresa.local | Enforced | Min 8 chars, complexity enabled |
| GPO-Restriccion-Panel | OU-Usuarios (3 sites) | Linked | Disables Control Panel |
| GPO-Bloqueo-USB | OU-Usuarios (3 sites) | Linked | Blocks removable storage devices |

### GPO — Password policy

```
Computer Configuration → Policies → Windows Settings
→ Security Settings → Account Policies → Password Policy

Minimum password length:              8
Password must meet complexity:        Enabled
Maximum password age:                 90 days
Minimum password age:                 1 day
Enforce password history:             5 passwords
```

### GPO — Control Panel restriction

```
User Configuration → Policies → Administrative Templates
→ Control Panel
→ Prohibit access to Control Panel and PC Settings: Enabled
```

![GPO Control Panel Blocked](screenshots/09-gpo-control-panel-blocked.png)

### GPO — Corporate wallpaper

```
User Configuration → Preferences → Windows Settings → Registry

HKCU\Control Panel\Desktop
  Wallpaper      = C:\Windows\Web\Wallpaper\Corporativo\fondo.jpg
  WallpaperStyle = 10 (Fill)
```

![GPO Wallpaper Applied](screenshots/10-gpo-wallpaper-applied.png)

### GPO — USB block

```
Computer Configuration → Policies → Administrative Templates
→ System → Removable Storage Access
→ All Removable Storage classes: Deny all access: Enabled
```

---

## Verification

### gpresult /r on PC-CLIENT-01

![gpresult 1](screenshots/12-gpresult-1.png)
![gpresult 2](screenshots/13-gpresult-2.png)

### Domain users — Get-ADUser

![Get-ADUser 1](screenshots/14-get-aduser-1.png)
![Get-ADUser 2](screenshots/15-get-aduser-2.png)

### Domain computers — Get-ADComputer

![Get-ADComputer](screenshots/16-get-adcomputer.png)

### Expected results

| Test | Expected output |
|---|---|
| `Get-ADUser -Filter *` | 6 domain users + system accounts |
| `Get-ADComputer -Filter *` | SRV-DC-01 and PC-CLIENT-01 |
| `gpresult /r` | GPO-Fondo-Corporativo and GPO-Restriccion-Panel applied |
| Control Panel | Blocked with restriction message |
| Desktop wallpaper | Corporate image applied via GPO |
| `whoami` on client | EMPRESA\alofer |

---

## Troubleshooting

**Issue:** Desktop wallpaper appeared black on the client even though the GPO was applied.
**Root cause:** Windows 11 blocks wallpapers served from UNC network paths when applied via Administrative Templates for security reasons.
**Fix:** Used User Configuration → Preferences → Windows Settings → Registry to write directly to `HKCU\Control Panel\Desktop\Wallpaper` with a local path previously copied to the client via GPO Files.

---

**Issue:** Wallpaper path was incorrectly specified in the registry.
**Root cause:** Typo in the path when configuring the registry key in the GPO.
**Fix:** Verified the path with `regedit` on the client under `HKEY_CURRENT_USER\Control Panel\Desktop\Wallpaper` and corrected it in the GPO.

---

*Lab built with VMware Workstation 17 Player — Daniel Moisés Loyo Vásquez*
