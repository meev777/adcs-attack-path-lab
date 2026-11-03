# AD CS Attack-Path Lab — Manual Build Runbook

**Status:** Final baseline for manual execution, updated for QEMU/KVM  
**Purpose:** Build and verify an isolated AD DS + AD CS lab that can later host one controlled attack path or ESC scenario at a time.  
**Scope of this document:** Infrastructure, domain, CA, workstation, verification, evidence, and snapshots. This runbook does not contain exploitation commands. The target path build is in `planing/01-attack-path-build.md`.  
**Source of truth:** This runbook, `planing/01-attack-path-build.md`, and `diagrams/architecture.md`. If another instruction conflicts with these files, stop and resolve the conflict first.

---

## 0. Read this before touching the existing VMs

You have already started three VMs and installed some roles. Do these checks before continuing. They prevent the most common rebuild-causing mistakes.

### 0.1 Locked design

| Item | Locked value |
|---|---|
| Hypervisor | QEMU/KVM managed with virt-manager (libvirt) |
| Isolation | libvirt isolated network only; no NAT, no bridged adapter |
| Lab subnet | `10.10.10.0/24` |
| Virtual network | libvirt network `ADCS-LAB` on bridge `virbr10` |
| Host address | `10.10.10.1/24`, DHCP disabled |
| Domain | `esi-sba-meev.local` |
| NetBIOS name | `ESI-SBA-MEEV` (confirm with `Get-ADDomain`) |
| DC | Windows Server 2019 Desktop Experience, `MEEV-DC01`, `10.10.10.10/24`, DNS `10.10.10.10` |
| CA | Windows Server 2019 Desktop Experience, `CA01`, `10.10.10.20/24`, DNS `10.10.10.10` |
| CA name | `MEEV-ROOT-CA` |
| CA type | Enterprise Root CA, RSA 2048, SHA-256, 10 years |
| Workstation | Windows 10/11 Pro or Enterprise, `MS01`, `10.10.10.30/24`, DNS `10.10.10.10` |
| Gateway | Empty on guests where possible; `10.10.10.1` is the host-side libvirt bridge |
| Kali | Host operating system, not a guest VM in this baseline |
| Web Enrollment | Installed because ESC8 may be studied later; HTTP only inside the isolated lab |

The lab now uses domain `esi-sba-meev.local`, DC `MEEV-DC01`, workstation `MS01`, and CA `CA01`. These names match the attack-path diagram. Keep them consistent across every VM, snapshot, command, and evidence file. If `Get-ADDomain` reports a NetBIOS name other than `ESI-SBA-MEEV`, use that reported value in logon and group commands; do not rebuild the domain only for the NetBIOS value.

### 0.2 Corrections to the draft plan

1. **Use QEMU/KVM and libvirt, not VirtualBox.** The existing lab network is the libvirt network `ADCS-LAB`, backed by `virbr10`. Verify it is isolated (no `<forward>` element or `<forward mode='none'/>`) and has no DHCP range. `virbr10` will be DOWN while the guests are off; that is normal. `virsh net-list --all` may show no networks when run as a normal user because the system connection is not selected; use `sudo virsh -c qemu:///system net-list --all` or virt-manager.
2. **Use `esi-sba-meev.local`.** Do not rename an already-promoted domain. `.local` is accepted here only for an isolated lab that follows the attack-path writeup. Never connect this domain or its DNS to a real network. If the DC is promoted under another name, rebuild it rather than attempting a domain rename.
3. **Use `10.10.10.0/24`.** The current DC screenshot already shows `10.10.10.10/24`; keep `CA01` at `.20` and `MS01` at `.30`.
4. **Kali stays on the host.** Do not add a Kali VM to this baseline. If you later add one, attach it only to the isolated libvirt network and review the isolation first.
5. **CA01 must be a domain member before AD CS Enterprise configuration.** Use a domain Enterprise Admins account, normally `ESI-SBA-MEEV\Administrator` or `Administrator@esi-sba-meev.local`, not `CA01\Administrator`.
6. **The CA common name is `MEEV-ROOT-CA`.** Keep host identity (`CA01`) separate from CA identity. If the CA is already configured under another name, document the deviation; do not rebuild only for the name.
7. **Your current DC screenshot shows gateway `10.10.10.1` and DNS `127.0.0.1`.** The gateway is the libvirt bridge and is acceptable only after the network is confirmed isolated. Change the DC's preferred DNS to `10.10.10.10` for consistency. Remove any public DNS forwarder such as `8.8.8.8`; the `Forw` test in `dcdiag /test:dns` may still fail in a network with no internet, and that is expected.
8. **The attack path is built after the clean baseline.** Step 3.6 creates the identities now. Do not create a vulnerable template or add the other intentional weaknesses until you reach the controlled attack-path sheet `planing/01-attack-path-build.md`.

### 0.3 Existing-build gate

Complete every check. Do not continue if a check fails.

#### Check A — libvirt isolated network

1. Open **virt-manager → Edit → Connection Details → Virtual Networks**, or run the commands below as root.
2. Select **ADCS-LAB**. Confirm it is active, uses bridge `virbr10`, has IP `10.10.10.1/24`, and has no DHCP range.
3. Confirm the network is isolated, not NAT:

```bash
sudo virsh -c qemu:///system net-list --all
sudo virsh -c qemu:///system net-dumpxml ADCS-LAB
ip -4 addr show virbr10
```

Expected in the XML: no `<forward>` element or an explicit `<forward mode='none'/>`, a `virbr10` bridge, and no `<dhcp>` block. A missing `<forward>` element is the normal isolated-network form. If it says `<forward mode='nat'/>` or names a physical interface, stop and recreate the network as isolated before continuing.

`virbr10` may show `state DOWN` when all guests are shut down. Start the network with `sudo virsh -c qemu:///system net-start ADCS-LAB` if needed. Do not use the libvirt `default` NAT network (`192.168.122.0/24`) for this lab.

If you must define the network from XML, use this shape and omit DHCP:

```xml
<network>
  <name>ADCS-LAB</name>
  <forward mode='none'/>
  <bridge name='virbr10' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'/>
</network>
```

#### Check B — DC state

On the DC, run:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain,DomainRole
Get-NetIPConfiguration
```

Then, if the AD module is available:

```powershell
Get-ADDomain
Get-DnsServerForwarder
```

Decision table:

| Observation | Action |
|---|---|
| Name is `MEEV-DC01`, domain is `esi-sba-meev.local`, IP/DNS are correct | Continue; verify DNS before CA join |
| Name is not `MEEV-DC01` and the host is not yet promoted | Rename it to `MEEV-DC01`, reboot, then continue Phase 1 |
| Domain is `lab.example.test` or any other name | Stop; rebuild the DC with `esi-sba-meev.local`; do not attempt a domain rename |
| IP is not `10.10.10.10` and domain is not promoted | Change it before promotion |
| IP is not `10.10.10.10` and domain is already promoted | Prefer rebuild; if you keep it, update every address, DNS record, and evidence file consistently |
| Gateway is `10.10.10.1` | Acceptable only if `ADCS-LAB` is confirmed isolated; the gateway is the host-side bridge |
| A public forwarder such as `8.8.8.8` is configured | Remove it; this lab has no internet by design. Do not add it back to make `dcdiag` pass |

#### Check C — CA01 state

On CA01, run:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain,DomainRole
Get-Service CertSvc -ErrorAction SilentlyContinue
certutil -cainfo
```

Decision table:

| Observation | Action |
|---|---|
| Name is not `CA01` and CA is not configured | Rename it to `CA01`, reboot, then continue |
| Name is not `CA01` and CA is already configured | Do not rename; rebuild the CA host |
| `PartOfDomain` is `False` and CA is not configured | Join the domain first, then install/configure AD CS |
| `PartOfDomain` is `False` and CA is configured | Rebuild the CA; it is likely Standalone |
| Role installed but `certutil -cainfo` says the CA is not configured | Keep the role, join the domain, then run the configuration wizard |
| `certutil -cainfo` says `Standalone Root CA` | Rebuild the CA role; conversion is not supported |
| `certutil -cainfo` says `Enterprise Root CA` with name `MEEV-ROOT-CA` | Continue; verify domain membership and move to MS01 |
| `certutil -cainfo` says `Enterprise Root CA` with a different name | Acceptable if domain-joined; record the deviation and update the docs, or rebuild for an exact baseline |
| `DomainRole` is 4 or 5 | CA01 is a domain controller; rebuild it as a member server, not a DC |

The CA screenshots in `evidence/screenshots/CA01/` show the role being installed while the host was still `WIN-94R5M7NBF60` and configuration was launched with `CA01\Administrator`. They are useful process evidence, but they are **not proof of a correctly configured Enterprise CA**. Capture new evidence after the checks above. The domain account for configuration is `ESI-SBA-MEEV\Administrator` or `Administrator@esi-sba-meev.local`.

#### Check D — libvirt network adapters

For every guest:

- Exactly one network adapter.
- Network source: **Virtual network 'ADCS-LAB'**.
- Model: `e1000e` unless you have installed virtio drivers; record the choice.
- No NAT, no bridged adapter, no macvtap.
- SPICE clipboard sharing and file transfer disabled.
- No shared folders or virtiofs shares attached.

#### Check E — Snapshot and secrets

1. Shut down all three VMs cleanly.
2. Record the current state in local notes.
3. Take a snapshot of each VM named `PRE-FIX-YYYY-MM-DD` so you can inspect the old state if needed. This is not a reusable baseline.
4. Put all passwords in a local password manager. Do not put them in this repository, screenshots, or shell history.
5. Do not publish product keys, PFX/P12 files, hashes, or raw VM disks.
6. For libvirt snapshots, use virt-manager **Snapshot** or `sudo virsh snapshot-create-as <domain> <name> "<description>" --disk-only --atomic` only after reviewing the snapshot type. Do not snapshot running domain controllers for a reusable baseline.

---

## 1. Build order and checkpoints

Do not skip ahead. Each checkpoint must pass before the next phase.

| Step | Work | Pass gate | Snapshot |
|---|---|---|---|
| 0 | libvirt network and VM isolation | `ADCS-LAB` isolated, `10.10.10.1/24`, DHCP off, one NIC per VM | None |
| 1 | DC base OS, name, static IP | `MEEV-DC01`, `10.10.10.10`, DNS points to DC | `MEEV-DC01-00-OS` / `MEEV-DC01-01-NET` |
| 2 | AD DS forest and DNS | `Get-ADDomain` works; DNS SRV records resolve | `MEEV-DC01-02-ADDS-VERIFIED` |
| 3 | Lab OUs, groups, attack-path identities | Users are non-admin and resolvable; pre-auth and SPN state recorded | `MEEV-DC01-03-OBJECTS` |
| 4 | CA01 base, name, static IP, domain join | CA01 is a domain member using DC DNS | `CA01-01-JOINED` |
| 5 | AD CS Enterprise Root CA and baseline template | `certutil -cainfo` says Enterprise Root CA; CertSvc running; User template published | `CA01-02-ENTERPRISE-CA` |
| 6 | MS01 base, domain join, enrollment test | User certificate appears in `certmgr.msc` | `MS01-01-JOINED` |
| 7 | Baseline acceptance | All checks pass; evidence captured; no vulnerable templates | `ESI-01-BASELINE` |
| 8 | Controlled attack-path build | Weaknesses match `01-attack-path-build.md`; snapshot is isolated and reversible | `ESI-02-ATTACK-PATH` |

---

## 2. Phase 0 — Hypervisor and isolation

### 2.1 VM specification

| VM | vCPU | RAM | Disk | Network | Operating system |
|---|---|---:|---:|---|---|
| MEEV-DC01 | 2 | 4096 MB | 60 GB | libvirt `ADCS-LAB` | Windows Server 2019 Desktop Experience |
| CA01 | 2 | 4096 MB | 60 GB | libvirt `ADCS-LAB` | Windows Server 2019 Desktop Experience |
| MS01 | 2 | 4096 MB | 80 GB | libvirt `ADCS-LAB` | Windows 10/11 Pro or Enterprise |
| Host Kali | — | — | — | `virbr10` | Existing host |

If your existing servers are Windows Server 2022, keep both servers on the same version, record the deviation in the build log, and update the architecture document. Do not mix undocumented OS assumptions.

If you already created MS01 with 60 GB, that is sufficient for Windows 10. Use 80 GB for Windows 11.

### 2.2 Configure the libvirt isolated network

The existing lab network is `ADCS-LAB` on `virbr10`. Confirm and, if necessary, recreate it as an isolated network:

1. Open **virt-manager → Edit → Connection Details → Virtual Networks**.
2. Select **ADCS-LAB** and open its properties.
3. Set IPv4 to `10.10.10.1/24`.
4. Disable DHCP.
5. Confirm forwarding is **isolated** / no NAT. Do not select **NAT** or a physical device.
6. Apply the changes and start the network.
7. On the host, verify:

```bash
sudo virsh -c qemu:///system net-list --all
sudo virsh -c qemu:///system net-dumpxml ADCS-LAB
ip -4 addr show virbr10
```

Expected: `ADCS-LAB` is active, the XML has no NAT/route/bridge forward element or has `<forward mode='none'/>`, there is no `<dhcp>` block, and `virbr10` has `inet 10.10.10.1/24`.

A libvirt domain name is only a local label; the Windows computer name is what AD uses. On the current host the three inactive domains may still be labelled `ESCLAB-DC0`, `ESCLAB-CA0`, and `ESCLAB-PC)`. Use the actual label in `virsh` commands, or rename each domain while it is shut off. Do not confuse these labels with the required Windows names `MEEV-DC01`, `CA01`, and `MS01`.

If the network does not exist, define it from XML:

```xml
<network>
  <name>ADCS-LAB</name>
  <forward mode='none'/>
  <bridge name='virbr10' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'/>
</network>
```

```bash
sudo virsh -c qemu:///system net-define /path/to/adcs-lab.xml
sudo virsh -c qemu:///system net-start ADCS-LAB
sudo virsh -c qemu:///system net-autostart ADCS-LAB
```

Do not use the libvirt `default` NAT network. Do not add a `<forward mode='nat'/>` or a physical interface to `ADCS-LAB`.

### 2.3 Configure every guest

For `MEEV-DC01`, `CA01`, and `MS01`:

1. Open the VM in virt-manager and choose **Show virtual hardware details → NIC**.
2. Keep exactly one NIC.
3. Network source: **Virtual network 'ADCS-LAB'**.
4. Model: `e1000e` unless virtio drivers are installed and documented.
5. Remove any additional NICs.
6. In the VM's SPICE settings, disable clipboard sharing and file transfer.
7. Do not attach shared folders or virtiofs shares.
8. Do not enable USB passthrough for lab data transfer.
9. Boot the guest and confirm it has only one usable Ethernet adapter.

### 2.4 Snapshot rules

- Shut down the guest cleanly before taking a baseline snapshot.
- Use descriptive snapshot names with the phase number.
- Keep snapshots local; do not commit them.
- If you restore a DC snapshot, restore CA01 and MS01 to a compatible snapshot as well. Restoring only MEEV-DC01 can leave domain machine-account passwords out of sync.
- Do not store passwords or private keys in snapshot names or descriptions.
- Snapshot while powered off, then power the required VMs back on for the next phase.

---

## 3. Phase 1 — MEEV-DC01 base operating system and network

### 3.1 Install and rename

1. Install Windows Server 2019 with **Desktop Experience**, not Server Core.
2. Set a unique local Administrator password and store it in the password manager.
3. Set the correct time zone.
4. Do not join a domain yet.
5. Open PowerShell as Administrator and run:

```powershell
Rename-Computer -NewName "MEEV-DC01" -Restart
```

6. After the reboot, log in as the local Administrator.

### 3.2 Configure the static address

Use **Settings → Network & Internet → Ethernet → Change adapter options → Ethernet → Properties → Internet Protocol Version 4 (TCP/IPv4) → Properties**.

Set:

| Field | Value |
|---|---|
| IP address | `10.10.10.10` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.10.10.1` if libvirt supplied it; otherwise empty. This is the isolated bridge, not an internet gateway. |
| Preferred DNS server | `10.10.10.10` |
| Alternate DNS server | Empty |

Verify:

```powershell
Get-NetIPConfiguration
Get-DnsClientServerAddress -AddressFamily IPv4
ipconfig /all
```

Expected: one IPv4 address `10.10.10.10/24`; gateway is empty or `10.10.10.1`; DNS is `10.10.10.10`.

The existing DC screenshot shows DNS `127.0.0.1`. Change it to `10.10.10.10` for consistency. Both are locally valid on a DC, but the static address makes the lab evidence and client configuration simpler. If the gateway is any other address, stop and fix the VM's network attachment before continuing.

Do not configure public DNS servers. The lab has no internet gateway by design.

### 3.3 Install AD DS and DNS

GUI:

1. Open **Server Manager → Manage → Add Roles and Features**.
2. Installation type: **Role-based or feature-based installation**.
3. Server: `MEEV-DC01`.
4. Server roles: select **Active Directory Domain Services** and **DNS Server**.
5. Add features when prompted.
6. Continue through the wizard and install.
7. Do not close Server Manager when the installation finishes.

PowerShell alternative:

```powershell
Install-WindowsFeature AD-Domain-Services,DNS -IncludeManagementTools
```

### 3.4 Promote MEEV-DC01 as a new forest

GUI:

1. In Server Manager, click the notification flag and choose **Promote this server to a domain controller**.
2. Deployment configuration: **Add a new forest**.
3. Root domain name: `esi-sba-meev.local`.
4. Domain controller options:
   - Forest functional level: Windows Server 2016 or the highest available level.
   - Domain functional level: Windows Server 2016 or the highest available level.
   - DNS server: checked.
   - Global Catalog: checked.
   - Read-only domain controller: unchecked.
   - DSRM password: generate a unique password and store it in the password manager.
5. DNS options: no delegation is needed for a new root forest. Continue through the warning.
6. Additional options: NetBIOS name `ESI-SBA-MEEV`.
7. Paths: leave the defaults.
8. Review options and install. The server reboots automatically.

PowerShell alternative:

```powershell
Install-ADDSForest `
  -DomainName "esi-sba-meev.local" `
  -DomainNetbiosName "ESI-SBA-MEEV" `
  -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password")
```

The command reboots the server when it finishes.

### 3.5 Verify the domain and DNS

Log in as `ESI-SBA-MEEV\Administrator`. Run:

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADForest
Get-ADDomainController
nltest /dsgetdc:esi-sba-meev.local
Resolve-DnsName meev-dc01.esi-sba-meev.local
Resolve-DnsName esi-sba-meev.local
Resolve-DnsName -Name _ldap._tcp.dc._msdcs.esi-sba-meev.local -Type SRV
dcdiag /test:dns /v
Get-Service NTDS,DNS,Netlogon,KDC
```

Pass criteria:

- `Get-ADDomain` shows DNS root `esi-sba-meev.local` and NetBIOS name `ESI-SBA-MEEV`.
- `nltest` finds `\\MEEV-DC01.esi-sba-meev.local`.
- `Resolve-DnsName meev-dc01.esi-sba-meev.local` returns `10.10.10.10`.
- The LDAP SRV record resolves to MEEV-DC01.
- NTDS, DNS, Netlogon, and KDC services are running.
- `dcdiag /test:dns /v` passes the AD/DNS tests. The `Forw` test may fail in this isolated lab if there is no internet. Do not add a public forwarder to make it pass. If `Forw` is the only failure and `Get-DnsServerForwarder` is empty, document it as expected. Investigate any failure in `Auth`, `Basc`, `Del`, `Dyn`, or `RReg`.
- `Get-DnsServerForwarder` returns no public resolver such as `8.8.8.8`. If it does, remove it and rerun the test.

If `Get-DnsServerForwarder` shows `8.8.8.8` or another public resolver, remove it and rerun the DNS test:

```powershell
Remove-DnsServerForwarder -IPAddress 8.8.8.8 -Force
dcdiag /test:dns /v
```

If DNS fails, do not continue to CA01. Fix DNS first.

If you are building from scratch, shut down MEEV-DC01 and take snapshot `MEEV-DC01-02-ADDS-VERIFIED` before creating lab objects.

### 3.6 Create the lab OUs, groups, and attack-path identities

Run PowerShell as `ESI-SBA-MEEV\Administrator` on `MEEV-DC01`.

These commands assume the OUs have not been created. If you previously created `OU=LAB`, leave it empty and unused; do not reuse it. The lab uses `OU=ESI-SBA-MEEV` from this point forward.

```powershell
Import-Module ActiveDirectory

$DomainDN = (Get-ADDomain).DistinguishedName
$RootOU = "OU=ESI-SBA-MEEV,$DomainDN"

New-ADOrganizationalUnit -Name "ESI-SBA-MEEV" -Path $DomainDN
New-ADOrganizationalUnit -Name "Users" -Path $RootOU
New-ADOrganizationalUnit -Name "Computers" -Path $RootOU
New-ADOrganizationalUnit -Name "Groups" -Path $RootOU
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path $RootOU

New-ADGroup -Name "GG-ESI-SBA-Users" -SamAccountName "GG-ESI-SBA-Users" -GroupScope Global -GroupCategory Security -Path "OU=Groups,$RootOU"
New-ADGroup -Name "GG-Helpdesk" -SamAccountName "GG-Helpdesk" -GroupScope Global -GroupCategory Security -Path "OU=Groups,$RootOU"
New-ADGroup -Name "GG-File-Service" -SamAccountName "GG-File-Service" -GroupScope Global -GroupCategory Security -Path "OU=Groups,$RootOU"
New-ADGroup -Name "GG-ESC1-Enrollees" -SamAccountName "GG-ESC1-Enrollees" -GroupScope Global -GroupCategory Security -Path "OU=Groups,$RootOU"

$PwMike = Read-Host -AsSecureString "Password for mike.ross"
New-ADUser -Name "Mike Ross" -SamAccountName "mike.ross" -UserPrincipalName "mike.ross@esi-sba-meev.local" -Path "OU=Users,$RootOU" -AccountPassword $PwMike -Enabled $true -ChangePasswordAtLogon $false -PasswordNeverExpires $true

$PwFiles = Read-Host -AsSecureString "Password for svc-filesmgr"
New-ADUser -Name "File Manager" -SamAccountName "svc-filesmgr" -UserPrincipalName "svc-filesmgr@esi-sba-meev.local" -Path "OU=ServiceAccounts,$RootOU" -AccountPassword $PwFiles -Enabled $true -ChangePasswordAtLogon $false -PasswordNeverExpires $true

$PwHarper = Read-Host -AsSecureString "Password for h.harper"
New-ADUser -Name "Helpdesk Harper" -SamAccountName "h.harper" -UserPrincipalName "h.harper@esi-sba-meev.local" -Path "OU=Users,$RootOU" -AccountPassword $PwHarper -Enabled $true -ChangePasswordAtLogon $false -PasswordNeverExpires $true

$PwJohn = Read-Host -AsSecureString "Password for h.john"
New-ADUser -Name "Harry John" -SamAccountName "h.john" -UserPrincipalName "h.john@esi-sba-meev.local" -Path "OU=Users,$RootOU" -AccountPassword $PwJohn -Enabled $true -ChangePasswordAtLogon $false -PasswordNeverExpires $true

$PwAdmin = Read-Host -AsSecureString "Password for meev.admin"
New-ADUser -Name "MEEV Admin" -SamAccountName "meev.admin" -UserPrincipalName "meev.admin@esi-sba-meev.local" -Path "OU=Users,$RootOU" -AccountPassword $PwAdmin -Enabled $true -ChangePasswordAtLogon $false -PasswordNeverExpires $true

Add-ADGroupMember -Identity "GG-ESI-SBA-Users" -Members "mike.ross","svc-filesmgr","h.harper","h.john","meev.admin"
Add-ADGroupMember -Identity "GG-Helpdesk" -Members "h.harper"
Add-ADGroupMember -Identity "GG-File-Service" -Members "svc-filesmgr"
Add-ADGroupMember -Identity "GG-ESC1-Enrollees" -Members "h.harper"
```

Identity mapping:

| Attack-path identity | sAMAccountName | Baseline role | Weakness added later |
|---|---|---|---|
| Mike Ross | `mike.ross` | Standard domain user | Kerberos pre-authentication disabled for AS-REP roasting |
| File Manager | `svc-filesmgr` | Service-style domain user | SPN added for Kerberoasting; SMB share write access |
| Helpdesk Harper | `h.harper` | Standard domain user; member of `GG-Helpdesk` and `GG-ESC1-Enrollees` | Local admin on MS01; ESC1 enrollment right |
| Harry John | `h.john` | Standard domain user | GenericAll over `meev.admin` |
| MEEV Admin | `meev.admin` | Standard domain user | Member of `DnsAdmins` and `Remote Management Users` |

Use fictional lab passwords and store them only in your local password manager. If you need the original writeup's cracking behavior, use the writeup's lab passwords locally; never commit them. The passwords do not belong in this repository, screenshots, shell history, or the build log.

Verify the clean baseline:

```powershell
$AttackPathUsers = "mike.ross","svc-filesmgr","h.harper","h.john","meev.admin"
$AttackPathUsers | ForEach-Object {
  Get-ADUser -Identity $_ -Properties DoesNotRequirePreAuth,ServicePrincipalNames,MemberOf |
    Select-Object Name,SamAccountName,Enabled,DoesNotRequirePreAuth,ServicePrincipalNames
}
Get-ADGroupMember "GG-ESC1-Enrollees" | Select-Object Name,SamAccountName
Get-ADGroupMember "GG-Helpdesk" | Select-Object Name,SamAccountName
Get-ADGroupMember "Domain Admins" | Select-Object Name,SamAccountName
Get-ADOrganizationalUnit -Filter * | Select-Object Name,DistinguishedName
```

Expected at this stage:

- All five identities are enabled.
- `mike.ross` has `DoesNotRequirePreAuth = False`.
- `svc-filesmgr` has no service principal names yet.
- `h.harper` is in `GG-Helpdesk` and `GG-ESC1-Enrollees`.
- No lab identity is in `Domain Admins` or `Enterprise Admins`.
- `meev.admin` is not yet in `DnsAdmins` or `Remote Management Users`.

The intentional weaknesses are added only in `planing/01-attack-path-build.md`, after the clean baseline is verified and snapshotted.

### 3.7 MEEV-DC01 checkpoint

1. Confirm the DNS forwarder is clean and the expected `dcdiag` result is documented.
2. Shut down MEEV-DC01 cleanly.
3. Take snapshot `MEEV-DC01-03-OBJECTS`.
4. Capture evidence:
   - E03: MEEV-DC01 IP and DNS configuration.
   - E04: domain verification (`Get-ADDomain`, NetBIOS name, DNS SRV lookup, and `dcdiag` summary).
5. Record the commands, the identity mapping, and any deviations in the build log.

---

## 4. Phase 2 — CA01 and the Enterprise CA

### 4.1 CA01 current-state decision

Do this before installing or configuring AD CS.

Run on CA01:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain,DomainRole
Get-Service CertSvc -ErrorAction SilentlyContinue
certutil -cainfo
```

Interpretation:

- **Name is not CA01 and the CA is not configured:** rename to `CA01` now, reboot, then continue.
- **Name is not CA01 and the CA is configured:** do not rename; rebuild the CA host.
- **Not domain-joined:**

  Rename and join the domain before installing or configuring AD CS.

- **Role installed, but `certutil -cainfo` says it is not configured:** keep the role, join the domain, then run the AD CS configuration wizard.

- **`certutil -cainfo` reports `Standalone Root CA`:** this is the blocker. A Standalone CA cannot be converted to an Enterprise CA. Use a snapshot from before AD CS, or remove the role and reinstall CA01. The cleanest beginner path is to rebuild CA01 from a clean snapshot or ISO.

- **`certutil -cainfo` reports `Enterprise Root CA`:** confirm CA01 is domain-joined. The expected name is `MEEV-ROOT-CA`; a different name is acceptable if you record the deviation and update this runbook. Do not rebuild solely because of the name if the CA is otherwise correct.

### 4.2 Recovering from an incorrect CA configuration

Use this if the CA was configured as Standalone, or if the host was renamed after the CA was configured.

1. Shut down CA01.
2. Prefer restoring the snapshot taken immediately before AD CS was installed.
3. If no suitable snapshot exists:
   - Remove the AD CS role from **Server Manager → Manage → Remove Roles and Features**.
   - When prompted, remove the CA database and private key.
   - Reboot.
   - If this was ever an Enterprise CA, the safest option is to reinstall CA01 from a clean Windows installation so no stale CA objects remain in AD.
4. Rejoin the domain if needed, verify the hostname, and install AD CS again.
5. Do not reuse the old CA name until the old CA objects and database are gone.

Do not try to force a Standalone CA into Enterprise mode with registry edits. Rebuild instead.

### 4.3 Configure the CA01 host

Power on MEEV-DC01 before joining CA01 to the domain.

If CA01 is not yet installed or needs to be rebuilt:

1. Install Windows Server 2019 Desktop Experience.
2. Set a unique local Administrator password.
3. Rename the computer to `CA01` before joining the domain:

```powershell
Rename-Computer -NewName "CA01" -Restart
```

4. Configure the static network:

| Field | Value |
|---|---|
| IP address | `10.10.10.20` |
| Subnet mask | `255.255.255.0` |
| Default gateway | Empty or `10.10.10.1` (isolated libvirt bridge only) |
| Preferred DNS server | `10.10.10.10` |
| Alternate DNS server | Empty |

5. Verify:

```powershell
Get-NetIPConfiguration
Get-DnsClientServerAddress -AddressFamily IPv4
Resolve-DnsName meev-dc01.esi-sba-meev.local
```

6. Join the domain `esi-sba-meev.local` using `ESI-SBA-MEEV\Administrator`.
7. Reboot and log in as `ESI-SBA-MEEV\Administrator`.

Verify domain membership:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain,DomainRole
whoami
nltest /dsgetdc:esi-sba-meev.local
Resolve-DnsName ca01.esi-sba-meev.local
```

Expected: computer name `CA01`, domain `esi-sba-meev.local`, `PartOfDomain: True`, and DNS resolution. The `DomainRole` for a member server should not be 4 or 5.

On MEEV-DC01, verify the computer object:

```powershell
Import-Module ActiveDirectory
Get-ADComputer CA01
```

Take snapshot `CA01-01-JOINED` before installing AD CS.

### 4.4 Install the AD CS role

1. Open **Server Manager → Manage → Add Roles and Features**.
2. Installation type: **Role-based or feature-based installation**.
3. Server: `CA01`.
4. Role: **Active Directory Certificate Services**.
5. Role services:
   - **Certification Authority** — required.
   - **Certificate Authority Web Enrollment** — install it in this baseline. It is needed for the browser enrollment check and for ESC8 later.
   - Do not install Online Responder or Network Device Enrollment Service for this baseline.
6. Add features when prompted. Web Enrollment requires IIS; accept the dependency.
7. Install.
8. Leave Server Manager open and click **Configure Active Directory Certificate Services on the destination server**. If the link is gone, use **Server Manager → AD CS → More → Configure Active Directory Certificate Services**.

### 4.5 Configure an Enterprise Root CA

Use these exact choices:

| Wizard page | Selection |
|---|---|
| Credentials | `ESI-SBA-MEEV\Administrator` (or a dedicated member of Enterprise Admins) |
| Role Services | Certification Authority and Certificate Authority Web Enrollment |
| Setup Type | **Enterprise CA** |
| CA Type | **Root CA** |
| Private Key | **Create a new private key** |
| Cryptography | RSA 2048, SHA-256, Microsoft Software Key Storage Provider (`RSA#Microsoft Software Key Storage Provider`) |
| CA Common Name | `MEEV-ROOT-CA` |
| Distinguished name suffix | `DC=esi-sba-meev,DC=local` |
| Validity | 10 years |
| Database locations | Defaults |
| Confirmation | Configure |

Critical checks:

- The credentials field must not say `CA01\Administrator`. If it does, click **Change** and enter the domain account. A local account cannot configure an Enterprise CA.
- The account must be a member of **Enterprise Admins** in the `esi-sba-meev.local` forest. The built-in `ESI-SBA-MEEV\Administrator` is a member by default. A custom account added to Enterprise Admins must log off and back on before the membership is effective.
- If **Enterprise CA** is unavailable, stop. CA01 is not domain-joined, DNS is wrong, or the credentials are insufficient.
- Do not choose Standalone. Do not use `CA01` as the CA common name.
- If Web Enrollment asks how to configure IIS, choose the option that creates a new IIS web site or uses the default web site, whichever the wizard offers in your version. Use port 80 inside the isolated lab.
- Wait for the configuration to complete. Restart the CA service or server if the wizard requests it.

### 4.6 Verify the CA

Run on CA01:

```powershell
Get-Service CertSvc
certutil -ping
certutil -cainfo
certutil -getreg CA\CRLPublicationURLs
certutil -getreg CA\CACertPublicationURLs
```

Pass criteria:

- `CertSvc` is running.
- `certutil -ping` reports that the CA is available.
- `certutil -cainfo` shows `Name: MEEV-ROOT-CA` (or the documented CA name) and `Type: Enterprise Root CA`.
- The CA console opens without an error:
  - Open `certsrv.msc`.
  - The CA node shows the expected CA name.
  - Certificate Templates, Issued Certificates, Revoked Certificates, and Pending Requests are visible.
- The publication URLs are recorded in the build log. Do not change them yet.

Test Web Enrollment locally:

```text
http://localhost/certsrv
```

You should see the Microsoft Certificate Services welcome page. Do not expose this URL outside the isolated libvirt network.

### 4.7 Publish a safe baseline template

Do not create a vulnerable template yet. Use the built-in **User** template to verify enrollment.

1. Open `certsrv.msc`.
2. Expand the CA, right-click **Certificate Templates**, and choose **New → Certificate Template to Issue**.
3. Select **User** and click **OK**.
4. Confirm the template appears under Certificate Templates.

This template is a baseline enrollment test. It is not an ESC scenario template. Do not modify its subject settings, EKUs, or ACLs in this phase.

### 4.8 CA checkpoint and evidence

1. Capture E05: CA01 domain membership and hostname.
2. Capture E06: `certutil -cainfo` showing Enterprise Root CA, plus the CA console node.
3. Capture E07: the published User template properties (baseline).
4. Shut down CA01 cleanly and take snapshot `CA01-02-ENTERPRISE-CA`.
5. Note the CA name, type, validity, and publication URLs in the build log.

Do not proceed to MS01 until `certutil -cainfo` says Enterprise Root CA.

---

## 5. Phase 3 — MS01 and the enrollment test

### 5.1 Install, rename, and join

Ensure MEEV-DC01 is powered on before joining MS01.

1. Install Windows 10/11 **Pro or Enterprise**. Windows Home cannot join a domain.
2. Set a unique local Administrator password.
3. Rename the computer to `MS01`.
4. Configure the static network:

| Field | Value |
|---|---|
| IP address | `10.10.10.30` |
| Subnet mask | `255.255.255.0` |
| Default gateway | Empty or `10.10.10.1` (isolated libvirt bridge only) |
| Preferred DNS server | `10.10.10.10` |
| Alternate DNS server | Empty |

5. Verify DNS:

```powershell
Resolve-DnsName meev-dc01.esi-sba-meev.local
Resolve-DnsName ca01.esi-sba-meev.local
```

6. Join `esi-sba-meev.local` using `ESI-SBA-MEEV\Administrator`.
7. Reboot and log in as `ESI-SBA-MEEV\h.harper`.

### 5.2 Verify the workstation

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain
nltest /dsgetdc:esi-sba-meev.local
Resolve-DnsName ca01.esi-sba-meev.local
gpupdate /force
```

Expected: `MS01`, domain `esi-sba-meev.local`, domain controller found, CA01 resolves to `10.10.10.20`.

### 5.3 Safe certificate enrollment and Web Enrollment test

Power on MEEV-DC01 and CA01 before starting the enrollment test.

1. Log in as `ESI-SBA-MEEV\h.harper`.
2. Open `certmgr.msc`.
3. Right-click **Personal**, then **All Tasks → Request New Certificate**.
4. Click **Next** through the enrollment policy page.
5. Select the **User** template.
6. Click **Enroll**.
7. Confirm a certificate appears under **Personal → Certificates**.
8. On CA01, open `certsrv.msc → Issued Certificates` and confirm the request was issued.
9. Optional trust check: open an elevated `certlm.msc` and confirm the `MEEV-ROOT-CA` certificate appears under **Trusted Root Certification Authorities → Certificates**. If it is not there yet, run `gpupdate /force` and allow a few minutes; otherwise check enterprise CA publication and DNS.

Web Enrollment check from MS01:

```text
http://ca01.esi-sba-meev.local/certsrv
```

The welcome page should load. If Internet Explorer Enhanced Security Configuration blocks it, add the URL to the local intranet/trusted sites zone. Do not disable security controls globally.

If the User template is not listed:

- Confirm it is published in `certsrv.msc`.
- Run `gpupdate /force` on MS01.
- Log off and back on.
- Confirm `h.harper` can read and enroll the template.
- Confirm CA01 is an Enterprise CA.

### 5.4 MS01 checkpoint

1. Capture the successful certificate enrollment as E07 or a related evidence item.
2. Shut down all three VMs cleanly.
3. Take snapshot `MS01-01-JOINED` on MS01.
4. Take a snapshot named `ESI-01-BASELINE` on MEEV-DC01, CA01, and MS01 at the same point in time. Treat the three snapshots as one baseline set.
5. Record the final state in the build log.

---

## 6. Phase 4 — Baseline acceptance and reproducibility

### 6.1 Acceptance checklist

Mark each item only after verifying it.

- [ ] libvirt network `ADCS-LAB` is isolated, uses `10.10.10.1/24`, and has DHCP disabled.
- [ ] MEEV-DC01, CA01, and MS01 each have exactly one NIC on `ADCS-LAB`.
- [ ] No NAT, bridged, or macvtap adapter exists on any lab guest.
- [ ] MEEV-DC01 is `meev-dc01.esi-sba-meev.local`, `10.10.10.10`, DNS points to itself.
- [ ] `Get-ADDomain` shows `esi-sba-meev.local` and NetBIOS `ESI-SBA-MEEV`.
- [ ] DNS SRV records for the domain resolve to MEEV-DC01.
- [ ] `dcdiag /test:dns` passes the AD/DNS tests; `Forw` failure is documented as expected if no forwarders exist.
- [ ] `Get-DnsServerForwarder` contains no public resolver such as `8.8.8.8`.
- [ ] CA01 is a domain member named `CA01` at `10.10.10.20`.
- [ ] CA01 uses MEEV-DC01 for DNS.
- [ ] `certutil -cainfo` reports `Enterprise Root CA` named `MEEV-ROOT-CA` (or a documented CA-name deviation).
- [ ] Certificate Services is running and `certutil -ping` succeeds.
- [ ] The CA console opens without errors.
- [ ] Web Enrollment loads locally and from MS01.
- [ ] The built-in User template is published.
- [ ] `h.harper` can enroll a User certificate from MS01.
- [ ] No vulnerable ESC template has been created or published.
- [ ] The clean identity baseline exists, and `mike.ross` still has pre-auth enabled and `svc-filesmgr` has no SPN.
- [ ] No real credentials, PFX files, private keys, or VM disks are in the repository.
- [ ] Snapshots and evidence are captured.

### 6.2 Evidence mapping

| ID | Capture | Notes |
|---|---|---|
| E01 | QEMU/KVM VM inventory | Show MEEV-DC01, CA01, MS01, and the `ADCS-LAB` isolated network |
| E02 | libvirt isolated network settings | Must show `10.10.10.1/24`, no NAT/route forward (no `<forward>` or `<forward mode='none'/>`), and DHCP disabled |
| E03 | MEEV-DC01 IP/DNS configuration | `ipconfig /all` or `Get-NetIPConfiguration` |
| E04 | Domain verification | `Get-ADDomain`, NetBIOS name, SRV lookup, `dcdiag` result |
| E05 | CA01 domain membership | Hostname, domain, `PartOfDomain: True` |
| E06 | Enterprise CA configuration | `certutil -cainfo` and CA console |
| E07 | Certificate template/enrollment | Baseline User template and successful enrollment |
| E08 | Attack-path validation | Controlled scenario from `01-attack-path-build.md` |
| E09 | Detection evidence | Later defensive phase |
| E10 | Remediation retest | Later defensive phase |

The existing `evidence/screenshots/E01-host-only-network.png` is a VirtualBox host-only capture. Replace it with QEMU/KVM + virt-manager evidence showing the libvirt `ADCS-LAB` network and VM NICs. The current DC screenshots (`evidence/screenshots/DC01/`) are partial evidence: `s1-domaininfos.png` shows the correct hostname/IP/DNS, and `s3-test-result.png` shows the DNS test with a `Forw` failure caused by a public forwarder. Remove `8.8.8.8`, rerun the test, and recapture before marking E03/E04 complete. The CA screenshots document an earlier role-install attempt and are not proof of an Enterprise CA.

### 6.3 Build log template

Copy this table into a local note or a separate non-sensitive document. Do not include passwords.

| Date | Phase | VM | Action | Expected result | Observed result | Evidence | Snapshot |
|---|---|---|---|---|---|---|---|
| | | | | | | | |

### 6.4 Snapshot map

| Snapshot | VM | Purpose |
|---|---|---|
| `PRE-FIX-YYYY-MM-DD` | Each VM | Inspect the previous partial state, not reusable as a baseline |
| `MEEV-DC01-00-OS` | MEEV-DC01 | Clean Server installation and network |
| `MEEV-DC01-01-NET` | MEEV-DC01 | Static IP and DNS configured |
| `MEEV-DC01-02-ADDS-VERIFIED` | MEEV-DC01 | Forest and DNS verified |
| `MEEV-DC01-03-OBJECTS` | MEEV-DC01 | Lab OUs, groups, and attack-path identities created |
| `CA01-00-OS` | CA01 | Clean Server installation and network |
| `CA01-01-JOINED` | CA01 | Domain-joined member server, before AD CS |
| `CA01-02-ENTERPRISE-CA` | CA01 | Enterprise Root CA verified; baseline User template published |
| `MS01-00-OS` | MS01 | Clean workstation and network |
| `MS01-01-JOINED` | MS01 | Domain-joined and certificate enrollment verified |
| `ESI-01-BASELINE` | All | Clean, verified baseline for the attack-path build |
| `ESI-02-01-ANON-LDAP` | All | Through the anonymous LDAP stage |
| `ESI-02-02-ASREP` | All | Through the AS-REP stage |
| `ESI-02-03-KERBEROAST` | All | Through the Kerberoast stage |
| `ESI-02-04-LNK-SHARE` | All | Through the LNK/SMB share stage |
| `ESI-02-05-ESC1` | All | Through the ESC1 stage |
| `ESI-02-06-GENERICALL` | All | Through the GenericAll stage |
| `ESI-02-ATTACK-PATH` | All | Intentional attack-path weaknesses applied together; never expose |

### 6.5 Reproducibility notes

- Exporting a QEMU/KVM disk image after acceptance is optional. Store exports outside the repository; `.qcow2`, `.ova`, and `.vbox` files are ignored.
- Keep a copy of the exact ISO names and Windows build numbers in local notes.
- Record any deviation from this runbook in the build log and update `diagrams/architecture.md`.
- Do not make manual configuration changes without documenting them.
- Rebuild from the baseline snapshots rather than patching a half-configured CA.
- Use `virsh snapshot-create-as` or virt-manager, and keep a separate snapshot for the clean baseline and the intentionally vulnerable path.

---

## 7. Phase 5 — Target attack path and later ESC work

The target path is defined by `planing/attack-path.png`. Its exact build steps are in `planing/01-attack-path-build.md`. That sheet is authorized only for this isolated lab and intentionally creates weaknesses.

### 7.1 Target path summary

| Stage | Lab component | Intentional weakness |
|---|---|---|
| Anonymous LDAP user enumeration | `MEEV-DC01` | Anonymous read on the Users OU and permissive `dSHeuristics` |
| AS-REP roasting | `mike.ross` | Kerberos pre-authentication disabled |
| Kerberoasting | `svc-filesmgr` | Service principal name registered on a user account |
| Malicious LNK / SMB share | `MS01`, `svc-filesmgr` | Share write access and a user-triggered LNK task |
| Hash catch / Responder | `MS01`, `h.harper` | A scheduled task opens the latest LNK as Helpdesk Harper |
| Hash crack and MS01 login | `MS01`, `h.harper` | Weak lab password and local administrator rights |
| ESC1 | `CA01`, `h.harper` | Template allows requester-supplied subject and grants enrollment |
| GenericAll | `h.john`, `meev.admin` | `h.john` has Full Control over the MEEV Admin account |
| Dump hashes / pass-the-hash | `meev.admin`, `MEEV-DC01` | MEEV Admin is in `DnsAdmins` and `Remote Management Users` |

### 7.2 Rules

1. Start from `ESI-01-BASELINE`; take a separate `ESI-02-ATTACK-PATH` snapshot after the full path is built.
2. Change **one** security-relevant setting per stage and record the exact change.
3. Never modify built-in certificate templates. Create a copy named `MEEV-ESC1-User`.
4. Use only the dedicated identities in `01-attack-path-build.md`.
5. Record the template ACL, EKUs, subject/SAN settings, enrollment rights, CA flags, and relevant AD object permissions before and after each change.
6. Publish only the template needed for the current path. Unpublish or restore the snapshot after testing.
7. Capture evidence with the dedicated test identities on MS01. Do not use real identities.
8. Revert to `ESI-01-BASELINE` after documenting the result. Do not stack unrelated scenarios.
9. Keep Web Enrollment on the isolated libvirt network. Never expose it to a real network.
10. Put offensive validation commands in a separate, explicitly authorized runbook. This build runbook remains non-offensive.

### 7.3 Later ESC modules

The baseline remains reusable for other controlled ESC modules after the target path is documented and reverted.

| Module | Security property to study | Baseline component | Keep isolated |
|---|---|---|---|
| ESC1 | A template allows requester-controlled subject/SAN with client authentication and broad enrollment | Enterprise CA, published template copy, low-priv group | Yes |
| ESC2 | A template has Any Purpose or no EKU and is broadly enrollable | Enterprise CA, template copy | Yes |
| ESC3 | Enrollment agent rights or templates are misconfigured | Enterprise CA, template copies, test users | Yes |
| ESC4 | A low-privilege principal has write control over a template object | AD ACL baseline, template copy | Yes |
| ESC5 | CA or related AD object ACLs are too permissive | Enterprise CA, AD ACL baseline | Yes |
| ESC6 | The CA allows SANs to be supplied in requests | Enterprise CA, isolated CA setting | Yes |
| ESC7 | CA officer/manage-CA rights are too broad | Enterprise CA, role-separated test users | Yes |
| ESC8 | Web Enrollment exposes a relayable HTTP endpoint | CA Web Enrollment, MS01 | Yes |

For each module, create a short worksheet containing the starting snapshot, the exact configuration change, the verification evidence, the rollback step, and the defensive lesson. Do not build all modules at once. The target path is the first controlled build, not a license to leave multiple unrelated weaknesses active.

---

## 8. Troubleshooting

| Symptom | Likely cause | Safe action |
|---|---|---|
| `virsh net-list` shows no networks as a normal user | The system libvirt connection is not selected | Use `sudo virsh -c qemu:///system net-list --all` or virt-manager; do not assume the network is missing |
| `virbr10` is DOWN | All lab guests are shut down, or the network is not started | Start `ADCS-LAB`; a down bridge with no guests is harmless |
| Guests cannot reach each other | Wrong network, NAT attached, or firewall | Check each VM has one NIC on `ADCS-LAB`; verify IPs; keep Windows Firewall enabled |
| `nslookup` or domain join cannot find the domain | CA01/MS01 DNS is not MEEV-DC01, or DNS is broken | Set DNS to `10.10.10.10`; run `Resolve-DnsName`; run `dcdiag /test:dns` on MEEV-DC01 |
| `dcdiag /test:dns` reports `Forw` failure | The isolated lab has no internet or no forwarders | Confirm `Get-DnsServerForwarder` is empty; document the expected `Forw` failure. Do not add `8.8.8.8` |
| `Get-DnsServerForwarder` shows `8.8.8.8` | A public forwarder was configured earlier | Remove it with `Remove-DnsServerForwarder -IPAddress 8.8.8.8 -Force`; rerun the DNS test |
| Domain join access denied | Wrong credentials or time skew | Use `ESI-SBA-MEEV\Administrator`; check time; verify DNS and domain controller connectivity |
| `Get-ADDomain` is not recognized | AD PowerShell module not loaded | Run `Import-Module ActiveDirectory` on MEEV-DC01; use RSAT on member servers if needed |
| AD CS credential field shows `CA01\Administrator` | Local account selected, or CA01 is not domain-joined | Join the domain; click Change and use `ESI-SBA-MEEV\Administrator` or another Enterprise Admins account |
| Enterprise CA option is disabled | Not domain-joined, insufficient credentials, or DNS failure | Stop; fix domain membership, Enterprise Admins membership, and DNS |
| `certutil -cainfo` says Standalone | The CA was configured in the wrong mode | Rebuild the CA role from a pre-AD CS snapshot or clean installation |
| Certificate Services will not stay running | Incomplete configuration, database permission problem, or corrupt CA database | Check Event Viewer → Windows Logs → Application; confirm the configuration wizard completed; restore the pre-CA snapshot if needed |
| Web Enrollment returns 404 or 500 | IIS or the CertSrv virtual directory is missing or DNS is wrong | Confirm the Web Enrollment role service is installed; browse `http://localhost/certsrv`; check IIS; verify CA01 DNS |
| The User template is missing on MS01 | Template not published, permissions, stale policy, or CA not Enterprise | Publish it in `certsrv.msc`; run `gpupdate /force`; log off/on; confirm enroll permission |
| Certificate request fails with RPC unavailable | CA service, firewall, DNS, or network problem | Confirm `CertSvc` is running; verify CA01 name/IP; check Windows Firewall; use localhost on CA01 to isolate the cause |
| Time-related Kerberos errors | Workstation or member server time skew | Correct the time zone; on the member run `w32tm /resync`; verify DC time |
| Only MEEV-DC01 was restored from a snapshot | Machine account passwords on CA01/MS01 no longer match | Restore all VMs to a compatible baseline or rejoin CA01/MS01 |
| CA01 was renamed after CA configuration | Renaming a configured CA host is unsupported | Rebuild CA01; do not rename again after configuration |
| Web Enrollment is reachable from a real network | Lab isolation is broken | Stop testing, disconnect the NIC, restore the isolated libvirt design, and review the snapshots |

Do not disable Windows Firewall for the whole lab as a troubleshooting shortcut. Use scoped rules only if a specific service is blocked, and document the change.

---

## 9. Appendix — Quick reference card

### 9.1 Address plan

| System | IP | Mask | Gateway | DNS |
|---|---|---|---|---|
| Host Kali `virbr10` | `10.10.10.1` | `255.255.255.0` | — | Existing host DNS |
| MEEV-DC01 | `10.10.10.10` | `255.255.255.0` | Empty or `10.10.10.1` | `10.10.10.10` |
| CA01 | `10.10.10.20` | `255.255.255.0` | Empty or `10.10.10.1` | `10.10.10.10` |
| MS01 | `10.10.10.30` | `255.255.255.0` | Empty or `10.10.10.1` | `10.10.10.10` |

A gateway of `10.10.10.1` is acceptable only because it is the host-side address of the isolated `ADCS-LAB` bridge. It must not provide NAT or internet access. Confirm with `virsh net-dumpxml ADCS-LAB` and `ip -4 addr show virbr10`.

### 9.2 CA settings

| Setting | Value |
|---|---|
| Common name | `MEEV-ROOT-CA` |
| Distinguished name suffix | `DC=esi-sba-meev,DC=local` |
| Type | Enterprise Root CA |
| Key | RSA 2048 |
| Hash | SHA-256 |
| Validity | 10 years |
| Web Enrollment | Installed for the baseline (ESC8 and browser enrollment check); HTTP only inside the isolated lab |
| Credentials for configuration | `ESI-SBA-MEEV\Administrator` or equivalent Enterprise Admins member |

### 9.3 Minimum verification commands

MEEV-DC01:

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-Service NTDS,DNS,Netlogon,KDC
Resolve-DnsName meev-dc01.esi-sba-meev.local
nltest /dsgetdc:esi-sba-meev.local
dcdiag /test:dns /v
```

CA01:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain,DomainRole
Get-Service CertSvc
certutil -ping
certutil -cainfo
```

MS01:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain
Resolve-DnsName ca01.esi-sba-meev.local
nltest /dsgetdc:esi-sba-meev.local
```

### 9.4 Final rule

The lab is ready for ESC work only when the baseline is clean, documented, and reversible. If a CA or domain setting cannot be explained from this runbook, do not build a vulnerability on top of it.
