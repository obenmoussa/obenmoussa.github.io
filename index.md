# PowerCli / PowerShell

## Basics 
Connect / Disconnect to vCenter or ESXi
```powershell
Connect-VIServer -Server x.x.x.x
Disconnect-VIServer
Disconnect-VIServer -server *
```

Skip confirmation and without out  put 
~~~~
command -confirm:$false | out-null
~~~~

Most of commands need to interact with powershell objects
~~~~
$vmhost = get-vmhost "esxi.fqdn"
$cluster = Get-VMHost -Location toto
foreach ($vmhost in $cluster) {"commands on each ESXi => $vmhost"}
$VM= get-vm "my-vm"
$VMsProd = get-cluster -name "prod" | get-vm
~~~~


## Operations on VMs

Add VM to DRS group
~~~~
Get-DrsClusterGroup -Cluster $Cluster -Name $DRSvmGroupName | Set-DrsClusterGroup -Add -VM $VMsToAdd;
~~~~

Create a Snapshot multiple VM per folder
~~~~
Get-Folder test-folder | Get-VM | New-Snapshot -Name init_snap
~~~~

Delete all Snapshot on multiple VMs
~~~~
foreach ($VM in $VMs){get-vm $VM | Get-Snapshot | Remove-Snapshot -Confirm:$false}
~~~~

Get VM restarted by HA
~~~~
Get-VIEvent -type warning | Where {$_.FullFormattedMessage -match "restarted"} |select ObjectName,CreatedTime,FullFormattedMessage |sort CreatedTime -Descending |export-CSV .\HA_VMs.csv
~~~~

Get VM by PortGroup
~~~~
Get-VM | where { ($ | Get-NetworkAdapter | where {$.networkname -match "vmlab-vlan705"})}
~~~~

remove VM network adatpters 
~~~~
$VM | get-NetworkAdapter | remove-NetworkAdapter -confirm:$false | out-null
~~~~

Convert VM to Template and Template to VM 
~~~~
$VM = get-vm "my_super_VM"
Set-VM $VM -ToTemplate -confirm:$False

$Template = Get-Template -Name "my_super_VM"
Set-Template -Template $Template -ToVM | out-null		
~~~~

upgrade VM HW version
~~~~
foreach ($VM in $NewTemplates){get-vm $VM | Set-VM -Version v14 -Confirm:$false}
~~~~

Start / Stop guest VMs
~~~~
foreach ($VM in $NewTemplates){get-vm $VM | Stop-VMGuest -Confirm:$false}
get-vm $VM | Start-VM -Confirm:$false | out-null
~~~~

## Operations on ESXi

Start / Stop SSH Service on ESXi
~~~~
Get-VMHostService -VMHost "esx* "| ?{$_.Label -eq "SSH"} | Stop-VMHostService -Confirm:$false
Get-VMHostService -VMHost "esx*" | ?{$_.Label -eq "SSH"} | Start-VMHostService
~~~~

Activate Shell and remove warning of SSH service on ESXi
~~~~
$cluster = Get-VMHost -Location toto
foreach ($vmhost in $cluster) {Get-VMHostService -VMHost $vmhost | Where-Object {$.Key -eq "TSM"} | Set-VMHostService -policy "on" -Confirm:$false ; Get-VMHostService -VMHost $vmhost | Where-Object {$.Key -eq "TSM"} | Restart-VMHostService -Confirm:$false ; Get-VMHost $vmhost| Set-VmHostAdvancedConfiguration -Name UserVars.SuppressShellWarning -Value 1 ;}
~~~~

Get Esx uuid
~~~~
Get-VMHost | Select Name,@{n="HostUUID";e={$_.ExtensionData.hardware.systeminfo.uuid}}
~~~~

Check Path selection Policy
~~~~
$AllESXHosts = Get-VMHost | Where { ($.ConnectionState -eq "Connected") -or ($.ConnectionState -eq "Maintenance")} | Sort Name
Foreach ($esxhost in $AllESXHosts) { Get-VMhost $esxhost | Get-ScsiLun -LunType disk | Where { $_.MultipathPolicy -notlike "RoundRobin" } | Select CanonicalName,MultipathPolicy }
~~~~

Set Path Selection Policy to Round Robin
~~~~
$AllESXHosts = Get-VMHost | Where { ($.ConnectionState -eq "Connected") -or ($.ConnectionState -eq "Maintenance")} | Sort Name
Foreach ($esxhost in $AllESXHosts) { Get-VMhost $esxhost | Get-ScsiLun -LunType disk | Where { $_.MultipathPolicy -notlike "RoundRobin" } | Set-ScsiLun -MultipathPolicy "RoundRobin" }
~~~~

get Host name and host ID
~~~~
Get-VMHost | Select-Object Id,Name
~~~~

Mounting the NFS datastore
~~~~
$IPNas = "1.1.1.1"
$DatastoreNFS = "DS-NFS"
$NFSPath = "/volume/nfs_path"
New-Datastore -Nfs -VMHost $vmhost -Name $DatastoreNFS -Path $NFSpath -NfsHost $IPNas
~~~~

Set vSwitch0 MTU to 9000
~~~~
$vSwitch = "vSwitch0"
Get-VirtualSwitch -VMHost $vmhost -Name $vSwitch | Set-VirtualSwitch -mtu 9000 -Confirm:$false
~~~~

Set vmk0 mtu to 9000 (sometimes ESXi disconnect for several minutes)
~~~~
Get-VMHost -name $vmhost | Get-VMHostNetworkAdapter -Name "vmk0" | Set-VMHostNetworkAdapter -mtu 9000 -Confirm:$false -ErrorAction Ignore 
~~~~
Rename Management Network 
~~~~
Get-VMHost -name $vmhost | Get-VirtualPortGroup -Name "Management Network" | Set-VirtualPortGroup -Name $theNewName
~~~~

Disable override Nic Teaming policy for mgmt vmkernel (follow vSwitch policy)
~~~~
$policy = Get-VirtualPortGroup -VMHost $vmhost -name "Management Network" | Get-NicTeamingPolicy;
$policy | Set-NicTeamingPolicy -InheritFailoverOrder $true
~~~~

Create new PortGroup
~~~~
Get-VirtualSwitch -VMhost $vmhost -Name $vSwitch | New-VirtualPortGroup -Name $pgname -vlanID $pgvLan
~~~~

Set vSwitch0 promiscuous mode at true 
~~~~
$vmhost | Get-VirtualSwitch -Name "vswitch0" | Get-SecurityPolicy | Set-SecurityPolicy -AllowPromiscuous $true
~~~~

Update $vSwitch Nic Teaming to set vmnic0, vmnic1 active
~~~~
$NetworkCard2 = $vmhost | Get-VMHostNetworkAdapter -name vmnic1
Get-VirtualSwitch -VMHost $vmhost -Name $vSwitch | Add-VirtualSwitchPhysicalNetworkAdapter -VMHostPhysicalNic $NetworkCard2 -Confirm:$false
~~~~

Create new vmkernel for vMotion $vmkp in $vSwitch with IP $vmkpvMotionIP 
~~~~
New-VMHostNetworkAdapter -VMHost $vmhost -PortGroup $vmkpvMotion -VirtualSwitch $vSwitch -IP $vmkpvMotionIP -SubnetMask $NetmaskvMotion -mtu 9000 -VMotionEnabled:$true

$PGvMotion = Get-VirtualPortgroup -VMHost $vmhost -Name $vmkpvMotion -standard

Set-VirtualPortGroup -VirtualPortGroup $PGvMotion -VlanId $vmkpvMotionvLan
~~~~

Add vnics to DvSwitch 
~~~~
$NetworkCard3 = $vmhost | Get-VMHostNetworkAdapter -name vmnic2
$NetworkCard4 = $vmhost | Get-VMHostNetworkAdapter -name vmnic3

Get-VDSwitch -Name $DvSwitchName | Add-VDSwitchVMHost -VMHost $vmhost
Get-VDSwitch -Name $DvSwitchName | Add-VDSwitchPhysicalNetworkAdapter -VMHostNetworkAdapter $NetworkCard3 -Confirm:$false
Get-VDSwitch -Name $DvSwitchName | Add-VDSwitchPhysicalNetworkAdapter -VMHostNetworkAdapter $NetworkCard4 -Confirm:$false
~~~~

Rename a Datastore 
~~~~
Get-VMHost $vmhost | get-datastore -name "datastore*" | set-datastore -name $DatastoreName 

~~~~

Create NFS Datastore
~~~~
New-Datastore -Nfs -VMHost $vmhost -Name $DSnfsName -Path $DSnfsPath -NfsHost $vmkpNFSserverIP
~~~~

## loops 

Wait until an ESXi is UP
~~~~
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
~~~~

# Ping subnet one line 

~~~~
$subnet="192.168.1."; for ($i=1; $i -lt 255; $i++){ if (Test-Connection -Count 1 -Quiet -ErrorAction SilentlyContinue $subnet$i){write-host "$subnet$i is used"} else {write-host "$subnet$i is available" -fore green} }
~~~~

# ESXi Cli

Get specific vib version 
~~~~
ssh to host 
esxcli software vib list | grep nfnic
esxcli software vib list | grep nenic
~~~~

Operation on Nics
~~~~
Get firmware version
esxcli network nic get -n vmnic
list of all nics
esxcfg-nics -l
get detail of the nics
ethtool -i vmnic4
~~~~

# Linux 
## Fail2Ban

get fail2ban status of all jails
~~~~
fail2ban-client status
~~~~

get fail2ban status for particular one
~~~~
fail2ban-client status nginx-http-auth
~~~~

how to unbanip an ip
~~~~
fail2ban-client set nginx-http-auth unbanip 111.111.111.111
~~~~

## Firewall-cmd

~~~~
starting firewall ( enable au lancement du sys)
systemctl start firewalld.service
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all
firewall-cmd --list-all-zones
assign interface to zone
firewall-cmd --zone=home --change-interface=eth0
firewall-cmd --set-default-zone=home
firewall-cmd --permanent --zone=public --add-source=192.168.100.0/24
firewall-cmd --permanent --zone=public --add-source=192.168.123.123/32
firewall-cmd --permanent --zone=public --add-port=1-22/tcp
firewall-cmd --permanent --zone=public --add-port=1-514/udp
firewall-cmd --reload
~~~~

## LVM

Extend Logical Volume
~~~~
 #df -hT
 Filesystem              Type      Size  Used Avail Use% Mounted on
 /dev/mapper/centos-root ext4       17G  8.9G  7.0G  57% /
~~~~

In case a new disk added please run the following command to rescans all controllers, channels and luns
~~~~
 #echo "- - -" > /sys/class/scsi_host/host0/scan
~~~~
In case the actual disk has been increased you need to list the devices
~~~~
 #ls /sys/class/scsi_device/
 The output should be a list of device like the following
 1:0:0:0/ 2:0:0:0/
~~~~

Apply the following command to rescan the devices
~~~~
 #echo 1 > /sys/class/scsi_device/1\:0\:0\:0/device/rescan
 #echo 1 > /sys/class/scsi_device/2\:0\:0\:0/device/rescan
~~~~

At this step verify if the OS see the new provisioned disk space using fdisk
~~~~
 #fdisk -l
~~~~

Now we prepare the provisioned disk space to be used with LVM.
In case the disk has been increased
~~~~
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
~~~~

In case new disk has been added
~~~~
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
~~~~

List volume groups
~~~~
 #vgdisplay
~~~~

Extend the Volume Group above
~~~~
 #vgextend centos /dev/sda3
  Volume group "centos" successfully extended
~~~~

Using pvscan command should confirm the addition of the new PV to the Volume Group
~~~~
 #pvscan

 #lvdisplay
~~~~ 

Next we extend the Logical volume
~~~~
 #lvextend /dev/centos/root /dev/sda3
~~~~

Final Step is to resize the file system so that it ca take in account the additional disk space
~~~~
 #resize2fs /dev/centos/root
 #df -h
~~~~

## Iptables

Lister des regles iptables : -L lister des regles -n sans resolution dns -v pour les interfaces
~~~~
iptables -nvL --line-numbers
~~~~

Ajout d'une regle dans une ligne précise : -I la chaine '2' pour la position de la regle -p protocole -s source -i interface -j

~~~~
iptables -I RH-Firewall-1-INPUT 2 -p all -s 1.1.1.0/16 -i eth4 -j ACCEPT
~~~~

Redirection de port
~~~~
iptables -A PREROUTING -t nat -i eth0 -p udp --dport 514 -j REDIRECT -d 10.1.3.74 --to-port 5514
iptables -A PREROUTING -t nat -i eth1 -p udp --dport 514 -j REDIRECT -d 10.1.3.202 --to-port 1514
~~~~

List redirection de port
~~~~
iptables -t nat --line-numbers -n -L
~~~~

Del regle nat
~~~~
iptables -t nat -D PREROUTING 1
~~~~


# Cisco
## UCS Fabric Interconnect 

Factory reset UCS Fabric Interconnect
~~~~
connect local-mgmt
erase configuration
~~~~

navigate on Cisco folder and show cluster details
~~~~
UCS-A# show cluster state
or 
UCS-A# show cluster extended-state
or 
UCS-A# scope system
UCS-A /system # show managed-entity detail
~~~~

Verifying the Ethernet Data Path
~~~~
UCS-A /fabric-interconnect # connect nxos a
UCS-A(nxos)# show int br | grep -v down | wc -l
UCS-A(nxos)# show platform fwm info hw-stm | grep '1.' | wc –l
~~~~

Verifying the Data Path for Fibre Channel End-Host Mode

~~~~
UCS-A /fabric-interconnect # connect nxos a
UCS-A(nxos)# show npv flogi-table
~~~~


Clear Counters
~~~~
UCS-A# connect nxos b
UCS-B(nxos)# clear counters interface san-port-channel 101
UCS-B(nxos)# clear counters interface fc1/20
UCS-B(nxos)# clear counters interface fc1/21
~~~~

## MDS

MDS remove port Channel
~~~~
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
~~~~

Configuration SAN port Channel
~~~~

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
~~~~

Show and Clear counters 
~~~~
show interface fc1/45
show interface fc1/45 counters

clear counters interface all clear counters interface fc1/45

show log interface
~~~~


# Active Directory

Check the replication health
~~~~
Repadmin /replsummary
~~~~
Check the inbound replication requests that are queued
~~~~
Repadmin /Queue
~~~~
Check the replication status
~~~~
Repadmin /Showrepl
~~~~
Synchronize replication between replication partners
~~~~
Repadmin /syncall
~~~~
AD Get FSMO Roles
~~~~
netdom query fsmo
~~~~

# Network route

## Windows
~~~~
route add -p 1.1.1.0 mask 255.255.255.0 2.2.2.2
route delete 1.1.1.0
~~~~

## Linux
~~~~
ip route add 192.168.1.0/24 dev eth0
route add default gw 192.168.1.254 eth0
ip route add 10.38.0.0/16 via 192.168.100.1
~~~~

# GIT
~~~~
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
~~~~


