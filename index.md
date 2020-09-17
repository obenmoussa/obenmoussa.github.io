# PowerCli 

 

## Basics 
Connect / Disconnect to vCenter or ESXi
~~~~
Connect-VIServer -Server x.x.x.x
Disconnect-VIServer
Disconnect-VIServer -server *
~~~~

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
