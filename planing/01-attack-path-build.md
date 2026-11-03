# AD CS Attack-Path Lab — Controlled Attack-Path Build

**Status:** Build sheet for the target path; not an exploitation guide  
**Starting point:** `ESI-01-BASELINE` on `MEEV-DC01`, `CA01`, and `MS01`  
**Ending point:** `ESI-02-ATTACK-PATH` snapshot  
**Authorized environment:** the isolated libvirt network `ADCS-LAB` only

This sheet creates the weaknesses shown in `planing/attack-path.png`. It deliberately makes the lab insecure. Do not expose it, bridge it, or reuse these settings outside the isolated lab. Do not add offensive validation commands here; use a separate, explicitly authorized validation runbook.

## 1. Preconditions

- `planing/00-build-runbook.md` has been completed through `ESI-01-BASELINE`.
- The domain is `esi-sba-meev.local`; the DC is `MEEV-DC01`; the CA is `CA01` with CA name `MEEV-ROOT-CA`.
- `MS01` is domain-joined, and `h.harper` can enroll the clean **User** template.
- The identities `mike.ross`, `svc-filesmgr`, `h.harper`, `h.john`, and `meev.admin` exist.
- `Get-DnsServerForwarder` contains no public resolver such as `8.8.8.8`.
- The libvirt network is `ADCS-LAB` with no `<forward>` element or `<forward mode='none'/>`; no NAT or bridged adapter exists.
- All passwords are stored only in the local password manager.
- The clean baseline snapshot is available on all three VMs.

## 2. Identity map

| Path identity | Account | Intended role in the path | Build stage |
|---|---|---|---|
| Mike Ross | `ESI-SBA-MEEV\mike.ross` | AS-REP roast target | 4 |
| File Manager | `ESI-SBA-MEEV\svc-filesmgr` | Kerberoast target; SMB share writer | 5 and 6 |
| Helpdesk Harper | `ESI-SBA-MEEV\h.harper` | Phished user, local admin on MS01, ESC1 enrollee | 6 and 7 |
| Harry John | `ESI-SBA-MEEV\h.john` | ESC1 impersonation target; GenericAll holder | 7 and 8 |
| MEEV Admin | `ESI-SBA-MEEV\meev.admin` | GenericAll target; `DnsAdmins` and `Remote Management Users` member | 8 |

The original writeup calls the admin identity `Nebula Admin`; this lab uses `meev.admin`. The original share name `Nebula` is replaced by `MEEVShare`.

## 3. Stage 1 — Anonymous LDAP user enumeration

**Goal:** allow an unauthenticated LDAP client to enumerate domain users.

### 3.1 Record the starting state

On `MEEV-DC01`:

```powershell
$configNC = (Get-ADRootDSE).ConfigurationNamingContext
$dsObject = "CN=Directory Service,CN=Windows NT,CN=Services,$configNC"
Get-ADObject -Identity $dsObject -Properties dSHeuristics
```

Save the observed `dSHeuristics` value for rollback. If the attribute is absent, note that.

### 3.2 Enable anonymous reads

1. Open **Active Directory Users and Computers**.
2. Enable **View → Advanced Features**.
3. Navigate to `ESI-SBA-MEEV → Users`.
4. Open the **Users** OU properties → **Security**.
5. Add **ANONYMOUS LOGON**.
6. Grant **Read** and apply it to **This object and all descendant objects**.
7. If the special identity cannot be selected, run the equivalent `dsacls` command from an elevated prompt:

```powershell
dsacls "OU=Users,OU=ESI-SBA-MEEV,DC=esi-sba-meev,DC=local" /G "ANONYMOUS LOGON:GR"
```

### 3.3 Set the permissive `dSHeuristics` value

Use ADSI Edit if you prefer the GUI:

1. Open `adsiedit.msc`.
2. Connect to the **Configuration** naming context.
3. Navigate to `CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=esi-sba-meev,DC=local`.
4. Open the object's properties and set `dSHeuristics` to `0000002`.

PowerShell equivalent:

```powershell
$configNC = (Get-ADRootDSE).ConfigurationNamingContext
$dsObject = "CN=Directory Service,CN=Windows NT,CN=Services,$configNC"
Set-ADObject -Identity $dsObject -Replace @{dSHeuristics="0000002"}
```

### 3.4 Verify the weakness

From the Kali host:

```bash
ldapsearch -x -H ldap://10.10.10.10 \
  -b "OU=Users,OU=ESI-SBA-MEEV,dc=esi-sba-meev,dc=local" \
  "(objectClass=user)" sAMAccountName
```

Expected: the query returns domain accounts without credentials. This is verification only. If anonymous simple bind is refused, check the LDAP signing policy and record the result before changing anything else.

### 3.5 Checkpoint and rollback

- Snapshot: `ESI-02-01-ANON-LDAP`.
- Evidence: E08 anonymous-query transcript or sanitized screenshot.
- Rollback: remove the `ANONYMOUS LOGON` ACE from the Users OU and restore the original `dSHeuristics` value (or clear it if it was absent).

## 4. Stage 2 — AS-REP roastable Mike Ross

**Goal:** disable Kerberos pre-authentication for `mike.ross`.

On `MEEV-DC01`:

```powershell
Set-ADAccountControl -Identity "mike.ross" -DoesNotRequirePreAuth $true
Get-ADUser "mike.ross" -Properties DoesNotRequirePreAuth |
  Select-Object Name,SamAccountName,DoesNotRequirePreAuth
```

Expected: `DoesNotRequirePreAuth = True`.

- Snapshot: `ESI-02-02-ASREP`.
- Evidence: E08 property screenshot.
- Rollback: `Set-ADAccountControl -Identity "mike.ross" -DoesNotRequirePreAuth $false`.
- Do not put the AS-REP request command in this build sheet. Use the authorized validation runbook or the original writeup in the isolated lab.

## 5. Stage 3 — Kerberoastable File Manager

**Goal:** register an SPN on `svc-filesmgr`.

On `MEEV-DC01`:

```powershell
setspn -A MEEV-DC01/svc-filesmgr.esi-sba-meev.local ESI-SBA-MEEV\svc-filesmgr
setspn -L ESI-SBA-MEEV\svc-filesmgr
```

Expected: the SPN list contains `MEEV-DC01/svc-filesmgr.esi-sba-meev.local`.

- Snapshot: `ESI-02-03-KERBEROAST`.
- Evidence: E08 SPN output.
- Rollback: `setspn -D MEEV-DC01/svc-filesmgr.esi-sba-meev.local ESI-SBA-MEEV\svc-filesmgr`.
- Do not put the Kerberoast request command in this build sheet. Use the authorized validation runbook.

## 6. Stage 4 — SMB share, LNK trigger, and Helpdesk Harper

**Goal:** create a share that `svc-filesmgr` can write to and a scheduled task that opens the newest LNK as `h.harper`.

### 6.1 Prepare MS01

On `MS01`, as a local or domain administrator:

```powershell
Add-LocalGroupMember -Group "Administrators" -Member "ESI-SBA-MEEV\h.harper"

New-Item -ItemType Directory -Path "C:\MEEVShare" -Force
New-Item -ItemType Directory -Path "C:\ProgramData\LNKTrigger" -Force

New-SmbShare -Name "MEEVShare" -Path "C:\MEEVShare" `
  -ChangeAccess "ESI-SBA-MEEV\svc-filesmgr" `
  -ReadAccess "ESI-SBA-MEEV\h.harper"

icacls "C:\MEEVShare" /grant "ESI-SBA-MEEV\svc-filesmgr:(OI)(CI)M"
icacls "C:\MEEVShare" /grant "ESI-SBA-MEEV\h.harper:(OI)(CI)R"
icacls "C:\ProgramData\LNKTrigger" /grant "ESI-SBA-MEEV\h.harper:(OI)(CI)RX"
```

This intentionally grants `h.harper` local administrator rights and gives `svc-filesmgr` write access to the share. Neither is a baseline setting.

### 6.2 Create the trigger script

Create `C:\ProgramData\LNKTrigger\lnk-trigger.ps1`:

```powershell
$share = "C:\MEEVShare"

while ($true) {
    $lnk = Get-ChildItem -Path $share -Filter *.lnk -File -ErrorAction SilentlyContinue |
        Sort-Object LastWriteTime -Descending |
        Select-Object -First 1

    if ($null -ne $lnk) {
        Start-Process -FilePath $lnk.FullName
        Start-Sleep -Seconds 5
        Remove-Item -Path $lnk.FullName -Force -ErrorAction SilentlyContinue
    }

    Start-Sleep -Seconds 60
}
```

The script does not create the LNK. The attacker writes it to the share after Kerberoasting `svc-filesmgr`; the task simulates a logged-in Helpdesk user opening the newest file.

### 6.3 Create the scheduled task

Use Task Scheduler or an elevated prompt:

1. Create a task named `LNK TRIGGER`.
2. Run it as `ESI-SBA-MEEV\h.harper`.
3. Select **Run whether user is logged on or not** and store the password.
4. Select **Run with highest privileges**.
5. Trigger: **At startup**.
6. Action: start `powershell.exe` with:

```text
-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File C:\ProgramData\LNKTrigger\lnk-trigger.ps1
```

Command-line alternative (it prompts for the account password):

```text
schtasks /Create /TN "LNK TRIGGER" /SC ONSTART /RU "ESI-SBA-MEEV\h.harper" /RP * /RL HIGHEST /TR "powershell.exe -NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File C:\ProgramData\LNKTrigger\lnk-trigger.ps1"
```

Do not put the password in the command or the repository.

If the task does not resolve the LNK in a non-interactive session, change the trigger to **At log on** and keep `h.harper` logged on to MS01 during the authorized validation. Do not switch the task to an S4U logon type; S4U cannot authenticate to the remote SMB listener.

### 6.4 Allow the lab traffic

Confirm that File and Printer Sharing is allowed on the domain profile and that `MS01` is reachable from the Kali host on TCP 445. Keep Windows Firewall enabled; use a scoped rule only if the domain profile blocks the lab network.

### 6.5 Verify

```powershell
Get-SmbShare -Name "MEEVShare"
Get-SmbShareAccess -Name "MEEVShare"
Get-LocalGroupMember -Group "Administrators" | Where-Object Name -match "h.harper"
Get-ScheduledTask -TaskName "LNK TRIGGER" | Get-ScheduledTaskInfo
(Get-ScheduledTask -TaskName "LNK TRIGGER").Principal | Format-List UserId,LogonType,RunLevel
Start-ScheduledTask -TaskName "LNK TRIGGER"
```

Expected: the share exists, `svc-filesmgr` has Change and `h.harper` has Read, `h.harper` is a local administrator, and the task exists as `h.harper` with highest privileges.

- Snapshot: `ESI-02-04-LNK-SHARE`.
- Evidence: E08 share ACL, local Administrators membership, and task configuration.
- Rollback: unregister `LNK TRIGGER`, remove `C:\ProgramData\LNKTrigger`, remove `MEEVShare` and `C:\MEEVShare`, and remove `h.harper` from local Administrators.
- Do not create the malicious LNK or run Responder in this build sheet.

## 7. Stage 5 — ESC1 certificate template

**Goal:** publish a template that allows `h.harper` to supply an arbitrary subject and authenticate as another user.

### 7.1 Create the template copy

On `CA01`:

1. Open `certtmpl.msc`.
2. Duplicate the built-in **User** template.
3. Name it `MEEV-ESC1-User`.
4. On **General**, set the validity to a lab-appropriate period. Long validity is not required for ESC1.
5. On **Subject Name**, select **Supply in the request**.
6. Confirm the **Client Authentication** EKU is present.
7. On **Security**, remove broad enrollment if present and add `ESI-SBA-MEEV\GG-ESC1-Enrollees` with **Enroll** and **Read**.
8. Do not modify the built-in **User** template.

### 7.2 Publish the template

1. Open `certsrv.msc` on `CA01`.
2. Right-click **Certificate Templates → New → Certificate Template to Issue**.
3. Select `MEEV-ESC1-User`.

Verify:

```powershell
certutil -CATemplates | findstr "MEEV-ESC1"
```

On `MS01`, run `gpupdate /force` and confirm `h.harper` can see the template in `certmgr.msc`. Do not actually request a certificate for another identity in this build sheet.

- Snapshot: `ESI-02-05-ESC1`.
- Evidence: E08 template subject setting, EKU, ACL, and published-template list.
- Rollback: unpublish the template and delete `MEEV-ESC1-User`; or restore the pre-stage snapshot.
- Do not set the CA-wide `EDITF_ATTRIBUTESUBJECTALTNAME2` flag. That is ESC6, not ESC1.

## 8. Stage 6 — GenericAll and MEEV Admin group membership

**Goal:** give `h.john` full control over `meev.admin`, then give `meev.admin` the group memberships needed for the final DC path.

### 8.1 Grant GenericAll

On `MEEV-DC01`:

1. Open **Active Directory Users and Computers** with **Advanced Features** enabled.
2. Open `meev.admin` → **Security**.
3. Add `ESI-SBA-MEEV\h.john`.
4. Grant **Full Control**.
5. Confirm `meev.admin` is enabled.

Verify:

```powershell
$dn = (Get-ADUser "meev.admin").DistinguishedName
(Get-Acl "AD:$dn").Access |
  Where-Object IdentityReference -match "h.john" |
  Select-Object IdentityReference,ActiveDirectoryRights,AccessControlType
```

Expected: `h.john` has `GenericAll` or equivalent Full Control.

### 8.2 Add MEEV Admin to the final-path groups

```powershell
Add-ADGroupMember -Identity "DnsAdmins" -Members "meev.admin"
Add-ADGroupMember -Identity "Remote Management Users" -Members "meev.admin"
Get-ADGroupMember "DnsAdmins" | Select-Object Name,SamAccountName
Get-ADGroupMember "Remote Management Users" | Select-Object Name,SamAccountName
```

This intentionally gives the MEEV Admin account DNS service and WinRM access. Do not add `meev.admin` to Domain Admins or Enterprise Admins.

- Snapshot: `ESI-02-06-GENERICALL`.
- Evidence: E08 ACL and group membership.
- Rollback: remove the `h.john` Full Control ACE and remove `meev.admin` from both groups.

## 9. Final path checkpoint

After all stages are verified:

1. Confirm each weakness independently.
2. Take one snapshot per VM named `ESI-02-ATTACK-PATH`.
3. Record the start/end times, snapshots, and E08 evidence in the build log.
4. Do not merge the attack-path snapshot with the clean baseline.
5. Revert to `ESI-01-BASELINE` after the authorized validation is complete.

## 10. Original writeup name mapping

Use this mapping when following the original walkthrough for the offensive validation phase:

| Original | This lab |
|---|---|
| `nebula.local` | `esi-sba-meev.local` |
| `NEBULA-DC` | `MEEV-DC01` |
| `NEBULA\mike.ross` | `ESI-SBA-MEEV\mike.ross` |
| `NEBULA\svc-filesmgr` | `ESI-SBA-MEEV\svc-filesmgr` |
| `NEBULA\helpdesk` | `ESI-SBA-MEEV\h.harper` |
| `NEBULA\h.john` | `ESI-SBA-MEEV\h.john` |
| `NEBULA\n.admin` | `ESI-SBA-MEEV\meev.admin` |
| `Nebula` share | `MEEVShare` |
| `Nebula-CA` | `MEEV-ROOT-CA` |
| `MS01` | `MS01` |

Do not run the original commands against any system outside this isolated lab. The original writeup may contain commands that assume different names, paths, or tools; verify each prerequisite instead of assuming the command will work unchanged.

## 11. Evidence and rollback summary

| Stage | Snapshot | Primary evidence | Rollback |
|---|---|---|---|
| Anonymous LDAP | `ESI-02-01-ANON-LDAP` | Anonymous query result | Remove OU ACE; restore `dSHeuristics` |
| AS-REP | `ESI-02-02-ASREP` | `DoesNotRequirePreAuth = True` | Set pre-auth back to true |
| Kerberoast | `ESI-02-03-KERBEROAST` | SPN list | Delete the SPN |
| LNK/SMB | `ESI-02-04-LNK-SHARE` | Share ACL, local admin, task | Remove task, share, folder, admin membership |
| ESC1 | `ESI-02-05-ESC1` | Template subject/EKU/ACL/published | Unpublish and delete template copy |
| GenericAll | `ESI-02-06-GENERICALL` | ACL and group memberships | Remove ACE and memberships |
| Full path | `ESI-02-ATTACK-PATH` | E08 package | Restore `ESI-01-BASELINE` |

The build is complete when the weaknesses are verified, the snapshots exist, and the clean baseline is still intact.
