# About

Hello, this is my way to outsource my memory ^^ Before using any code or command make sure you understand what you are doing ;)

# PowerCli / PowerShell

## Basics 
Connect / Disconnect to vCenter or ESXi
``` powershell
Connect-VIServer -Server x.x.x.x
Disconnect-VIServer
Disconnect-VIServer -server *
```

Skip confirmation and without out  put 
``` powershell
command -confirm:$false | out-null
```

Most of commands need to interact with powershell objects
``` powershell
$vmhost = get-vmhost "esxi.fqdn"
$cluster = Get-VMHost -Location toto
foreach ($vmhost in $cluster) {"commands on each ESXi => $vmhost"}
$VM= get-vm "my-vm"
$VMsProd = get-cluster -name "prod" | get-vm
```


## Operations on VMs

Add VM to DRS group
``` powershell
Get-DrsClusterGroup -Cluster $Cluster -Name $DRSvmGroupName | Set-DrsClusterGroup -Add -VM $VMsToAdd;
```

Create a Snapshot multiple VM per folder
``` powershell
Get-Folder test-folder | Get-VM | New-Snapshot -Name init_snap
```

Delete all Snapshot on multiple VMs
``` powershell
foreach ($VM in $VMs){get-vm $VM | Get-Snapshot | Remove-Snapshot -Confirm:$false}
```

Get VM restarted by HA
``` powershell
Get-VIEvent -type warning | Where {$_.FullFormattedMessage -match "restarted"} |select ObjectName,CreatedTime,FullFormattedMessage |sort CreatedTime -Descending |export-CSV .\HA_VMs.csv
```

Get VM by PortGroup
``` powershell
Get-VM | where { ($ | Get-NetworkAdapter | where {$.networkname -match "vmlab-vlan705"})}
```

remove VM network adatpters 
``` powershell
$VM | get-NetworkAdapter | remove-NetworkAdapter -confirm:$false | out-null
```

Convert VM to Template and Template to VM 
``` powershell
$VM = get-vm "my_super_VM"
Set-VM $VM -ToTemplate -confirm:$False

$Template = Get-Template -Name "my_super_VM"
Set-Template -Template $Template -ToVM | out-null		
```

upgrade VM HW version
``` powershell
foreach ($VM in $NewTemplates){get-vm $VM | Set-VM -Version v14 -Confirm:$false}
```

Start / Stop guest VMs
``` powershell
foreach ($VM in $NewTemplates){get-vm $VM | Stop-VMGuest -Confirm:$false}
get-vm $VM | Start-VM -Confirm:$false | out-null
```

## Operations on ESXi

Start / Stop SSH Service on ESXi
``` powershell
Get-VMHostService -VMHost "esx* "| ?{$_.Label -eq "SSH"} | Stop-VMHostService -Confirm:$false
Get-VMHostService -VMHost "esx*" | ?{$_.Label -eq "SSH"} | Start-VMHostService
```

Activate Shell and remove warning of SSH service on ESXi
``` powershell
$cluster = Get-VMHost -Location toto
foreach ($vmhost in $cluster) {Get-VMHostService -VMHost $vmhost | Where-Object {$.Key -eq "TSM"} | Set-VMHostService -policy "on" -Confirm:$false ; Get-VMHostService -VMHost $vmhost | Where-Object {$.Key -eq "TSM"} | Restart-VMHostService -Confirm:$false ; Get-VMHost $vmhost| Set-VmHostAdvancedConfiguration -Name UserVars.SuppressShellWarning -Value 1 ;}
```

Get Esx uuid
``` powershell
Get-VMHost | Select Name,@{n="HostUUID";e={$_.ExtensionData.hardware.systeminfo.uuid}}
```

Check Path selection Policy
``` powershell
$AllESXHosts = Get-VMHost | Where { ($.ConnectionState -eq "Connected") -or ($.ConnectionState -eq "Maintenance")} | Sort Name
Foreach ($esxhost in $AllESXHosts) { Get-VMhost $esxhost | Get-ScsiLun -LunType disk | Where { $_.MultipathPolicy -notlike "RoundRobin" } | Select CanonicalName,MultipathPolicy }
```

Set Path Selection Policy to Round Robin
```powershell
$AllESXHosts = Get-VMHost | Where { ($.ConnectionState -eq "Connected") -or ($.ConnectionState -eq "Maintenance")} | Sort Name
Foreach ($esxhost in $AllESXHosts) { Get-VMhost $esxhost | Get-ScsiLun -LunType disk | Where { $_.MultipathPolicy -notlike "RoundRobin" } | Set-ScsiLun -MultipathPolicy "RoundRobin" }
```

get Host name and host ID
```powershell
Get-VMHost | Select-Object Id,Name
```

Mounting the NFS datastore
```powershell
$IPNas = "1.1.1.1"
$DatastoreNFS = "DS-NFS"
$NFSPath = "/volume/nfs_path"
New-Datastore -Nfs -VMHost $vmhost -Name $DatastoreNFS -Path $NFSpath -NfsHost $IPNas
```

Set vSwitch0 MTU to 9000
```powershell
$vSwitch = "vSwitch0"
Get-VirtualSwitch -VMHost $vmhost -Name $vSwitch | Set-VirtualSwitch -mtu 9000 -Confirm:$false
```

Set vmk0 mtu to 9000 (sometimes ESXi disconnect for several minutes)
```powershell
Get-VMHost -name $vmhost | Get-VMHostNetworkAdapter -Name "vmk0" | Set-VMHostNetworkAdapter -mtu 9000 -Confirm:$false -ErrorAction Ignore 
```
Rename Management Network 
```powershell
Get-VMHost -name $vmhost | Get-VirtualPortGroup -Name "Management Network" | Set-VirtualPortGroup -Name $theNewName
```

Disable override Nic Teaming policy for mgmt vmkernel (follow vSwitch policy)
```powershell
$policy = Get-VirtualPortGroup -VMHost $vmhost -name "Management Network" | Get-NicTeamingPolicy;
$policy | Set-NicTeamingPolicy -InheritFailoverOrder $true
```

Create new PortGroup
```powershell
Get-VirtualSwitch -VMhost $vmhost -Name $vSwitch | New-VirtualPortGroup -Name $pgname -vlanID $pgvLan
```

Set vSwitch0 promiscuous mode at true 
```powershell
$vmhost | Get-VirtualSwitch -Name "vswitch0" | Get-SecurityPolicy | Set-SecurityPolicy -AllowPromiscuous $true
```

Update $vSwitch Nic Teaming to set vmnic0, vmnic1 active
```powershell
$NetworkCard2 = $vmhost | Get-VMHostNetworkAdapter -name vmnic1
Get-VirtualSwitch -VMHost $vmhost -Name $vSwitch | Add-VirtualSwitchPhysicalNetworkAdapter -VMHostPhysicalNic $NetworkCard2 -Confirm:$false
```

Create new vmkernel for vMotion $vmkp in $vSwitch with IP $vmkpvMotionIP 
```powershell
New-VMHostNetworkAdapter -VMHost $vmhost -PortGroup $vmkpvMotion -VirtualSwitch $vSwitch -IP $vmkpvMotionIP -SubnetMask $NetmaskvMotion -mtu 9000 -VMotionEnabled:$true

$PGvMotion = Get-VirtualPortgroup -VMHost $vmhost -Name $vmkpvMotion -standard

Set-VirtualPortGroup -VirtualPortGroup $PGvMotion -VlanId $vmkpvMotionvLan
```

Add vnics to DvSwitch 
```powershell
$NetworkCard3 = $vmhost | Get-VMHostNetworkAdapter -name vmnic2
$NetworkCard4 = $vmhost | Get-VMHostNetworkAdapter -name vmnic3

Get-VDSwitch -Name $DvSwitchName | Add-VDSwitchVMHost -VMHost $vmhost
Get-VDSwitch -Name $DvSwitchName | Add-VDSwitchPhysicalNetworkAdapter -VMHostNetworkAdapter $NetworkCard3 -Confirm:$false
Get-VDSwitch -Name $DvSwitchName | Add-VDSwitchPhysicalNetworkAdapter -VMHostNetworkAdapter $NetworkCard4 -Confirm:$false
```

Rename a Datastore 
```powershell
Get-VMHost $vmhost | get-datastore -name "datastore*" | set-datastore -name $DatastoreName 

```

Create NFS Datastore
```powershell
New-Datastore -Nfs -VMHost $vmhost -Name $DSnfsName -Path $DSnfsPath -NfsHost $vmkpNFSserverIP
```

## ESXi Custom Image Build witch latest patch

```powershell
add-esxsoftwaredepot C:\VMware-ESXi-depot.zip
add-esxsoftwaredepot C:\Esxi-patchs.zip
add-esxsoftwaredepot drivers.zip

Get-EsxImageProfile | ft Name

new-esximageprofile -Cloneprofile EsxImageProfile_added_above -Name EsxImageProfile_you_want


Set-ESXImageProfile EsxImageProfile_you_want -SoftwarePackage (Get-ESXSoftwarePackage -Newest )

Get-EsxImageProfile | ft Name 

export-esximageprofile -imageprofile EsxImageProfile_you_want -Filepath .\EsxImageProfile_you_want.iso -Exporttoiso
export-esximageprofile -imageprofile EsxImageProfile_you_want -Filepath .\EsxImageProfile_you_want.zip -Exporttobundle -NoSignatureCheck
```

## loops 

Wait until an ESXi is UP
```powershell
Write-Host("[i] Wainting until ESX reconnect ") -fore Green;
start-sleep -s 15

$n = 1
do{
	$ESXiFQDNconnected = (Get-VMHost -Name $vmhost).PowerState
	
	if ($ESXiFQDNconnected -ne "PoweredOn"){
		write-host "[i] $vmhost is not connected to vCenter please wait ! retry $n"
		start-sleep -s 3
	}
	$n = $n+1;
}
until($ESXiFQDNconnected -eq "PoweredOn")
```

# Ping subnet one line 

Using Powershell 
```powershell
$subnet="192.168.1."; for ($i=1; $i -lt 255; $i++){ if (Test-Connection -Count 1 -Quiet -ErrorAction SilentlyContinue $subnet$i){write-host "$subnet$i is used"} else {write-host "$subnet$i is available" -fore green} }
```

Using Bash 
```bash
SUBNET="192.168.1."; for i in {1..40};do if ping -c 1 -w 1 "$SUBNET$i" >/dev/null; then echo "$SUBNET$i alive"; else echo "$SUBNET$i dead"; fi ; done
```

# ESXi Cli

Get specific vib version 
```bash
ssh to host 
esxcli software vib list | grep nfnic
esxcli software vib list | grep nenic
```

Operation on Nics
```bash
> Get firmware version
esxcli network nic get -n vmnic

> list of all nics
esxcfg-nics -l

> get detail of the nics
ethtool -i vmnic4

> To list information on the current vswitches and DVS’
esxcfg-vswitch –l

> To list information on the current vmk’s
esxcfg-vmknic –l

> To add a vSwitch named vSwitch0 to the server
esxcfg-vswitch –a vSwitch0

> To delete a vSwitch named vSwitch0 from server
esxcfg-vswitch –d vSwitch0

> To set Jumbo Frames on a vswitch and underlying vmnics
esxcfg-vswitch –m 9000 vSwitch0

> To add a port-group named Service Console to vSwitch0
esxcfg-vswitch –A “Service Console” vSwitch0

> To remove a port-group named Service Console from vSwitch0
esxcfg-vswitch –D “Service Console” vSwitch0

> To set a port-group named Service Console to user VLAN 102 on vSwitch0
esxcfg-vswitch –p “Service Console” –v 102 vSwitch0

> To see list of vmnics and the properties (MTU, etc.)
esxcfg-nics –l

> To remove the vmnic0 uplink from the DVS named myN1kDVS
esxcfg-vswitch –Q vmnic0 –V [port#] myN1kDVS

> To add the vmnic0 uplink to the DVS named myN1kDVS
esxcfg-vswitch –P vmnic0 –V [port#] myN1kDVS

> To add vmnic0 as an uplink to vSwitch0
esxcfg-vswitch –L vmnic0 vSwitch0

> To delete a vmk from the port-group on vSwitch
esxcfg-vmknic –d –v xxx –s myN1kDVS

> To add a vmk to the ESX server:
esxcfg-vmknic –a –i x.x.x.x –n 255.255.255.0 –m 9000 “VMkernel”

> To add a vmk with IP x.x.x.x/24 to the vSwitch with Jumbo Frames
esxcfg-vmknic –a –i x.x.x.x –n 255.255.255.0 –m 9000 –s myN1kDVS

> To set the jumbo frames on a vmk on a ESX server:
esxcfg-vmknic –m 9000 –v xxx –s myN1kDVS
```

Test MTU 9000 
``` bash
vmkping -d -s 8972 x.x.x.x
```




# Linux 

## Bash

Dig
```bash
dig +short example.com @nameserver
dig -x XX.XX.XX.XX @8.8.8.8 +trace (reverse PTR)
```

## Fail2Ban

get fail2ban status of all jails
```bash
fail2ban-client status
```

get fail2ban status for particular one
```bash
fail2ban-client status nginx-http-auth
```

how to unbanip an ip
```bash
fail2ban-client set nginx-http-auth unbanip 111.111.111.111
```

## Firewall-cmd

```bash
systemctl start firewalld.service
systemctl enable firewalld.service
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all
firewall-cmd --list-all-zones

firewall-cmd --zone=home --change-interface=eth0
firewall-cmd --set-default-zone=home
firewall-cmd --permanent --zone=public --add-source=x.x.x.x/24
firewall-cmd --permanent --zone=public --add-source=x.x.x.y/32
firewall-cmd --permanent --zone=public --add-port=1-22/tcp
firewall-cmd --permanent --zone=public --add-port=1-514/udp
firewall-cmd --reload
```

## LVM

Extend Logical Volume
```bash
 #df -hT
 Filesystem              Type      Size  Used Avail Use% Mounted on
 /dev/mapper/centos-root ext4       17G  8.9G  7.0G  57% /
```

In case a new disk added please run the following command to rescans all controllers, channels and luns
```bash
 #echo "- - -" > /sys/class/scsi_host/host0/scan
```
In case the actual disk has been increased you need to list the devices
```bash
 #ls /sys/class/scsi_device/
 The output should be a list of device like the following
 1:0:0:0/ 2:0:0:0/
```

Apply the following command to rescan the devices
```bash
 #echo 1 > /sys/class/scsi_device/1\:0\:0\:0/device/rescan
 #echo 1 > /sys/class/scsi_device/2\:0\:0\:0/device/rescan
```

At this step verify if the OS see the new provisioned disk space using fdisk
```bash
 #fdisk -l
```

Now we prepare the provisioned disk space to be used with LVM.
In case the disk has been increased
```bash
#fdisk /dev/sda
Press p to print the partition table to identify the number of partitions. By default, there are 2: sda1 and  sda2.
Press n to create a new primary partition.
Press p for primary.
Press 3 for the partition number, depending on the output of the partition table print.
Press Enter two times. (sometimes you need to put the good value with the end cylinders of last partition +1)
Press t to change the system’s partition ID.
Press 3 to select the newly creation partition.
Type 8e to change the Hex Code of the partition for Linux LVM.
Press w to write the changes to the partition table.
```

In case new disk has been added
```bash
#fdisk /dev/sdb
Press p to print the partition table to identify the number of partitions. By default, there are 2: sda1 and  sda2.
Press n to create a new primary partition.
Press p for primary.
Press 1 for the partition number, depending on the output of the partition table print.
Press Enter two times. (sometimes you need to put the good value with the end cylinders of last partition +1)
Press t to change the system’s partition ID.
Press 1 to select the newly creation partition.
Type 8e to change the Hex Code of the partition for Linux LVM.
Press w to write the changes to the partition table.
Before adding the new patition to LVM we need the execute the following command to inform the OS od partition table changes
 #partprobe
The new partition is now created and ready to be added to LVM
 #pvcreate /dev/sda3
 Physical volume "/dev/sda3" successfully created
```

List volume groups
```bash
vgdisplay
```

Extend the Volume Group above
```bash
vgextend centos /dev/sda3
  > Volume group "centos" successfully extended
```

Using pvscan command should confirm the addition of the new PV to the Volume Group
```bash
pvscan

lvdisplay
``` 

Next we extend the Logical volume
```bash
lvextend /dev/centos/root /dev/sda3
```

Final Step is to resize the file system so that it ca take in account the additional disk space
```bash
resize2fs /dev/centos/root
df -h
```

## Iptables

-L : List iptables rules   
-n : without resolution dns 
-v : for all interfaces
```bash
iptables -nvL --line-numbers
```

add rule in specific line : -I la chaine '2' rule position -p protocole -s source -i interface -j action

```bash
iptables -I fw-input 2 -p all -s 1.1.1.0/16 -i eth4 -j ACCEPT
```

Redirection de port
```bash
iptables -A PREROUTING -t nat -i eth0 -p udp --dport 514 -j REDIRECT -d x.x.x.x --to-port 5514
iptables -A PREROUTING -t nat -i eth1 -p udp --dport 514 -j REDIRECT -d y.y.y.y --to-port 1514
```

List redirection de port
```bash
iptables -t nat --line-numbers -n -L
```

Del regle nat
```bash
iptables -t nat -D PREROUTING 1
```
## NTP

to list ALL NTP Peers
```bash
ntpq -pn
```

to Sync with specific server
```bash
ntpdate -d 37.187.122.11
```

Conf example
```
logconfig =syncevents +peerevents +sysevents +allclock
logfile /var/log/ntpd.log

driftfile /var/lib/ntp/drift

restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery

restrict 127.0.0.1
restrict -6 ::1

restrict 10.0.0.0 mask 255.0.0.0 nomodify notrap
restrict 4.3.2.0 mask 255.255.252.0 nomodify notrap

server 1.2.3.4
```

test NTP from Windows
```
w32tm /stripchart /computer:1.2.3.4
```

# Cisco

## UCS Fabric Interconnect 

Factory reset UCS Fabric Interconnect
```
connect local-mgmt
erase configuration
```

navigate on Cisco folder and show cluster details
```
UCS-A# show cluster state
or 
UCS-A# show cluster extended-state
or 
UCS-A# scope system
UCS-A /system # show managed-entity detail
```

Verifying the Ethernet Data Path
```
UCS-A /fabric-interconnect # connect nxos a
UCS-A(nxos)# show int br | grep -v down | wc -l
UCS-A(nxos)# show platform fwm info hw-stm | grep '1.' | wc –l
```

Verifying the Data Path for Fibre Channel End-Host Mode

```
UCS-A /fabric-interconnect # connect nxos a
UCS-A(nxos)# show npv flogi-table
```

Clear Counters
```
UCS-A# connect nxos b
UCS-B(nxos)# clear counters interface san-port-channel 101
UCS-B(nxos)# clear counters interface fc1/20
UCS-B(nxos)# clear counters interface fc1/21
```

## MDS

MDS remove port Channel
```
MDS# conf t
Enter configuration commands, one per line.  End with CNTL/Z.

MDS(config)# show interface port-channel X
port-channel10 is down 
...
MDS(config)# no interface port-channel X
port-channel X deleted and all its members disabled
please do the same operation on the switch at the other end of the port-channel

MDS(config)# interface fc1/13
MDS(config-if)# no switchport description
MDS(config-if)# exit

MDS(config)# interface fc1/14
MDS(config-if)# no switchport description
MDS(config-if)# exit

MDS(config)# vsan database
MDS(config-vsan-db)# vsan 1 interface fc1/13
MDS(config-vsan-db)# vsan 1 interface fc1/14
MDS(config-vsan-db)# end
```

Configuration SAN port Channel
```

MDS# show feature | i fport-channel-trunk
fport-channel-trunk   1         enabled
if not activated 
MDS# conf t
Enter configuration commands, one per line.  End with CNTL/Z.
MDS(config)# feature fport-channel-trunk


Port Channel creation 
MDS(config)# conf t
MDS(config)# interface port-channel 10
MDS(config-if)# channel mode active
MDS(config-if)# switchport trunk mode off
MDS(config-if)# switchport description SAN Port Channel to equipment


MDS(config-if)# interface fc1/13
MDS(config-if)#  switchport description equipment interface 1/1
MDS(config-if)#  channel-group 10 force
fc1/13 added to port-channel 10 and disabled
please do the same operation on the switch at the other end of the port-channel,
then do "no shutdown" at both ends to bring it up
MDS(config-if)#  no shutdown

MDS(config-if)# interface fc1/14
MDS(config-if)#  switchport description equipment
MDS(config-if)#  channel-group 10 force
fc1/14 added to port-channel 10 and disabled
please do the same operation on the switch at the other end of the port-channel,
then do "no shutdown" at both ends to bring it up
MDS(config-if)#  no shutdown
4/ Assigner le port-channel au vsan
MDS# conf t
MDS(config-if)# vsan database
MDS(config-vsan-db)# vsan 101 interface port-channel 10
MDS(config-vsan-db)# end
```

Show and Clear counters 
```
show interface fc1/45
show interface fc1/45 counters

clear counters interface all clear counters interface fc1/45

show log interface
```

When Enhanced zoning is enable configure a vSAN on one member... but it mandary to execute this command to save config on all members
```
copy running-config startup-config
```

Add new Zone on existing zoneset 
```
config t
 
fcalias name ESX_hba_a VSAN 11
member pwwn 10:00:00:xx:xx:xx:xx:xx
exit
 
zone name ESX_hba_a-DC1-3PAR-8200_0_0_1 VSAN 11
member fcalias ESX_hba_a
member fcalias DC1-3PAR-8200_0_0_1
exit
 
zone name ESX_hba_a-DC1-3PAR-8200_1_0_1 VSAN 11
member fcalias ESX_hba_a
member fcalias DC1-3PAR-8200_1_0_1
exit
 
zone name ESX_hba_a-DC2-3PAR-8200_0_0_1 VSAN 11
member fcalias ESX_hba_a
member fcalias DC2-3PAR-8200_0_0_1
exit
 
zone name ESX_hba_a-DC2-3PAR-8200_1_0_1 VSAN 11
member fcalias ESX_hba_a
member fcalias DC2-3PAR-8200_1_0_1
exit
 
zoneset name zoneset-fab1-3PAR-access VSAN 11
member ESX_hba_a-DC1-3PAR-8200_0_0_1
member ESX_hba_a-DC1-3PAR-8200_1_0_1
member ESX_hba_a-DC2-3PAR-8200_0_0_1
member ESX_hba_a-DC2-3PAR-8200_1_0_1
exit
 
zoneset activate name zoneset-fab1-3PAR-access VSAN 11
zone commit vsan 11
```


# Active Directory

Check the replication health
```
Repadmin /replsummary
```
Check the inbound replication requests that are queued
```
Repadmin /Queue
```
Check the replication status
```
Repadmin /Showrepl
```
Synchronize replication between replication partners
```
Repadmin /syncall
```
AD Get FSMO Roles
```
netdom query fsmo
```

# Network route

## Windows
```
route add -p 1.1.1.0 mask 255.255.255.0 2.2.2.2
route delete 1.1.1.0
```

## Linux
```
ip route add 192.168.1.0/24 dev eth0
route add default gw 192.168.1.254 eth0
ip route add 10.38.0.0/16 via 192.168.100.1
```

# GIT
```
Init new repo
git init

Access to Git repo
ssh git@git.web.com

Clone existing repo
git clone git@git.web.com

Creer une nouvelle branche
git checkout -b newbranch

Change branch
git checkout branch

Status
git status

Add all file to commit
git add --alll

Commit
git commit -m "message related to the commit"

Push
git push

Sync branch with master
git checkout master
git pull
git checkout brancheToMerge
git merge master

Save creadentials locally
git config credential.helper store

set global user mail
git config --global user.email "email@example.com"

set global user name
git config --global user.name "ben"

set only for local repo
git config user.email "email@example.com"

Merge using Command line
git fetch origin
git checkout -b Update-Repo-Structure origin/Update-Repo-Structure
git fetch origin
git checkout origin/master
git merge --no-ff Update-Repo-Structure

git push origin master
```


