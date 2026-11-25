# Retail Help Desk Lab | Azure, Active Directory, Group Policy, PowerShell

I built a small retail company domain (`storelab.local`) in Microsoft Azure to practice the kind of work a Tier 1/Tier 2 retail IT support team handles every day: user accounts, store PCs, mapped drives, printers, and Group Policy. After building it, I broke things on purpose and fixed them, documenting each fix like a real help desk ticket.

**Skills shown:** Active Directory Domain Services, DNS, DHCP, Group Policy, Windows Server 2022, Azure VMs and networking, Azure Bastion, PowerShell, troubleshooting, ticket documentation.

---

## Table of contents

1. [Lab environment](#lab-environment)
2. [Build steps](#build-steps)
3. [Ticket write-ups](#ticket-write-ups)
4. [Lessons learned](#lessons-learned)

---

## Lab environment

| Item | Setting |
| --- | --- |
| Cloud | Azure for Students subscription |
| Region | North Central US (my subscription's policy only allows a few regions) |
| Network | `vnet-storelab` 10.0.0.0/16, subnet `snet-servers` 10.0.0.0/24, no public IPs |
| Remote access | Azure Bastion (Developer tier), browser-based RDP |
| Domain controller | `DC01`, Windows Server 2022, static private IP 10.0.0.4 |
| Store PC | `STORE101-PC01`, Windows Server 2022 standing in as a store workstation, 10.0.0.5 |
| VM size | B2als_v2 (2 vCPU, 4 GB RAM) |
| Domain | `storelab.local` (NetBIOS `STORELAB`) |

```text
You (browser) --> Azure Bastion --> vnet-storelab 10.0.0.0/24
                                     ├── DC01 (10.0.0.4): AD DS, DNS, DHCP, GPOs, file share, print server
                                     └── STORE101-PC01 (10.0.0.5): domain-joined store PC
```

---

## Build steps

### 1. Azure network and domain controller VM

- Created resource group `rg-storelab` and the virtual network with no public IPs.
- Created `DC01` and set its private IP to **static (10.0.0.4) in Azure**, not inside Windows. Azure hands out IPs through its own DHCP, so setting a static IP inside the OS can lock you out.
- Deployed Azure Bastion so I could reach the VMs without exposing RDP to the internet.

### 2. Domain controller roles

Installed AD DS, DNS, DHCP, Print Server, and Group Policy Management, then promoted DC01 to a new forest:

```powershell
Install-WindowsFeature AD-Domain-Services, DNS, DHCP, Print-Server, GPMC -IncludeManagementTools
Install-ADDSForest -DomainName "storelab.local" -DomainNetbiosName "STORELAB" -InstallDns
```

The static IP warnings are expected in Azure, since the IP is locked at the Azure level.

![Roles installed](screenshots/04a-install-roles.png)

Then I pointed the whole VNet's DNS at DC01 (10.0.0.4), so every VM finds the domain automatically. Verified the domain, services, and DNS:

![DC verification](screenshots/04b-dc-verification.png)

**DHCP (practice only):** Azure runs its own DHCP inside every VNet, so a Windows DHCP server can't hand out real leases there. I still built and authorized a scope for Store 101 registers and a reservation for a receipt printer, to practice the admin side.

![DHCP scope and reservation](screenshots/04c-dhcp-scope-reservation.png)

### 3. Retail OU structure, groups, and users

Modeled the directory like a retail chain:

```text
storelab.local
├── Stores
│   ├── Store-101 (Users, Computers)
│   └── Store-102 (Users, Computers)
├── Corporate (Users)
├── IT (Users)
├── Groups
└── Disabled Users
```

- **Security groups:** Store Managers, Store Associates, Store101-Staff, Store102-Staff, IT Support
- **Users:** store managers and associates for both stores, a corporate user, and an IT user. New users must change their password at first logon.
- **Separate admin account** (`adm-mrobles`) so daily work and admin work are split, like real IT shops do.

![OU structure and users](screenshots/05-ou-structure-users.png)

### 4. Store PC joined to the domain

Created `STORE101-PC01`, confirmed it was using DC01 for DNS, and joined it straight into the Store-101 Computers OU:

```powershell
Add-Computer -DomainName "storelab.local" -OUPath "OU=Computers,OU=Store-101,OU=Stores,DC=storelab,DC=local" -Credential STORELAB\adm-mrobles -Restart
```

Added `Store101-Staff` to the PC's Remote Desktop Users group so store employees can sign in through Bastion. Tested a real domain login as a store associate:

![Domain login as bcarter](screenshots/06-domain-login-bcarter.png)

### 5. File share, lockout policy, GPOs, and printer

**Store file share and account lockout policy** (5 bad attempts locks the account for 15 minutes):

![File share and lockout policy](screenshots/07a-file-share-lockout-policy.png)

**GPO: `Store101 - Drive Map`** (linked to Store-101): maps `\\DC01\Store101` as the **S:** drive, using item-level targeting so only `Store101-Staff` gets it.

![Drive map GPO](screenshots/07b-gpo-drive-map.png)

**GPO: `Stores - Workstation Lockdown`** (linked to Stores): blocks Control Panel for store users and shows a logon banner.

![Logon banner GPO](screenshots/07c-gpo-logon-banner.png)

**Back-office printer:** created a shared printer on DC01 and deployed it to Store 101 staff through the drive map GPO (same group targeting).

![Printer created](screenshots/07d-printer-created.png)

**Testing as a store user** on STORE101-PC01:

| Check | Result |
| --- | --- |
| `gpresult /r` shows both GPOs applied | ![gpresult](screenshots/07e-gpresult.png) |
| S: drive mapped with the opening checklist | ![S drive](screenshots/07f-s-drive-test.png) |
| Control Panel blocked | ![Control Panel blocked](screenshots/07g-control-panel-blocked.png) |
| Printer deployed from DC01 | ![Printer deployed](screenshots/07h-printer-deployed.png) |
| Logon banner shows before sign-in | ![Logon banner](screenshots/07i-logon-banner-test.png) |

---

## Ticket write-ups

I simulated five common help desk tickets. For each one, I broke something, confirmed the symptom, troubleshot it, fixed it, and verified the fix.

| # | Ticket | Category |
| --- | --- | --- |
| 001 | Associate locked out before shift | Account access |
| 003 | New hire starts tomorrow | Onboarding |
| 004 | Employee terminated | Offboarding |
| 007 | "The domain could not be contacted" | Network / DNS |
| 010 | Back-office printer offline | Printing |

### Ticket 001: Associate locked out before shift

**Priority:** High (employee can't work)
**Reported by:** Store manager, Store 101
**Affected user:** Chloe Nguyen (`cnguyen`)

**Issue:** User can't sign in to the store PC. Login keeps failing even with the right password.

**Troubleshooting:**
1. Bastion only showed a generic "unreachable or username/password incorrect" error, which doesn't say whether the account is locked. So I checked AD directly instead of guessing.
2. `Search-ADAccount -LockedOut` confirmed the account was locked.

   ![Account locked](screenshots/t1a-locked-out.png)

3. Tried to find the lockout source with event ID 4740, but no events existed because account lockout auditing wasn't enabled on the DC.
4. Pulled the lockout details from AD instead: 5 bad logon attempts (matching the lockout threshold) and the exact lockout time.

   ![Lockout details](screenshots/t1b-lockout-details.png)

**Resolution:** Enabled auditing for future lockouts, then unlocked the account. The lockout auto-expires after 15 minutes, but the user was about to start a shift, so I unlocked it manually.

```powershell
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
Unlock-ADAccount -Identity cnguyen
```

![Unlocked](screenshots/t1c-unlocked.png)

**Verification:** User signed back in to STORE101-PC01 successfully.

![User back in](screenshots/t1d-user-back-in.png)

**Root cause / prevention:** Too many wrong password attempts. Turned on lockout auditing so next time event 4740 shows which device caused the lockout (often a phone or PC with an old saved password).

---

### Ticket 003: New hire starts tomorrow at Store 101

**Request:** Set up Jordan Lee, Store Associate, Store 101.

**Steps (Active Directory Users and Computers):**
1. Created `jlee` in `Store-101 > Users` with a temporary password and "User must change password at next logon" checked.
2. Added him to `Store Associates` and `Store101-Staff`.

![New hire added to groups](screenshots/t3-new-hire-groups.png)

**Why groups matter:** Adding him to `Store101-Staff` automatically gives him the S: drive and the back-office printer through Group Policy. Nothing had to be set up one by one.

**Note:** Bastion can't display the "change your password" prompt, so a first login with a forced password change can't be tested through it. In production, the user would set their own password at first sign-in.

---

### Ticket 004: Employee terminated

**Request:** Emma Brooks's last day was today. Remove access.

**Steps:**
1. **Disabled** the account instead of deleting it, so her files and history stay available if HR or legal needs them.
2. **Removed her group memberships** (Store Associates, Store102-Staff), leaving only Domain Users. This cuts access even if someone re-enables the account by mistake.
3. **Documented it** in the account's Description field with the date and ticket number.

   ![Offboarding description](screenshots/t4a-offboard-description.png)

4. **Moved** the account to the `Disabled Users` OU so it's clearly separated from active staff.

   ![Disabled Users OU](screenshots/t4b-disabled-users-ou.png)

---

### Ticket 007: "The domain could not be contacted"

**Issue:** Store PC can't reach the domain.

**Break (simulated):** Changed the PC's DNS server to 8.8.8.8 instead of the domain controller.

**Troubleshooting:**
1. `Resolve-DnsName storelab.local` timed out, so the PC couldn't find the domain.
2. `nltest /dsgetdc:storelab.local` still succeeded, because Windows caches the last domain controller it talked to. This is why DNS problems often show up gradually ("it was working this morning").

   ![DNS broken](screenshots/t7a-dns-broken.png)

3. `ipconfig /all` showed the root cause: the DNS server was **8.8.8.8** instead of DC01 (10.0.0.4).

   ![Wrong DNS server](screenshots/t7b-wrong-dns-server.png)

**Resolution:** Reset the DNS settings back to the network default (which points at DC01) and flushed the DNS cache.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
ipconfig /flushdns
```

**Verification:** `storelab.local` resolves to 10.0.0.4 again.

![DNS fixed](screenshots/t7c-dns-fixed.png)

**Takeaway:** For any domain issue, check DNS first. Domain-joined PCs find the domain controller through DNS, so a wrong DNS server breaks logins, Group Policy, and file shares.

---

### Ticket 010: Back-office printer offline

**Issue:** Nothing prints in the back office.

**Break (simulated):** Stopped the Print Spooler service on DC01 (the print server).

**Troubleshooting:**
1. `Get-Printer -ComputerName DC01` failed: "The spooler service is not reachable."

   ![Printer error](screenshots/t10a-printer-error.png)

2. Ruled out the network first. `Test-NetConnection DC01 -Port 445` returned **TcpTestSucceeded: True**, so the server was reachable.

   ![Network OK](screenshots/t10b-network-ok.png)

3. Checked the service on the server. The Print Spooler was **Stopped**. Root cause found.

   ![Spooler stopped](screenshots/t10c-spooler-stopped.png)

**Resolution:** Restarted the spooler remotely from the store PC.

```powershell
Get-Service Spooler -ComputerName DC01 | Start-Service
```

**Verification:** `Store101-BackOffice` is listed again.

![Printer restored](screenshots/t10d-printer-restored.png)

**Takeaway:** On any "can't reach X" ticket, split it into network vs. service first. If the network is fine, the problem is on the server.

---

## Lessons learned

- **Azure for Students has region restrictions.** My first deployments failed with a policy violation. I found the "Allowed resource deployment regions" policy assignment and rebuilt in an allowed region (North Central US). That also explained why some VM sizes showed as unavailable.
- **Static IPs in Azure are set on the NIC, not in Windows.** Windows still warns about "no static IP," which is expected.
- **Bastion has limits that affect troubleshooting.** It allows one connection at a time on the Developer tier, it hides the specific Windows login error, it can't show the forced password-change screen, and it started blocking me after repeated failed logins. I worked around each one by verifying in AD directly.
- **Audit policy has to be turned on before you need it.** The lockout ticket had no event 4740 to check because auditing wasn't enabled.
- **Cached data can hide problems.** The DC lookup kept working from cache even after DNS broke.
- **Run admin tools from the right machine.** `dsa.msc` didn't work on the store PC at first because only the AD PowerShell module was installed there, not the GUI tools.
- **Cost control matters.** I stopped and deallocated the VMs between sessions and set auto-shutdown. Bastion bills hourly as long as it exists, so it should be deleted when the lab isn't in use.
