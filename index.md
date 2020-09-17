# PowerCli 

Start/stop SSH Service on ESXi
~~~~
Get-VMHostService -VMHost "esx* "| ?{$_.Label -eq "SSH"} | Stop-VMHostService -Confirm:$false
Get-VMHostService -VMHost "esx*" | ?{$_.Label -eq "SSH"} | Start-VMHostService
~~~~
