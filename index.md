# PowerCli 

Connect / Disconnect to vCenter or ESXi
~~~~
Connect-VIServer -Server x.x.x.x
Disconnect-VIServer
Disconnect-VIServer -server *
~~~~

Start / Stop SSH Service on ESXi
~~~~
Get-VMHostService -VMHost "esx* "| ?{$_.Label -eq "SSH"} | Stop-VMHostService -Confirm:$false
Get-VMHostService -VMHost "esx*" | ?{$_.Label -eq "SSH"} | Start-VMHostService
~~~~

Get Esx uuid
~~~~
Get-VMHost | Select Name,@{n="HostUUID";e={$_.ExtensionData.hardware.systeminfo.uuid}}
~~~~

Add VM to DRS group
~~~~
Get-DrsClusterGroup -Cluster $Cluster -Name $DRSvmGroupName | Set-DrsClusterGroup -Add -VM $VMsToAdd;
~~~~

Snapshot multiple VM per folder
~~~~
Get-Folder test-folder | Get-VM | New-Snapshot -Name init_snap
~~~~

Get VM restarted by HA
~~~~
Get-VIEvent -type warning | Where {$_.FullFormattedMessage -match "restarted"} |select ObjectName,CreatedTime,FullFormattedMessage |sort CreatedTime -Descending |export-CSV .\HA_VMs.csv
~~~~
