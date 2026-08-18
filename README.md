# Windows Server 2025 Active Directory Homelab

## Project Overview

This project documents the design and deployment of a virtualized Windows enterprise environment for **MakahaV Solutions**, a fictional small business.

The lab was built using **Hyper-V, Windows Server 2025, and Windows 11 Pro** to gain hands-on experience with Active Directory administration, DNS, Group Policy, identity and access management, Windows networking, file sharing, and troubleshooting.

The environment includes a Windows Server domain controller and a domain-joined Windows 11 workstation. Users are organized by department and granted access to resources through security groups rather than individual permissions.

## Lab Environment

| Component | Configuration |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Domain Controller | Windows Server 2025 |
| Client | Windows 11 Pro |
| Domain | `ad.makahav.test` |
| Domain Controller | `DC01` |
| Client Workstation | `CLIENT01` |
| DC01 IPv4 | `10.10.10.10` |
| CLIENT01 IPv4 | `10.10.10.20` |
| DNS Server | `10.10.10.10` |
| Lab Network | `10.10.10.0/24` |

### Logical Architecture

```text
                    ad.makahav.test
                           |
                         DC01
                     10.10.10.10
                           |
              Active Directory + DNS
                           |
                     MakahaV-Lab
                           |
                        CLIENT01
                     10.10.10.20
                           |
                  Windows 11 Pro
```

## Active Directory Design

A custom organizational structure was created for MakahaV Solutions:

```text
ad.makahav.test
|
└── MakahaV
    ├── Computers
    │   └── CLIENT01
    ├── Groups
    │   ├── GG_IT
    │   ├── GG_Accounting
    │   └── GG_Sales
    └── Users
        ├── IT
        ├── Accounting
        └── Sales
```

Eleven fictional employee accounts were provisioned across IT, Accounting, and Sales.

Global security groups were used to manage departmental access:

- `GG_IT`
- `GG_Accounting`
- `GG_Sales`

This separates organizational structure from authorization and avoids assigning resource permissions directly to individual users.

## Domain Integration

`CLIENT01` was configured with a static IPv4 address and pointed to `DC01` for DNS.

DNS resolution and Active Directory service discovery were verified before joining the workstation to:

`ad.makahav.test`

After the domain join, authentication was successfully tested using a domain employee account.

```powershell
whoami
hostname
```

Example verified result:

```text
makahav\arivera
CLIENT01
```

## Group Policy

A custom workstation security GPO was created:

`MakahaV - Workstation Security Baseline`

The GPO was linked to the MakahaV Computers OU rather than modifying the Default Domain Policy.

A **600-second machine inactivity limit** was configured to automatically lock inactive workstations.

Policy deployment was verified on CLIENT01 using:

```powershell
gpupdate /force
gpresult /r /scope computer
```

The resulting configuration was independently verified through the registry:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v InactivityTimeoutSecs
```

Result:

```text
REG_DWORD    0x258
```

`0x258` corresponds to **600 seconds / 10 minutes**.

Successful and failed logon auditing was also enabled through Advanced Audit Policy and verified with:

```powershell
auditpol /get /subcategory:"Logon"
```

## File Sharing and Least Privilege

Three departmental SMB shares were created on DC01:

| Share | Security Group | Share Permission | NTFS Permission |
|---|---|---|---|
| `\\DC01\Accounting` | `GG_Accounting` | Change + Read | Modify |
| `\\DC01\IT` | `GG_IT` | Change + Read | Modify |
| `\\DC01\Sales` | `GG_Sales` | Change + Read | Modify |

Generic user access was removed from the departmental NTFS ACLs while administrative and SYSTEM access was retained.

This created a group-based access model:

```text
User
  ↓
Department Security Group
  ↓
SMB Share Permission
  ↓
NTFS Permission
  ↓
Department Resource
```

### Access-Control Testing

Both negative and positive tests were performed.

An IT user attempted to access:

`\\DC01\Accounting`

Access was denied as expected.

An Accounting user then accessed the same share and successfully opened and modified a test document.

This verified that permissions were enforcing **least privilege**, rather than merely existing as configured ACL entries.

## Automated Department Drive Mapping

A second GPO was created:

`MakahaV - Department Drive Maps`

Group Policy Preferences and **item-level targeting** were used to automatically provision departmental network drives based on security-group membership:

| Group | Drive | Network Path |
|---|---|---|
| `GG_Accounting` | `A:` | `\\DC01\Accounting` |
| `GG_IT` | `I:` | `\\DC01\IT` |
| `GG_Sales` | `S:` | `\\DC01\Sales` |

Testing confirmed that an Accounting user received the Accounting drive while an IT user received the IT drive without manually mapping either resource on the workstation.

## Validation and Troubleshooting

The environment was validated using Windows administrative and diagnostic tools including:

```powershell
Resolve-DnsName
Get-Service
Get-ADDomainController
Get-SmbShare
gpupdate
gpresult
auditpol
nltest
dcdiag
whoami
hostname
```

Troubleshooting performed during the project included:

- Correcting DNS configuration for Active Directory service discovery
- Recovering the environment after an unexpected host power outage
- Verifying AD DS and DNS health after recovery
- Resolving Windows 11 OOBE networking using a temporary Hyper-V Default Switch
- Correcting NTFS inheritance before modifying departmental ACLs
- Detecting and removing an unintended parent SMB share
- Resetting a domain user's temporary password through Active Directory Users and Computers

## Skills Demonstrated

- Windows Server 2025 administration
- Active Directory Domain Services
- DNS configuration and troubleshooting
- Hyper-V virtualization
- Windows 11 domain integration
- Organizational Unit design
- User and computer account administration
- Security group management
- Group Policy administration
- Group Policy Preferences
- Windows security auditing
- SMB file sharing
- NTFS permissions
- Least-privilege access control
- Windows networking
- PowerShell administration and validation
- Technical troubleshooting

## Project Screenshots

Screenshots documenting the build and validation process are available in the [`01-screenshots`](./01-screenshots/) directory.

Highlights include:

- Active Directory OU structure
- Domain user provisioning
- Security group membership
- Windows 11 domain authentication
- Group Policy deployment and verification
- Unauthorized Accounting-share access denial
- Authorized Accounting-share read/write access
- SMB share verification
- Group-targeted departmental drive mapping

## Key Takeaways

This project reinforced that Active Directory depends heavily on correctly functioning **DNS, identity, networking, and policy infrastructure**.

It also demonstrated the value of managing authorization through **security groups rather than individual user permissions**. Combining Active Directory groups, NTFS permissions, SMB permissions, and Group Policy provided a scalable way to centrally manage users, workstations, security settings, and departmental resources.

Most importantly, each major configuration was **tested and verified from both the administrator and end-user perspectives** rather than assumed to be working based solely on configuration.
