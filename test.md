# 📘 Windows Server 2022 – Day 1 Practical Runbook

## 🧪 Active Directory + DNS + DHCP (On-Prem + Azure Entra ID Mapping)

**Domain (On-Prem):** technova.local

**Domain Controller:** DC01

**IP Address:** 192.168.100.10

**OS:** Windows Server 2022

**Client:** Windows 11 (TNS01)

---

# 🧭 Table of Contents

- Lab Overview
- Architecture (On-Prem + Azure Mapping)
- Server Setup (DC01)
- Network Configuration
- DNS Setup & Commands Explained
- Active Directory Installation
- Domain Promotion
- Service Accounts
- Password Policy (GPO)
- Users & Groups
- Domain Join Process
- PowerShell Validation
- Azure Entra ID Equivalent (Step-by-Step)
- MCSA Questions (With Answers)
- Interview Questions (With Answers)
- Troubleshooting Guide

---

# 🏗️ 1. Lab Overview

This lab builds a complete enterprise identity system:

### On-Prem Services

- Active Directory Domain Services (AD DS)
- DNS Server
- DHCP Server
- Group Policy (GPO)
- User & Group Management

### Cloud Equivalent

- Microsoft Entra ID (Azure AD)
- Microsoft Intune
- Azure DNS + VNet
- Conditional Access Policies

---

# 🧠 2. Architecture Overview

## 🖥️ On-Prem

```
DC01

├── AD DS (Identity)

├── DNS (Name Resolution)

├── DHCP (IP Assignment)

└── GPO (Policies)

Client (Windows 11)

└── Domain Joined → technova.local
```

## ☁️ Azure Equivalent

```
Microsoft Entra ID

├── Users & Groups

├── Devices (Azure Joined)

├── Intune Policies

└── Conditional Access
```

---

# 🖥️ 3. Server Setup (DC01)

## VM Creation

```text
VMware / Hyper-V → New VM → Windows Server 2022
```

### Command Explanation

- This is not a command but a virtualization setup step
- Creates a virtual machine to install Windows Server

---

## Configuration

SettingValueNameDC01RAM4 GBCPU2 vCPUDisk80 GB

---

## ☁️ Azure Equivalent

- Azure Virtual Machine instead of VMware
- Marketplace image used instead of ISO install

---

# 🌐 4. Network Configuration

## Command

```
ncpa.cpl
```

### Explanation

- Opens Network Connections panel
- Used to configure IP settings manually

---

## IPv4 Settings

FieldValueIP192.168.100.10Mask255.255.255.0Gateway192.168.100.2DNS192.168.100.10

---

## ☁️ Azure Equivalent

- IP assigned via Azure NIC
- DNS configured via VNet settings

---

# 🧪 5. DNS Validation

## Command 1

```
Get-Service DNS
```

### Explanation

- Checks if DNS service is running
- Required for Active Directory functionality

### Expected Output

- Status: Running

---

## Command 2

```
nslookup dc01.technova.local
```

### Explanation

- Queries DNS for internal domain name
- Verifies AD DNS records

---

## Command 3

```
nslookup google.com
```

### Explanation

- Tests external DNS resolution
- Ensures forwarders are working

---

# 🌍 6. DNS Forwarders

## Navigation

```
Server Manager → Tools → DNS → DC01 → Properties → Forwarders
```

## Added DNS Servers

```
8.8.8.8
1.1.1.1
```

---

## Why this is needed

- Internal DNS cannot resolve internet names by itself
- Forwarders send external queries to public DNS

---

## ☁️ Azure Equivalent

- Azure DNS Private Zones
- Custom DNS in VNet settings

---

# 🏗️ 7. Active Directory Installation

## Navigation

```
Server Manager → Add Roles → AD DS → Install
```

### Explanation

- Installs Active Directory role
- Enables domain creation and authentication

---

## ☁️ Azure Equivalent

- No installation required
- Identity is managed by Microsoft Entra ID

---

# 🏛️ 8. Domain Controller Promotion

## Navigation

```
Promote this server to a Domain Controller
```

## Configuration

```
New Forest: technova.local
NetBIOS: TECHNOVA
```

### Explanation

- Creates a new Active Directory forest
- Establishes domain structure

---

## ☁️ Azure Equivalent

- Create Entra ID tenant instead
- No forest/domain concept in cloud

---

# 👤 9. Service Accounts

## Created Account

```
domainjoin / domainjoin
```

## Command Explanation (Creation Path)

```
AD Users and Computers → Users → New User
```

### Explanation

- Creates a non-human account
- Used for automation (domain join scripts)
- Avoids using Administrator account

---

## ☁️ Azure Equivalent

- Managed Identity
- Service Principal (App Registration)

---

# 🔐 10. Password Policy Configuration (Fine-Grained Password Policy – ADAC Method)

## 🎯 What was done in this step?

A **Fine-Grained Password Policy (FGPP)** was created using the **Active Directory Administrative Center (ADAC)** and applied to a specific service account (`domainjoin`).

This allows different password rules for specific users instead of applying one global policy.

---

# 🧭 Path (Correct Navigation)

```text
Server Manager
→ Tools
→ Active Directory Administrative Center
→ technova.local
→ System
→ Password Settings Container
→ New → Password Settings

## Applied Settings

SettingValueMin Length1 (Lab)ComplexityDisabled

---

# 👤 11. User Creation

## Users Created

NameDepartment
Ahmed HassanIT
Fatima AliFinance
Omar KhanSales
Noor IbrahimMarketing
Zain AbdullahOperations

---

## Command Path

```
ADUC → OU → New User
```

### Explanation

- Creates domain user accounts
- Stored in Active Directory database

---

## ☁️ Azure Equivalent

```
Entra Admin Center → Users → New User
```

---

# 👥 12. Security Groups

```
GG_IT_Users
GG_Finance_Users
GG_Sales_Users
GG_Marketing_Users
GG_Operations_Users
```

### Explanation

- Groups simplify permission management
- Used in Role-Based Access Control (RBAC)

---

## ☁️ Azure Equivalent

- Entra ID Security Groups
- Microsoft 365 Groups

---

# 💻 13. Domain Join Process

## Command

```
sysdm.cpl
```

### Explanation

- Opens System Properties
- Used to rename PC or join domain

---

## Steps

```
Computer Name → Change → Domain → technova.local
```

---

## Verification Command

```
ipconfig /all
```

### Explanation

- Shows full network configuration
- Used to verify DNS pointing to DC

---

## ☁️ Azure Equivalent

```
Settings → Accounts → Access Work or School → Join Azure AD
```

---

# ⚙️ 14. PowerShell Commands

## Command 1

```
Get-ADUser ahmed.hassan
```

### Explanation

- Retrieves user object from Active Directory

---

## Command 2

```
Get-ADGroupMember GG_IT_Users
```

### Explanation

- Lists all users in a group

---

## Command 3

```
Get-ADDomain
```

### Explanation

- Shows domain configuration

---

## Command 4

```
Get-ADForest
```

### Explanation

- Shows forest-level configuration

---

## ☁️ Azure Equivalent Commands

```
Get-MgUser
Get-MgGroup
```

---

# ☁️ 15. Azure Entra ID Equivalent Setup

## Step 1

```
portal.azure.com → Entra ID → Create Tenant
```

## Step 2

```
Users → New User
```

## Step 3

```
Groups → Create Security Group
```

## Step 4

```
Devices → Join Azure AD
```

## Step 5

```
Intune → Device Policies
```

---

# 🎯 16. MCSA Questions (With Answers)

## Q1 Why DNS before AD?

✔ AD depends on DNS for locating services.

---

## Q2 Why static IP for DC?

✔ Prevents service failure due to IP change.

---

## Q3 What is SYSVOL?

✔ Stores GPOs and scripts.

---

## Q4 Global vs Domain Local?

✔ Global = users

✔ Domain Local = permissions

---

## Q5 What is LDAP?

✔ Protocol to query AD objects.

---

## Q6 Why DHCP authorization?

✔ Prevent rogue DHCP servers.

---

## Q7 What is FSMO?

✔ Critical AD roles for consistency.

---

## Q8 What is OU?

✔ Organizational structure in AD.

---

## Q9 AD vs Azure AD?

✔ AD = on-prem

✔ Azure AD = cloud identity

---

## Q10 Kerberos?

✔ Ticket-based authentication system.

---

# 🧑‍💼 17. Interview Questions (With Answers)

## What is AD?

✔ Central identity system for Windows networks.

---

## What is DC?

✔ Server that handles authentication.

---

## Why DNS important?

✔ AD cannot function without DNS.

---

## What is service account?

✔ Non-human automation account.

---

## What is GPO?

✔ Policy system for users and computers.

---

# ⚠️ 18. Troubleshooting

IssueFix
Domain join failsCheck DNS
Login failsCheck groups
Internet failsCheck forwarders
DHCP failsCheck authorization

---

# 📌 19. Key Summary

- DNS is backbone of AD
- DC must always be static IP
- Groups simplify access control
- OUs structure enterprise
- Azure replaces AD in cloud

---

# 🏁 20. Final Status

```
✔ AD DS Installed
✔ DNS Configured
✔ DHCP Working
✔ Users Created
✔ Groups Created
✔ Service Account Created
✔ Password Policy Applied
✔ Azure Mapping Understood
```
