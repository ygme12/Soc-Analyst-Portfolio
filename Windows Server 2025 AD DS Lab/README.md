# Windows Server 2025 Lab: From Empty VM to Domain Controller with Storage Spaces

A practical guide to building a small Windows Server 2025 lab. It covers installation, initial configuration, Active Directory Domain Services (AD DS), DNS, users and groups, Kerberos ticket inspection, and a two-way mirrored Storage Spaces volume.

**Course:** Network Operating Systems Windows  
**Institution:** University of Dunaújváros  
**Instructor:** Tibor Újbányi, PhD

> [!IMPORTANT]
> **Academic lab only.** Names, IP addresses, and credentials are for an isolated practice environment. Never reuse lab passwords in production or commit real credentials, private keys, or sensitive screenshots to GitHub.

## Project status

This repository contains a reproducible lab guide. Complete the steps in your own virtual machine and add your own screenshots and validation output before describing the environment as completed. No screenshots or test results are represented here as evidence of execution.

## Learning objectives

- Install and configure Windows Server 2025.
- Set a static IPv4 address and configure DNS client settings.
- Install AD DS and DNS and create a new forest/domain.
- Validate domain-controller health, DNS service records, and FSMO role ownership.
- Create organizational units (OUs), users, and security groups.
- Inspect Kerberos tickets and service principal names (SPNs) from a domain-joined client.
- Create and validate a two-way mirrored Storage Spaces volume.
- Troubleshoot common installation, DNS, authentication, and storage issues.

## Contents

1. [Environment overview](#1-environment-overview)
2. [Requirements](#2-requirements)
3. [Create the virtual machine](#3-create-the-virtual-machine)
4. [Install Windows Server 2025](#4-install-windows-server-2025)
5. [Initial configuration](#5-initial-configuration)
6. [Install AD DS and DNS](#6-install-ad-ds-and-dns)
7. [Validate the domain controller](#7-validate-the-domain-controller)
8. [Create OUs, users, and groups](#8-create-ous-users-and-groups)
9. [Observe Kerberos authentication](#9-observe-kerberos-authentication)
10. [Create a Storage Spaces mirror](#10-create-a-storage-spaces-mirror)
11. [Troubleshooting](#11-troubleshooting)
12. [Evidence checklist](#12-evidence-checklist)
13. [Cleanup and references](#13-cleanup-and-references)

---

## 1. Environment overview

| Component | Lab value |
|---|---|
| VM name | <code>WS2025-LAB</code> |
| Operating system | Windows Server 2025 Standard, Desktop Experience |
| Server hostname | <code>DC1</code> |
| IPv4 address | <code>192.168.10.100/24</code> |
| DNS server | <code>192.168.10.100</code> |
| DNS domain | <code>lab.local</code> |
| NetBIOS domain | <code>LAB</code> |
| Test user | John Doe — <code>jdoe@lab.local</code> |
| Test security group | <code>IT</code> (Global, Security) |
| Storage pool | <code>LabPool</code> |
| Virtual disk | <code>MirrorVD</code> |
| Data volume | <code>M:\</code> — label <code>MirrorData</code> |

The lab uses one server as a domain controller, DNS server, and storage host. This is suitable for a small isolated exercise, not a resilient production design: a single domain controller is a single point of failure.

## 2. Requirements

- A Windows Server 2025 ISO from the official Microsoft Evaluation Center or another authorised source.
- A supported hypervisor, such as Hyper-V, VMware Fusion, or UTM.
- Suggested VM resources: 2 vCPUs, 4–8 GB RAM, and a 60–80 GB system disk.
- Two additional, empty virtual disks of approximately 10 GB each for Storage Spaces.
- Separate strong passwords for the local Administrator account and Directory Services Restore Mode (DSRM).
- A separate Windows client VM joined to the domain for the Kerberos section.

**Apple Silicon note:** use guest media compatible with the host architecture where available. x64 guests on Apple Silicon require emulation rather than native virtualisation and can be significantly slower. Check current support for your hypervisor and chosen Windows Server media.

## 3. Create the virtual machine

Recommended settings:

| Setting | Value |
|---|---|
| vCPU | 2 or more |
| RAM | 4–8 GB |
| System disk | 60–80 GB, dynamically allocated |
| Firmware | UEFI; Generation 2 on Hyper-V |
| Network | Internal, private, or host-only virtual network |

### Hyper-V

1. Open Hyper-V Manager and select **New → Virtual Machine**.
2. Name the VM <code>WS2025-LAB</code> and choose **Generation 2**.
3. Assign at least 4 GB of memory.
4. Connect it to an internal or private virtual switch.
5. Create a dynamically expanding 60–80 GB VHDX.
6. Attach the Windows Server ISO as the boot image.

### VMware Fusion or UTM

Create a new VM, choose the appropriate guest architecture, attach the ISO, configure UEFI where supported, and connect the VM to an isolated or host-only network. Menu names vary by product version.

**Why isolate the lab?** A domain controller should not conflict with DHCP or DNS on a home or campus network. If you temporarily use NAT to download updates, switch to the isolated lab network before configuring the static address and promoting the server.

## 4. Install Windows Server 2025

1. Boot the VM from the attached ISO.
2. Choose language, time/currency format, and keyboard layout.
3. Select **Install Windows Server**.
4. Choose **Windows Server 2025 Standard (Desktop Experience)** for a graphical interface. The non-Desktop Experience option is Server Core.
5. Accept the licence terms and choose **Custom** installation.
6. Select the empty virtual system disk and continue.
7. Allow setup to finish and the VM to restart. Ensure it boots from the virtual disk rather than restarting the installer.
8. Set a strong password for the built-in local Administrator account and sign in.
9. Install applicable guest integration tools for your hypervisor.

## 5. Initial configuration

Run PowerShell as Administrator. Check the actual network adapter name before using it in commands.

### 5.1 Confirm OS and hostname

~~~powershell
hostname
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, CsName
Get-NetAdapter
~~~

### 5.2 Rename the server

~~~powershell
Rename-Computer -NewName "DC1" -Restart
~~~

Sign in again after the restart.

### 5.3 Configure a static IPv4 address

The example assumes the adapter is named <code>Ethernet</code> and the isolated lab does not need a default gateway. Change the adapter name if necessary.

~~~powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "192.168.10.100" -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.10.100"
~~~

Only configure a default gateway if the lab network has a router. If the interface already has an IPv4 address, inspect the existing configuration before adding another address.

~~~powershell
ipconfig /all
Get-NetIPConfiguration
~~~

### 5.4 Set time zone and install updates

Correct time is important for Kerberos authentication.

~~~powershell
Get-TimeZone
Set-TimeZone -Id "Central Europe Standard Time"
~~~

Install available Windows updates using a supported method, then restart if required.

### 5.5 Explore management tools

Open Server Manager and locate **Local Server**, **All Servers**, and **Add Roles and Features**. Also open Event Viewer, Services, Computer Management, and PowerShell. Optionally take a clean VM checkpoint; a checkpoint is convenient but is not a backup.

## 6. Install AD DS and DNS

Installing the roles and promoting the server are separate steps.

### 6.1 Install the roles

**GUI:** Server Manager → **Manage → Add Roles and Features** → role-based installation → select the server → select **Active Directory Domain Services** and **DNS Server**. Include the requested management tools and complete the wizard.

**PowerShell:**

~~~powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
~~~

The roles are now installed, but the server is not yet a domain controller.

### 6.2 Create the forest and domain

In Server Manager, select the notification flag and choose **Promote this server to a domain controller**.

1. Select **Add a new forest** and enter <code>lab.local</code>.
2. Select the available forest and domain functional levels offered by your Windows Server installation.
3. Keep **DNS Server** and **Global Catalog** selected.
4. Set a separate DSRM password and store it securely outside this repository.
5. Review the DNS delegation warning. An isolated lab domain may not have a delegation.
6. Confirm NetBIOS name <code>LAB</code> and review the database, log, and SYSVOL paths.
7. Run the prerequisite check and resolve blocking errors.
8. Install and allow the server to restart.
9. Sign in using the domain Administrator account, for example <code>LAB\Administrator</code>.

PowerShell alternative:

~~~powershell
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDNS
~~~

Follow the prompts, including the secure prompt for the DSRM password. Do not put that password in a command or script.

## 7. Validate the domain controller

A successful restart alone does not prove that AD DS and DNS are healthy. Run from an elevated PowerShell session:

~~~powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController -Filter *
Get-DnsServerZone
dcdiag
nslookup -type=SRV _ldap._tcp.dc._msdcs.lab.local
netdom query fsmo
~~~

Verify that:

- <code>Get-ADDomain</code> and <code>Get-ADForest</code> return the expected <code>lab.local</code> information.
- <code>dcdiag</code> completes; investigate failures instead of assuming they are harmless.
- The SRV lookup returns the domain controller's service records, including <code>DC1.lab.local</code>.
- <code>netdom query fsmo</code> lists all five FSMO roles. In this single-DC lab, all five should be owned by <code>DC1</code>.
- DNS Manager shows the <code>lab.local</code> zone and relevant AD DNS records.
- Active Directory Sites and Services shows the default site and domain controller.

## 8. Create OUs, users, and groups

### 8.1 Create organizational units

In **Active Directory Users and Computers** (Server Manager → Tools), create these OUs under <code>lab.local</code>:

- <code>Students</code>
- <code>Workstations</code>
- <code>Servers</code>
- <code>LabUsers</code>

PowerShell alternative:

~~~powershell
$ouNames = "Students", "Workstations", "Servers", "LabUsers"

foreach ($ou in $ouNames) {
    New-ADOrganizationalUnit -Name $ou -Path "DC=lab,DC=local"
}
~~~

If rerunning the commands, check whether the OUs already exist first.

### 8.2 Create the test user

In Active Directory Users and Computers, open the <code>LabUsers</code> OU and select **New → User**. Use first name John, last name Doe, logon name <code>jdoe</code>, and UPN <code>jdoe@lab.local</code>. Set a temporary lab password and require a password change at the next logon where appropriate.

PowerShell alternative:

~~~powershell
$pw = Read-Host "Enter a temporary lab password" -AsSecureString

New-ADUser -Name "John Doe" `
  -GivenName "John" `
  -Surname "Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@lab.local" `
  -Path "OU=LabUsers,DC=lab,DC=local" `
  -AccountPassword $pw `
  -Enabled $true `
  -ChangePasswordAtLogon $true
~~~

Use a disposable lab password only. Do not save it in the script or repository.

### 8.3 Create the IT security group

Create a group named <code>IT</code> with **Global** scope and **Security** group type, then add <code>jdoe</code> as a member.

~~~powershell
New-ADGroup -Name "IT" `
  -SamAccountName "IT" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=LabUsers,DC=lab,DC=local"

Add-ADGroupMember -Identity "IT" -Members "jdoe"
~~~

### 8.4 Verify directory objects

~~~powershell
Get-ADUser jdoe | Select-Object DistinguishedName, ObjectGUID, SID
Get-ADGroupMember -Identity "IT"
Get-ADPrincipalGroupMembership -Identity "jdoe" |
  Select-Object Name, GroupScope, GroupCategory
Get-ADComputer -Filter * |
  Select-Object Name, DNSHostName, ObjectGUID, SID
~~~

**Concepts to understand**

- **UPN:** a logon name such as <code>jdoe@lab.local</code>.
- **sAMAccountName:** a traditional short logon name such as <code>jdoe</code>.
- **Distinguished Name (DN):** the object's directory location; it changes if the object is moved.
- **ObjectGUID:** an identifier that remains stable for the lifetime of that directory object.
- **SID:** the security identifier Windows uses when evaluating access tokens and permissions.
- **Security group:** a group that can be assigned permissions. Global scope is suitable for organising users in this single-domain exercise.

## 9. Observe Kerberos authentication

This section requires a separate Windows client joined to <code>lab.local</code>, configured to use <code>192.168.10.100</code> as its DNS server and able to reach the domain controller. Join the client to the domain with an authorised account, then sign in as <code>jdoe</code>.

### 9.1 Confirm the domain logon

Run on the domain-joined client:

~~~cmd
whoami
whoami /upn
nltest /dsgetdc:lab.local
echo %LOGONSERVER%
~~~

### 9.2 Inspect Kerberos tickets

~~~cmd
klist
klist tgt
~~~

After a successful domain logon, inspect the ticket-granting ticket (TGT). To observe a service ticket, access a shared folder on the domain controller for which the user has permission, then run <code>klist</code> again.

For a normal-user test, create a dedicated folder such as <code>C:\LabShare</code>, share it, and grant the test user appropriate share and NTFS permissions. Access it using a UNC path such as <code>\\DC1.lab.local\LabShare</code>. Avoid using the administrative <code>C$</code> share as the normal-user test because it requires administrative access.

A service ticket for a service such as <code>cifs/DC1.lab.local</code> may appear after the share is accessed.

### 9.3 Purge and reacquire tickets (lab only)

~~~cmd
klist purge
~~~

Access the authorised share again and inspect the tickets. Purging tickets can disrupt access until tickets are reacquired; do this only in the disposable lab.

### 9.4 Inspect SPNs and logs

On the domain controller:

~~~cmd
setspn -L DC1
setspn -Q cifs/DC1.lab.local
~~~

In Event Viewer, inspect **Windows Logs → Security** for relevant logon and Kerberos events. Available events depend on the audit policy and actions performed. Do not claim a particular event was generated unless you observed it.

**Concepts to explain**

- **Authentication** establishes identity; **authorisation** determines what that identity may access.
- Kerberos uses tickets to authenticate to domain services.
- A **TGT** is used to request service tickets; a **service ticket** is used to access a particular service.
- An **SPN** associates a service with its service account. Incorrect or duplicate SPNs can cause authentication problems.
- Windows can negotiate Kerberos or NTLM depending on the environment. Do not assume every successful logon used Kerberos without checking evidence.

## 10. Create a Storage Spaces mirror

> [!CAUTION]
> **Destructive operation:** creating a storage pool and virtual disk can erase or repurpose selected disks. Use only the two new, empty data disks for this lab. Verify disk numbers and sizes carefully. Never select the operating-system disk.

### 10.1 Attach two data disks

1. Shut down the VM.
2. Attach two new virtual data disks of approximately 10 GB each.
3. Start the VM and inspect:

~~~powershell
Get-Disk | Format-Table Number, FriendlyName, Size, PartitionStyle
Get-PhysicalDisk | Format-Table FriendlyName, SerialNumber, Size, CanPool
~~~

Confirm which disks are the new virtual data disks. Both should be eligible for pooling (<code>CanPool = True</code>). If not, investigate the disk state rather than forcing the operation.

### 10.2 Create the pool, virtual disk, and volume

**GUI:** Server Manager → **File and Storage Services → Storage Pools**.

1. Create a pool named <code>LabPool</code> using only the two new data disks.
2. Create a virtual disk named <code>MirrorVD</code>.
3. Choose **Mirror** resiliency and **Fixed** provisioning; request a conservative size such as 8 GB.
4. Create a volume, assign drive letter <code>M:</code>, format it as NTFS, and use label <code>MirrorData</code>.

PowerShell alternative:

~~~powershell
$candidateDisks = @(Get-PhysicalDisk -CanPool $true)

if ($candidateDisks.Count -ne 2) {
    throw "Expected exactly two poolable data disks. Inspect Get-PhysicalDisk before continuing."
}

$subsystem = Get-StorageSubSystem |
    Where-Object FriendlyName -Like "Windows Storage*" |
    Select-Object -First 1

if (-not $subsystem) {
    throw "Could not find the Windows Storage subsystem. Inspect Get-StorageSubSystem."
}

New-StoragePool -FriendlyName "LabPool" `
    -StorageSubSystemFriendlyName $subsystem.FriendlyName `
    -PhysicalDisks $candidateDisks

New-VirtualDisk -StoragePoolFriendlyName "LabPool" `
    -FriendlyName "MirrorVD" `
    -ResiliencySettingName Mirror `
    -ProvisioningType Fixed `
    -Size 8GB

$disk = Get-VirtualDisk -FriendlyName "MirrorVD" | Get-Disk
$disk | Initialize-Disk -PartitionStyle GPT

$disk | New-Partition -DriveLetter M -UseMaximumSize |
    Format-Volume -FileSystem NTFS `
        -NewFileSystemLabel "MirrorData" `
        -Confirm:$false
~~~

Review the selected disks and pool capacity before running destructive commands. If the requested virtual-disk size is rejected, inspect available capacity and reduce the size if appropriate.

### 10.3 Verify the volume

~~~powershell
Get-StoragePool -FriendlyName "LabPool" |
    Format-List FriendlyName, HealthStatus, OperationalStatus, Size

Get-VirtualDisk -FriendlyName "MirrorVD" |
    Format-List FriendlyName, ResiliencySettingName, HealthStatus, Size

Get-PhysicalDisk |
    Format-Table FriendlyName, Size, HealthStatus, OperationalStatus

Get-Volume -DriveLetter M |
    Format-List DriveLetter, FileSystem, FileSystemLabel, Size

"Storage Spaces mirror lab" | Out-File "M:\mirror-test.txt"
Get-Content "M:\mirror-test.txt"
~~~

Record the actual output. Check the mirror layout, NTFS volume label, and readable test data. Health states can vary during creation or repair, so investigate warnings rather than assuming the expected state.

**Why isn't usable capacity 20 GB?** A two-way mirror stores two copies of data. Two 10 GB disks therefore provide roughly the capacity of one disk for mirrored data, minus metadata and other overhead. Actual usable capacity depends on disk sizes and pool configuration.

### 10.4 Optional: simulate a disk failure

Only attempt this if the virtual disks are disposable and you are comfortable restoring the lab.

1. Confirm the test file exists and the virtual disk is healthy.
2. Shut down the VM and disconnect one data disk in the hypervisor. Do not delete it.
3. Boot and inspect pool, virtual-disk, and physical-disk health.
4. Attempt to read <code>M:\mirror-test.txt</code> from the surviving copy.
5. Shut down, reconnect the same disk, and inspect health again.
6. If the pool remains degraded, investigate the Storage Spaces repair workflow. Do not initialise or format the surviving pool member.

A mirror is **not a backup**: deletions and corruption can be mirrored. Two virtual disks stored on the same physical host drive also do not protect against failure of that host drive.

## 11. Troubleshooting

| Symptom | Checks |
|---|---|
| VM does not boot from ISO | Confirm the ISO is attached, boot order is correct, and guest architecture is compatible. |
| Installer starts again after restart | Ensure the VM boots from the virtual system disk rather than the ISO. |
| No system disk appears | Check virtual disk attachment and storage controller. |
| Network is unavailable | Check virtual switch, adapter state, IP configuration, and isolation. |
| Static IP command fails | Confirm adapter name and inspect existing addresses. |
| Domain promotion fails | Read the prerequisite error; check hostname, static IP, DNS, credentials, and time. |
| Client cannot locate the DC | Set client DNS to the lab DNS/DC address and verify SRV records. |
| Kerberos authentication fails | Check DNS, clock synchronisation, domain connectivity, and SPN configuration. |
| User cannot access a share | Verify share and NTFS permissions, group membership, and UNC path. |
| Data disk cannot be pooled | Confirm it is an intended data disk and is eligible for pooling. |
| Mirror creation fails | Inspect physical/pool capacity, disk eligibility, and resiliency/provisioning settings. |

## 12. Evidence checklist

After completing the lab, add a few genuine, legible screenshots or sanitised command outputs. Exclude passwords, recovery keys, tokens, personal data, and unrelated desktop information.

- [ ] Windows Server version and hostname.
- [ ] Static IP configuration.
- [ ] AD DS forest/domain details.
- [ ] Domain-controller diagnostic output, with any warnings explained.
- [ ] DNS SRV lookup and FSMO role ownership.
- [ ] OUs, test user, and group membership.
- [ ] Kerberos tickets before and after accessing the authorised share.
- [ ] Storage pool and virtual-disk resiliency settings.
- [ ] NTFS volume and readable test file.
- [ ] Optional disk-failure observation, if actually performed.

Add a short caption for each screenshot explaining the task and what the output demonstrates. Leave items unchecked until you have actually completed them.

## 13. Cleanup and references

Shut down the VM when finished. To rebuild, remove the disposable VM and its virtual disks or revert to a known clean checkpoint. Verify that no disk contains data you need before deleting it.

### Official documentation

- [Windows Server hardware requirements](https://learn.microsoft.com/windows-server/get-started/hardware-requirements)
- [Create a virtual machine in Hyper-V](https://learn.microsoft.com/windows-server/virtualization/hyper-v/get-started/create-a-virtual-machine-in-hyper-v)
- [Install Active Directory Domain Services](https://learn.microsoft.com/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-)
- [Kerberos authentication overview](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)
- [Storage Spaces overview](https://learn.microsoft.com/en-us/windows-server/storage/storage-spaces/overview)
- [UTM Windows guide](https://docs.getutm.app/guides/windows/)

---

*Prepared for the Network Operating Systems Windows course at the University of Dunaújváros. Instructor: Tibor Újbányi, PhD. This is an academic learning resource, not a production deployment guide.*
