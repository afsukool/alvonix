# 🖥️ Windows Server 2022 — Day 1 Runbook
## Active Directory + DNS + DHCP Setup Guide
### *For IT Engineers and Beginners — Build It Right, Understand Why*

---

> **Lab Identity**
> | Setting | Value |
> |---|---|
> | Domain (On-Prem) | `technova.local` |
> | Domain Controller | `DC01` |
> | DC IP Address | `192.168.100.10` |
> | OS | Windows Server 2022 |
> | Client Machine | Windows 11 (`TNS01`) |

---

## 📋 Table of Contents

1. [Why This Lab Exists — The Big Picture](#1-why-this-lab-exists--the-big-picture)
2. [Architecture: What You're Building](#2-architecture-what-youre-building)
3. [Step 1 — Create the Virtual Machine (DC01)](#3-step-1--create-the-virtual-machine-dc01)
4. [Step 2 — Configure a Static IP Address](#4-step-2--configure-a-static-ip-address)
5. [Step 3 — Install Active Directory Domain Services (AD DS)](#5-step-3--install-active-directory-domain-services-ad-ds)
6. [Step 4 — Promote the Server to Domain Controller](#6-step-4--promote-the-server-to-domain-controller)
7. [Step 5 — Validate DNS is Working](#7-step-5--validate-dns-is-working)
8. [Step 6 — Configure DNS Forwarders](#8-step-6--configure-dns-forwarders)
9. [Step 7 — Create a Service Account](#9-step-7--create-a-service-account)
10. [Step 8 — Apply a Fine-Grained Password Policy](#10-step-8--apply-a-fine-grained-password-policy)
11. [Step 9 — Create Organisational Units, Users & Groups](#11-step-9--create-organisational-units-users--groups)
12. [Step 10 — Join a Windows 11 Client to the Domain](#12-step-10--join-a-windows-11-client-to-the-domain)
13. [Step 11 — Validate Everything with PowerShell](#13-step-11--validate-everything-with-powershell)
14. [Azure Entra ID Equivalents (Cloud Mapping)](#14-azure-entra-id-equivalents-cloud-mapping)
15. [Quick Reference: Key Concepts & MCSA Q&A](#15-quick-reference-key-concepts--mcsa-qa)
16. [Interview Cheat Sheet](#16-interview-cheat-sheet)
17. [Troubleshooting Guide](#17-troubleshooting-guide)
18. [Day 1 Completion Checklist](#18-day-1-completion-checklist)

---

## 1. Why This Lab Exists — The Big Picture

Before touching a single command, understand *what* you're building and *why* it matters.

### The Core Problem Active Directory Solves

Imagine a company with 500 employees. Without a central system:
- Every PC needs its own username/password managed locally
- There's no way to enforce a security policy across all machines
- You can't control what software people install or what folders they can access
- IT has to visit every desk to make any change

**Active Directory (AD) solves all of this.** It is the central nervous system of a Windows enterprise network — one place to control identity, access, and policy for every user and device.

### The Three Pillars You're Setting Up Today

```
┌─────────────────────────────────────────────────────────┐
│                      DC01 Server                        │
│                                                         │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │   AD DS      │  │   DNS    │  │     DHCP         │  │
│  │              │  │          │  │                  │  │
│  │ WHO you are  │  │ WHERE to │  │ WHAT IP you get  │  │
│  │ (Identity)   │  │ find it  │  │ (Address assign) │  │
│  │              │  │ (Names)  │  │                  │  │
│  └──────────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
              ┌───────────┘
              │
     ┌────────▼────────┐
     │  Windows 11     │
     │  TNS01          │
     │  Domain Joined  │
     └─────────────────┘
```

| Service | What It Does | Real-World Analogy |
|---|---|---|
| **AD DS** | Stores all users, computers, and policies | Company HR database + Security guard |
| **DNS** | Translates names to IP addresses | Phone book for computers |
| **DHCP** | Automatically assigns IP addresses | Hotel receptionist assigning room numbers |
| **GPO** | Pushes policies to all machines | Company rulebook enforced automatically |

---

## 2. Architecture: What You're Building

### On-Premises Layout

```
technova.local (Forest/Domain)
│
└── DC01 (192.168.100.10)
    ├── AD DS         → Stores users, groups, computers, policies
    ├── DNS           → Resolves technova.local names internally
    ├── DHCP          → Assigns IPs to domain clients
    └── Group Policy  → Enforces security and config settings
         │
         └── Clients (192.168.100.x range)
              └── TNS01 (Windows 11) — domain joined
```

### Azure / Cloud Equivalent Layout

```
Microsoft Entra ID (tenant)
├── Users & Groups        → Same as AD Users/Groups
├── Devices (Entra Joined) → Same as Domain Join
├── Intune Policies        → Same as Group Policy (GPO)
└── Conditional Access     → Same as fine-grained access control
```

> **💡 Key Insight:** Everything you learn on-prem has a direct cloud equivalent. This lab is your foundation for understanding both worlds.

---

## 3. Step 1 — Create the Virtual Machine (DC01)

### What You're Doing
Creating a virtual server that will become your Domain Controller. This is the machine that runs Active Directory.

### VM Specifications

| Setting | Value | Why This Value |
|---|---|---|
| **Name** | DC01 | Naming convention: DC = Domain Controller, 01 = first one |
| **RAM** | 4 GB | Minimum for Server 2022 with AD DS running comfortably |
| **CPU** | 2 vCPUs | Enough for a lab environment |
| **Disk** | 80 GB | Enough for OS + AD database (NTDS.dit) + logs |
| **OS** | Windows Server 2022 | Current LTS release |
| **Network** | Host-only or Internal | Isolated from real network for lab safety |

### Steps
1. Open VMware Fusion / Workstation / Hyper-V
2. Create New VM → Select Windows Server 2022 ISO
3. Apply the specs above
4. Boot the VM and complete Windows Server installation
5. Choose **"Desktop Experience"** during setup (gives you a GUI)
6. Set a strong Administrator password — write it down

### ☁️ Azure Equivalent
Instead of a VM in VMware, you'd deploy an **Azure Virtual Machine** from the Marketplace using a Windows Server 2022 image. The concept is identical — just cloud-hosted.

---

## 4. Step 2 — Configure a Static IP Address

### Why This Step Is Critical

> ⚠️ **This is the most important pre-AD step.** A Domain Controller MUST have a static IP. If the IP changes, every domain-joined computer will lose contact with AD — logins fail, policies stop applying, everything breaks.

DNS records point to `192.168.100.10`. If that IP changes, those records are wrong. SYSVOL replication uses the IP. Kerberos tickets reference the DC's address. **Static IP = stable foundation.**

### How to Do It

**Open Network Settings:**
```
Win + R → ncpa.cpl → Enter
```
> `ncpa.cpl` is a Control Panel shortcut for **Network Connections**. It's faster than clicking through Settings.

**Right-click your NIC → Properties → Internet Protocol Version 4 (TCP/IPv4) → Properties**

**Enter these values:**

| Field | Value | Explanation |
|---|---|---|
| IP Address | `192.168.100.10` | This server's fixed address |
| Subnet Mask | `255.255.255.0` | /24 network — 254 usable hosts |
| Default Gateway | `192.168.100.2` | Your router/hypervisor NAT gateway |
| Preferred DNS | `192.168.100.10` | **Points to itself** — DC is its own DNS server |
| Alternate DNS | `8.8.8.8` | Fallback for internet names |

> **💡 Why does DNS point to itself?**
> Active Directory requires DNS to function. The DC runs a DNS server, so it must resolve its own domain. Pointing DNS at itself means it can find `dc01.technova.local` and all AD service records (SRV records).

### Verify the IP Was Applied
```powershell
ipconfig /all
```
Look for your NIC showing `192.168.100.10` with the correct mask and gateway.

### ☁️ Azure Equivalent
In Azure, you configure a **static private IP on the NIC** of your VM via the Azure Portal → VM → Networking → Network Interface → IP Configurations.

---

## 5. Step 3 — Install Active Directory Domain Services (AD DS)

### What You're Doing
Installing the AD DS **role** — this adds the software/components needed to run a domain. Installing the role doesn't create a domain yet; that happens in Step 4.

### Navigation Path
```
Server Manager → Manage → Add Roles and Features
→ Role-based or feature-based installation
→ Select DC01
→ Roles: ✅ Active Directory Domain Services
→ Add Features (accept all defaults)
→ Install
```

### What Gets Installed
- **AD DS binaries** — the actual AD software
- **DNS Server** — will be auto-configured during promotion
- **Management tools** — ADUC, ADAC, PowerShell AD module

> **💡 Why not install DNS separately?**
> When you promote to DC in the next step, the wizard offers to install DNS automatically and creates the correct DNS zones for your domain. It's easier and less error-prone to let the AD promotion handle DNS setup.

### ☁️ Azure Equivalent
With **Microsoft Entra ID**, there's nothing to install. Microsoft manages the identity platform. You simply create a tenant and start adding users.

---

## 6. Step 4 — Promote the Server to Domain Controller

### What You're Doing
This is where the magic happens. You're turning a plain Windows Server into an **Active Directory Domain Controller** — the authority for `technova.local`.

### Navigation Path
After AD DS installs, a yellow warning flag appears in Server Manager:

```
Server Manager → Flag icon (⚠️) → Promote this server to a domain controller
```

### Configuration Choices

**Deployment Configuration:**
- Select: **Add a new forest**
- Root domain name: `technova.local`

> **Why `.local`?** It's convention for internal (non-internet-routable) domains. Never use a real public domain (like `.com`) for an internal AD — it causes DNS split-brain issues.

**Domain Controller Options:**
| Setting | Value | Why |
|---|---|---|
| Forest Functional Level | Windows Server 2016 or 2022 | Enables all modern AD features |
| Domain Functional Level | Windows Server 2016 or 2022 | Match the forest level |
| DNS Server | ✅ Checked | Auto-creates DNS zones |
| Global Catalog | ✅ Checked | Required on first DC |
| DSRM Password | Set a strong password | **Write this down** — needed to restore AD if it breaks |

**Additional Options:**
- NetBIOS name: `TECHNOVA` (auto-populated from domain name)

**Paths (leave as default for lab):**
| Path | Default Location | What's Stored Here |
|---|---|---|
| Database (NTDS.dit) | `C:\Windows\NTDS` | All AD objects (users, groups, computers) |
| Log files | `C:\Windows\NTDS` | Transaction logs for AD changes |
| SYSVOL | `C:\Windows\SYSVOL` | GPOs and login scripts — replicated to all DCs |

> **💡 What is SYSVOL?** A shared folder on every DC that stores Group Policy Objects and logon scripts. When you create a GPO, it gets stored in SYSVOL and replicated to all DCs in the domain. Clients pull policies from SYSVOL at login.

**Click Install — server will restart automatically.**

### What Happened After Restart?
- A new AD forest and domain (`technova.local`) was created
- DNS zones were created: `technova.local` (Forward) and `100.168.192.in-addr.arpa` (Reverse)
- The server is now the **Primary Domain Controller (PDC)** — it holds all 5 FSMO roles
- You can now log in as `TECHNOVA\Administrator`

### ☁️ Azure Equivalent
In Azure: **Create a new Entra ID tenant** — no forest/domain/FSMO concepts exist in cloud identity. Microsoft handles all the infrastructure.

---

## 7. Step 5 — Validate DNS is Working

### Why Validate Before Proceeding?
DNS must be working correctly before you do anything else. Everything in AD — logins, policy, replication — depends on DNS. If DNS is broken, AD is broken.

### Validation Commands

**Check if DNS Service is Running:**
```powershell
Get-Service DNS
```
**Expected Output:**
```
Status   Name               DisplayName
------   ----               -----------
Running  DNS                DNS Server
```
> If Status shows `Stopped`, run: `Start-Service DNS` then check if it stays running.

**Verify Internal Name Resolution:**
```cmd
nslookup dc01.technova.local
```
**Expected Output:**
```
Server:  dc01.technova.local
Address:  192.168.100.10

Name:    dc01.technova.local
Address:  192.168.100.10
```
> This confirms your DC can resolve its own name — the most basic DNS test for AD.

**Verify External Name Resolution:**
```cmd
nslookup google.com
```
**Expected Output:**
```
Server:  dc01.technova.local
Address:  192.168.100.10

Non-authoritative answer:
Name:    google.com
Address:  142.250.x.x
```
> If this fails, your DNS forwarders aren't working (configure them in Step 6).

**Check AD-Specific DNS Records (SRV Records):**
```cmd
nslookup -type=srv _ldap._tcp.technova.local
```
> SRV records tell domain clients how to find the DC. If these are missing, domain joins and logins will fail.

---

## 8. Step 6 — Configure DNS Forwarders

### What Are DNS Forwarders and Why Do You Need Them?

Your DC's DNS server knows everything about `technova.local` — it's the authority. But it knows nothing about `google.com`, `microsoft.com`, or any internet name.

**Without forwarders:** Users can't browse the internet because the DC's DNS can't resolve external names.

**With forwarders:** When someone asks for `google.com`, your DC says *"I don't know that one — let me ask Google's DNS for you"* and passes the request to `8.8.8.8`.

```
Client asks: "What's the IP for google.com?"
      │
      ▼
DC01 DNS: "I don't know google.com — forwarding..."
      │
      ▼
8.8.8.8 (Google DNS): "google.com = 142.250.x.x"
      │
      ▼
DC01 returns answer to client ✅
```

### Navigation Path
```
Server Manager → Tools → DNS
→ Expand DC01
→ Right-click DC01 → Properties
→ Forwarders tab
→ Edit → Add:
    8.8.8.8    (Google DNS)
    1.1.1.1    (Cloudflare DNS)
→ OK → Apply
```

### Verify Forwarders Work
```cmd
nslookup microsoft.com
```
Should return an IP address. If it does, internet DNS resolution is working through your forwarders.

### ☁️ Azure Equivalent
- **Azure DNS Private Zones** — for internal name resolution
- **Custom DNS settings on VNet** — to control how VMs resolve names
- Azure's public DNS handles external resolution automatically

---

## 9. Step 7 — Create a Service Account

### What Is a Service Account and Why Create One?

A **service account** is a special user account created for a program or automated task — not for a human to log into interactively.

**Bad practice:** Using `Administrator` to join all computers to the domain.
- If someone learns the Administrator password, they own everything.
- Audit logs show "Administrator did this" — you don't know *which* person.
- You can't easily revoke access without changing the main admin password.

**Good practice:** Create a dedicated `domainjoin` account with *only* the permissions needed to join computers to the domain.

### Navigation Path
```
Server Manager → Tools → Active Directory Users and Computers (ADUC)
→ technova.local
→ Users container
→ Right-click → New → User
```

### Account Details

| Field | Value |
|---|---|
| First Name | Domain Join |
| Last Name | Account |
| User Logon Name | `domainjoin` |
| Password | Set a complex password |
| Password never expires | ✅ (for lab — review in production) |
| User cannot change password | ✅ |

### Principle of Least Privilege
> **💡 Security Concept:** Give every account only the minimum permissions it needs to do its job. The `domainjoin` account should only be able to add computers to the domain — nothing else. This limits damage if the account is compromised.

### ☁️ Azure Equivalent
- **Managed Identity** — automatically managed credentials for Azure services (no password needed)
- **Service Principal (App Registration)** — for applications that need to authenticate to Azure

---

## 10. Step 8 — Apply a Fine-Grained Password Policy

### What Is a Fine-Grained Password Policy (FGPP)?

**Default behaviour:** One global password policy applies to ALL users in the domain (via Default Domain Policy GPO).

**Problem:** A service account like `domainjoin` might need a different policy than regular users. Or your executive team needs stronger passwords than general staff.

**Solution:** Fine-Grained Password Policy — lets you apply *different* password rules to *specific* users or groups.

### Navigation Path
```
Server Manager → Tools → Active Directory Administrative Center (ADAC)
→ technova.local
→ System container
→ Password Settings Container
→ New → Password Settings
```

### Lab Policy Settings (Simplified for Lab Use)

| Setting | Lab Value | Production Recommendation |
|---|---|---|
| Name | `PSO_ServiceAccounts` | Descriptive name |
| Precedence | `10` | Lower number = higher priority |
| Min Password Length | `1` | **12+ in production** |
| Password Complexity | Disabled | **Enabled in production** |
| Password History | `0` | **24 in production** |
| Min Password Age | `0 days` | **1 day in production** |
| Max Password Age | `0 (never)` | **90 days in production** |
| Account Lockout | Disabled | **5 attempts in production** |

**Apply to:** Add `domainjoin` user to this policy's **"Directly Applies To"** section.

> ⚠️ **Lab vs Production:** The lab settings are minimal so you don't get locked out while learning. In a real environment, always enforce complexity, history, and lockout thresholds.

### ☁️ Azure Equivalent
- **Conditional Access Policies** — control *when* and *how* users can authenticate
- **Authentication Strength** — enforce MFA for specific groups
- **Password Protection** — ban common passwords in Entra ID

---

## 11. Step 9 — Create Organisational Units, Users & Groups

### Understanding the Structure

Think of your AD like a company org chart stored in a database:

```
technova.local (the domain)
├── Builtin  (default — don't touch)
├── Computers (default — move computers out of here)
├── Users (default — move users out of here)
└── _TechNova (your custom OU structure)
    ├── IT
    │   ├── ahmed.hassan (user)
    │   └── GG_IT_Users (group)
    ├── Finance
    │   ├── fatima.ali
    │   └── GG_Finance_Users
    ├── Sales
    │   ├── omar.khan
    │   └── GG_Sales_Users
    ├── Marketing
    │   ├── noor.ibrahim
    │   └── GG_Marketing_Users
    └── Operations
        ├── zain.abdullah
        └── GG_Operations_Users
```

> **💡 Why create OUs instead of using default containers?**
> You can't apply Group Policy to the default `Users` or `Computers` containers — only to OUs. Proper OU structure lets you apply specific GPOs to specific departments.

### Create Organisational Units

```
ADUC → technova.local → Right-click → New → Organizational Unit
```

Create one OU per department: `IT`, `Finance`, `Sales`, `Marketing`, `Operations`

### Create Users

| Name | Username | Department OU | Job Title |
|---|---|---|---|
| Ahmed Hassan | `ahmed.hassan` | IT | IT Administrator |
| Fatima Ali | `fatima.ali` | Finance | Finance Analyst |
| Omar Khan | `omar.khan` | Sales | Sales Executive |
| Noor Ibrahim | `noor.ibrahim` | Marketing | Marketing Manager |
| Zain Abdullah | `zain.abdullah` | Operations | Operations Coordinator |

**PowerShell (faster way — create all users at once):**
```powershell
$users = @(
    @{Name="Ahmed Hassan"; SAM="ahmed.hassan"; OU="IT"},
    @{Name="Fatima Ali";   SAM="fatima.ali";   OU="Finance"},
    @{Name="Omar Khan";    SAM="omar.khan";    OU="Sales"},
    @{Name="Noor Ibrahim"; SAM="noor.ibrahim"; OU="Marketing"},
    @{Name="Zain Abdullah";SAM="zain.abdullah";OU="Operations"}
)

foreach ($u in $users) {
    New-ADUser `
        -Name $u.Name `
        -SamAccountName $u.SAM `
        -UserPrincipalName "$($u.SAM)@technova.local" `
        -Path "OU=$($u.OU),DC=technova,DC=local" `
        -AccountPassword (ConvertTo-SecureString "P@ssword123!" -AsPlainText -Force) `
        -Enabled $true
    Write-Host "Created: $($u.Name)" -ForegroundColor Green
}
```

### Create Security Groups

**Group naming convention:** `GG_` prefix = **G**lobal **G**roup — used for organizing users.

| Group Name | Location OU | Purpose |
|---|---|---|
| `GG_IT_Users` | IT | All IT department staff |
| `GG_Finance_Users` | Finance | All Finance department staff |
| `GG_Sales_Users` | Sales | All Sales department staff |
| `GG_Marketing_Users` | Marketing | All Marketing department staff |
| `GG_Operations_Users` | Operations | All Operations department staff |

**PowerShell (create all groups):**
```powershell
$depts = @("IT","Finance","Sales","Marketing","Operations")
foreach ($dept in $depts) {
    New-ADGroup `
        -Name "GG_$($dept)_Users" `
        -GroupScope Global `
        -GroupCategory Security `
        -Path "OU=$dept,DC=technova,DC=local" `
        -Description "Security group for $dept department users"
    Write-Host "Created: GG_$($dept)_Users" -ForegroundColor Cyan
}
```

**Add users to their groups:**
```powershell
Add-ADGroupMember -Identity "GG_IT_Users"         -Members "ahmed.hassan"
Add-ADGroupMember -Identity "GG_Finance_Users"    -Members "fatima.ali"
Add-ADGroupMember -Identity "GG_Sales_Users"      -Members "omar.khan"
Add-ADGroupMember -Identity "GG_Marketing_Users"  -Members "noor.ibrahim"
Add-ADGroupMember -Identity "GG_Operations_Users" -Members "zain.abdullah"
```

### Group Types Explained

| Type | Scope | Use For |
|---|---|---|
| **Global Group** (`GG_`) | Within the domain | Organising users by role/department |
| **Domain Local Group** (`DL_`) | Local to domain | Assigning permissions to resources |
| **Universal Group** | Across forests | Multi-domain / forest-wide access |

> **The AGDLP Rule (Best Practice):**
> **A**ccounts → **G**lobal Groups → **D**omain **L**ocal Groups → **P**ermissions
> Put users in Global Groups → put Global Groups into Domain Local Groups → assign permissions to Domain Local Groups.

### ☁️ Azure Equivalent
- **Entra ID Security Groups** — same concept, cloud-managed
- **Microsoft 365 Groups** — adds mailbox + SharePoint + Teams to the group
- **Dynamic Groups** — auto-add users based on attributes (e.g., all users in `department=IT`)

---

## 12. Step 10 — Join a Windows 11 Client to the Domain

### What You're Doing
Connecting the Windows 11 machine (`TNS01`) to `technova.local` so users can log in with their AD credentials.

### Pre-Check: DNS Must Point to DC

On the Windows 11 machine, DNS must point to `192.168.100.10` (the DC). If DNS points elsewhere, the client can't find the domain.

```cmd
# Run on TNS01
ipconfig /all
```
Look for: `DNS Servers . . . . . . . . . . . : 192.168.100.10`

If it doesn't say `192.168.100.10`, set it:
```
ncpa.cpl → NIC Properties → IPv4 → DNS = 192.168.100.10
```

### Join the Domain

```
Win + R → sysdm.cpl
```
> `sysdm.cpl` opens **System Properties**. This is the classic, fastest way to rename a PC or join a domain.

```
Computer Name tab
→ Change...
→ Select "Domain" radio button
→ Enter: technova.local
→ OK
→ Enter credentials: domainjoin / [password]  ← using your service account!
→ OK
```

**You'll see:** *"Welcome to the technova.local domain"*

Restart the machine when prompted.

### After Restart — Log in as a Domain User
```
Other User → ahmed.hassan → [password]
```
Or:
```
TECHNOVA\ahmed.hassan
```

### Verify Domain Join (from DC01)
```powershell
# On DC01 — check if TNS01 appears in AD
Get-ADComputer -Filter {Name -eq "TNS01"} | Select Name, DistinguishedName
```

### ☁️ Azure Equivalent
```
Settings → Accounts → Access Work or School → Join this device to Azure Active Directory
→ Enter your Entra ID credentials
→ Device appears in Entra ID → Devices
```

---

## 13. Step 11 — Validate Everything with PowerShell

Run these on DC01 to confirm your entire setup is correct.

### Domain & Forest Health
```powershell
# View domain info
Get-ADDomain | Select DNSRoot, NetBIOSName, DomainMode, PDCEmulator

# View forest info
Get-ADForest | Select Name, ForestMode, Domains, GlobalCatalogs

# Check all FSMO role holders
netdom query fsmo
```

### Check Users
```powershell
# Look up a specific user
Get-ADUser ahmed.hassan -Properties Department, Title, Enabled

# List all users in the domain
Get-ADUser -Filter * | Select Name, SamAccountName, Enabled

# Find disabled accounts
Get-ADUser -Filter {Enabled -eq $false} | Select Name
```

### Check Groups
```powershell
# See who's in a group
Get-ADGroupMember GG_IT_Users | Select Name, SamAccountName

# See all groups
Get-ADGroup -Filter * | Select Name, GroupScope, GroupCategory
```

### Check Computers
```powershell
# List all domain-joined computers
Get-ADComputer -Filter * | Select Name, IPv4Address, Enabled

# Find computers that haven't logged on in 90 days (stale accounts)
$cutoff = (Get-Date).AddDays(-90)
Get-ADComputer -Filter {LastLogonDate -lt $cutoff} | Select Name, LastLogonDate
```

### Check DNS
```powershell
# Verify DNS zones
Get-DnsServerZone

# Check SRV records (critical for AD)
Resolve-DnsName -Name _ldap._tcp.technova.local -Type SRV

# Check DC's own A record
Resolve-DnsName dc01.technova.local
```

### Overall AD Health Check
```powershell
# Run the built-in DC diagnostic tool
dcdiag /test:dns /v
dcdiag /test:replications

# Check AD replication (important when you add more DCs later)
repadmin /showrepl
```

### ☁️ Azure / Microsoft Graph Equivalents
```powershell
# Install Microsoft Graph module
Install-Module Microsoft.Graph

# Connect
Connect-MgGraph -Scopes "User.Read.All","Group.Read.All"

# Get all users
Get-MgUser | Select DisplayName, UserPrincipalName

# Get group members
Get-MgGroupMember -GroupId <group-id>
```

---

## 14. Azure Entra ID Equivalents (Cloud Mapping)

### Side-by-Side Comparison

| On-Premises | Azure / Entra ID | Notes |
|---|---|---|
| Active Directory Domain Services | Microsoft Entra ID | Core identity platform |
| Domain Controller (DC01) | Entra ID tenant | Microsoft manages the infrastructure |
| Forest / Domain | Tenant | One tenant = one organisation |
| Users (ADUC) | Entra ID Users | Same concept, web-managed |
| Security Groups | Entra Security Groups | Identical concept |
| OU (Organizational Unit) | Administrative Units | Similar but less granular |
| Group Policy (GPO) | Intune + Conditional Access | Cloud-native policy enforcement |
| Domain Join | Entra Join / Hybrid Join | Same result, different method |
| DHCP Server | Azure VNet + DHCP scope | Built into Azure networking |
| DNS Server | Azure Private DNS Zone | Managed DNS in Azure |
| Service Account | Managed Identity / Service Principal | No passwords to manage |
| FSMO Roles | Not applicable | Microsoft handles this |
| LDAP (port 389) | Microsoft Graph API (HTTPS) | REST-based in the cloud |
| Kerberos authentication | OAuth 2.0 / OIDC | Modern authentication protocols |

### Setting Up Entra ID (Step-by-Step)
```
1. Go to portal.azure.com → Sign in
2. Search "Microsoft Entra ID" → Create Tenant
   → Organisation name: TechNova
   → Initial domain: technova.onmicrosoft.com
3. Users → New User → Create
4. Groups → New Group → Security → Add members
5. Devices → Entra Join (from Windows 11 Settings)
6. Endpoint Manager (Intune) → Device Configuration Policies
7. Security → Conditional Access → New Policy
```

---

## 15. Quick Reference: Key Concepts & MCSA Q&A

### Why DNS Must Be Set Up Before AD
AD uses DNS **SRV records** to publish the location of domain services. When a client wants to log in, it asks DNS: *"Who's the Kerberos server for technova.local?"* DNS answers with the DC's IP. Without DNS, clients can't find the DC — authentication fails entirely.

### Why a Static IP for the DC
DNS records, Kerberos tickets, LDAP referrals, and replication all reference the DC's IP address. If the IP changes, all of those references break simultaneously. **One IP change = complete domain outage.**

### FSMO Roles (Flexible Single Master Operations)
Five special roles that can only be held by one DC at a time (to prevent conflicts):

| Role | What It Does |
|---|---|
| PDC Emulator | Handles password changes, time sync, and legacy clients |
| RID Master | Issues unique ID numbers for new AD objects |
| Infrastructure Master | Tracks references to objects in other domains |
| Schema Master | Controls changes to the AD schema (structure) |
| Domain Naming Master | Controls adding/removing domains from the forest |

### Authentication: Kerberos vs NTLM

| Protocol | How It Works | Use Case |
|---|---|---|
| **Kerberos** | Ticket-based — client gets a ticket from DC, presents it to resources | Default for domain-joined systems |
| **NTLM** | Challenge-response — older, less secure | Fallback for non-domain or legacy scenarios |

> In AD, Kerberos is always preferred. NTLM is the fallback.

### LDAP
**Lightweight Directory Access Protocol** — the language used to query Active Directory.
- Port 389 (unencrypted) / Port 636 (LDAPS — encrypted)
- PowerShell's `Get-ADUser` uses LDAP under the hood
- Used by many applications (email servers, VPNs, HR systems) to look up users

### DHCP Authorization
When you install DHCP on a domain member, it must be **authorized** in AD before it starts handing out IPs. This prevents a rogue/accidental DHCP server from giving out wrong addresses on your network. Only authorized DHCP servers can operate in an AD domain.

### Common MCSA/Exam Questions

**Q: What is the SYSVOL folder?**
A: A shared folder replicated to all DCs that stores Group Policy Objects (GPOs) and logon scripts. Located at `C:\Windows\SYSVOL`. Clients download policies from SYSVOL at logon.

**Q: What's the difference between a Global Group and a Domain Local Group?**
A: Global Groups contain users — you put department members in them. Domain Local Groups control access to resources (like shared folders). Best practice: put Global Groups *into* Domain Local Groups, then assign permissions to the Domain Local Group (AGDLP model).

**Q: What port does Kerberos use?**
A: Port **88** (UDP/TCP).

**Q: What port does LDAP use?**
A: Port **389** (plain) / **636** (LDAPS).

**Q: What is a tombstone lifetime in AD?**
A: The period (default 180 days) before a deleted AD object is permanently removed. Important for DC restore scenarios.

---

## 16. Interview Cheat Sheet

### Common Questions and Strong Answers

**"What is Active Directory?"**
> Active Directory is Microsoft's centralised identity and access management platform for Windows environments. It authenticates who you are (via Kerberos), authorizes what you can access (via group membership and permissions), and enforces configuration through Group Policy.

**"What is a Domain Controller?"**
> A Domain Controller is a Windows Server running the AD DS role. It stores the AD database (NTDS.dit), handles authentication requests, hosts DNS for the domain, and replicates changes to other DCs for redundancy.

**"Why is DNS so important to Active Directory?"**
> AD is completely dependent on DNS. It uses DNS SRV records to publish the location of Kerberos, LDAP, and other services. Without working DNS, domain clients can't locate the DC, authentication fails, and Group Policy can't be applied.

**"What is a service account?"**
> A service account is a non-interactive domain user account created for an application or automated process. It follows the principle of least privilege — given only the permissions needed for its specific function. It allows auditing of application actions separately from human actions.

**"What is Group Policy (GPO)?"**
> Group Policy is AD's central configuration and security management system. GPOs are applied to OUs and can control hundreds of settings — desktop background, software installation, firewall rules, password policies, drive mappings — automatically across all machines in the domain.

**"What is the difference between AD and Azure AD / Entra ID?"**
> Traditional Active Directory is on-premises — you own and manage the servers, using Kerberos/LDAP protocols, joined via domain join. Microsoft Entra ID (formerly Azure AD) is a cloud identity platform managed by Microsoft, using OAuth 2.0/OIDC for modern apps and Azure AD Join for devices. Many organisations run both in a **hybrid** configuration, syncing on-prem AD to Entra ID using Entra Connect.

**"What would you check first if a user can't log in?"**
> I'd follow a logical sequence: (1) Check if the account is enabled and not locked in ADUC, (2) Verify the user is in the correct group with the right permissions, (3) Check if the client machine can reach the DC on the network, (4) Run `dcdiag` on the DC to check for AD/DNS issues, (5) Check Event Viewer on both the client and DC for error codes.

---

## 17. Troubleshooting Guide

### Domain Join Fails

**Symptom:** *"The following error occurred attempting to join the domain... The network path was not found"*

**Diagnostic steps:**
```cmd
# Step 1: Can the client ping the DC?
ping 192.168.100.10

# Step 2: Can the client resolve the domain name?
nslookup technova.local

# Step 3: Is DNS pointing to the DC?
ipconfig /all | findstr DNS
```

**Fix:** Set DNS on the client to `192.168.100.10` → then retry domain join.

---

### User Can't Log In

**Symptom:** Login fails with wrong password error or "the trust relationship failed"

**Check:**
```powershell
# Check if account is locked
Get-ADUser ahmed.hassan -Properties LockedOut, BadLogonCount | Select Name, LockedOut, BadLogonCount

# Unlock if locked
Unlock-ADAccount -Identity ahmed.hassan

# Check if account is disabled
Get-ADUser ahmed.hassan | Select Enabled

# Enable if disabled
Enable-ADAccount -Identity ahmed.hassan
```

---

### Internet Not Working (From Domain Clients)

**Symptom:** Internal sites work but `google.com` doesn't resolve

**Check:** DNS forwarders on the DC
```
DNS Manager → DC01 → Properties → Forwarders
Verify 8.8.8.8 and 1.1.1.1 are listed
```
```cmd
# Test from DC
nslookup google.com 8.8.8.8
```

---

### DHCP Not Assigning IPs

**Symptom:** Clients stuck on APIPA address (169.254.x.x)

**Check:**
```powershell
# Is DHCP service running?
Get-Service DHCPServer

# Is DHCP authorized in AD?
Get-DhcpServerInDC
```
**Fix:** Open DHCP console → Right-click server → Authorize

---

### AD Replication Issues (Multi-DC environments)

```powershell
# Check replication status
repadmin /showrepl

# Force replication
repadmin /syncall /AdeP

# Run full DC diagnostics
dcdiag /test:replications /v
```

---

### Quick Diagnostic Reference

| Problem | First Command to Run | What to Look For |
|---|---|---|
| Domain join fails | `nslookup technova.local` (on client) | Should return DC's IP |
| Login fails | `Get-ADUser <name> -Properties LockedOut` | LockedOut = True |
| Internet broken | `nslookup google.com` (on DC) | Should return an IP |
| DHCP broken | `Get-Service DHCPServer` | Status = Running |
| AD broken | `dcdiag /v` | Any FAIL results |
| DNS broken | `dcdiag /test:dns /v` | Look for errors |

---

## 18. Day 1 Completion Checklist

Use this to confirm everything was done correctly before moving to Day 2.

### Infrastructure
- [ ] DC01 VM created with correct specs (4GB RAM, 2 vCPU, 80GB disk)
- [ ] Windows Server 2022 installed with Desktop Experience
- [ ] Static IP configured: `192.168.100.10`
- [ ] DNS on DC points to itself (`192.168.100.10`)

### Active Directory
- [ ] AD DS role installed
- [ ] Domain promoted: `technova.local` (new forest)
- [ ] NetBIOS name set: `TECHNOVA`
- [ ] DSRM password recorded securely

### DNS
- [ ] DNS service running (`Get-Service DNS` → Running)
- [ ] Internal resolution works (`nslookup dc01.technova.local` → `192.168.100.10`)
- [ ] External resolution works (`nslookup google.com` → returns IP)
- [ ] Forwarders configured: `8.8.8.8` and `1.1.1.1`

### Users & Groups
- [ ] Service account created: `domainjoin`
- [ ] Fine-Grained Password Policy created and applied to `domainjoin`
- [ ] 5 OUs created (IT, Finance, Sales, Marketing, Operations)
- [ ] 5 users created in their respective OUs
- [ ] 5 security groups created (`GG_*`)
- [ ] Users added to their respective groups

### Client Setup
- [ ] TNS01 DNS set to `192.168.100.10`
- [ ] TNS01 joined to `technova.local`
- [ ] Domain user login tested successfully from TNS01

### Validation
- [ ] `Get-ADDomain` runs without errors
- [ ] `Get-ADUser ahmed.hassan` returns correct user
- [ ] `Get-ADGroupMember GG_IT_Users` shows Ahmed Hassan
- [ ] `Get-ADComputer -Filter *` shows TNS01
- [ ] `dcdiag /v` shows no FAIL results

---

> **📌 What's Next (Day 2 Preview)**
> - Group Policy Objects (GPOs) — creating and linking policies
> - Shared folders and NTFS/share permissions
> - DHCP server setup and scope configuration
> - Roaming profiles and folder redirection
> - Read-Only Domain Controller (RODC) concepts
> - Basic AD backup and restore

---

*Runbook version: Day 1 | Domain: technova.local | Last updated for Windows Server 2022*
