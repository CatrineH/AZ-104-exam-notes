# VM Commands – Azure CLI and PowerShell

[← Back to README](../README.md) · Based on Microsoft Learn: *Common Azure CLI commands – Azure Virtual Machines*

## Read any command

| | Azure CLI | Azure PowerShell |
|---|---|---|
| Pattern | `az <noun> <verb>` | `Verb-AzNoun` |
| Read / view | `list`, `show` | `Get-` (`Get-AzVM`, `Get-AzDisk`) |
| Create | `create` | `New-` (`New-AzVM`, `New-AzDisk`) |
| Change | `update`, `set` | `Set-` / `Update-` (`Set-AzVM`, `Update-AzVM`) |
| Delete | `delete` | `Remove-` (`Remove-AzVM`, `Remove-AzDisk`) |

## Create VMs

| Task | Azure CLI | PowerShell |
|---|---|---|
| Create a resource group | `az group create --name myRG --location norwayeast` | `New-AzResourceGroup -Name myRG -Location norwayeast` |
| Create a Linux VM | `az vm create -g myRG -n myVM --image Ubuntu2404 --generate-ssh-keys` | `New-AzVM -ResourceGroupName myRG -Name myVM -Image Ubuntu2204 -Credential $cred` |
| Create a Windows VM | `az vm create -g myRG -n myVM --image Win2022Datacenter --admin-username azureuser` | `New-AzVM -ResourceGroupName myRG -Name myVM -Image Win2022Datacenter -Credential $cred` |

## Manage VM state

| Task | Azure CLI | PowerShell |
|---|---|---|
| Start | `az vm start -g myRG -n myVM` | `Start-AzVM -ResourceGroupName myRG -Name myVM` |
| Stop (**still billed**) | `az vm stop -g myRG -n myVM` | `Stop-AzVM -ResourceGroupName myRG -Name myVM -StayProvisioned` |
| Deallocate (not billed for compute) | `az vm deallocate -g myRG -n myVM` | `Stop-AzVM -ResourceGroupName myRG -Name myVM` |
| Restart | `az vm restart -g myRG -n myVM` | `Restart-AzVM -ResourceGroupName myRG -Name myVM` |
| Redeploy (new host) | `az vm redeploy -g myRG -n myVM` | `Set-AzVM -ResourceGroupName myRG -Name myVM -Redeploy` |
| Delete | `az vm delete -g myRG -n myVM` | `Remove-AzVM -ResourceGroupName myRG -Name myVM` |

Note the trap: in **PowerShell**, `Stop-AzVM` **deallocates** by default. In **CLI**, `az vm stop` does **not** deallocate.

## Get VM info

| Task | Azure CLI | PowerShell |
|---|---|---|
| List VMs | `az vm list -o table` | `Get-AzVM` |
| Details of a VM | `az vm show -g myRG -n myVM` | `Get-AzVM -ResourceGroupName myRG -Name myVM` |
| Power state | `az vm show -d -g myRG -n myVM --query powerState` | `Get-AzVM -ResourceGroupName myRG -Name myVM -Status` |
| vCPU usage / quota | `az vm list-usage --location norwayeast` | `Get-AzVMUsage -Location norwayeast` |
| Sizes in a region | `az vm list-sizes --location norwayeast` | `Get-AzVMSize -Location norwayeast` |
| Sizes this VM can resize to | `az vm list-vm-resize-options -g myRG -n myVM` | `Get-AzVMSize -ResourceGroupName myRG -VMName myVM` |

## Disks and images

| Task | Azure CLI |
|---|---|
| Add a new data disk | `az vm disk attach -g myRG --vm-name myVM --name myDataDisk --size-gb 128 --new` |
| List attached disks | `az vm show -g myRG -n myVM --query "storageProfile"` |
| Detach a data disk | `az vm disk detach -g myRG --vm-name myVM --name myDataDisk` |
| Resize a disk | `az disk update -g myRG -n myDataDisk --size-gb 256` |
| Snapshot a disk | `az snapshot create -g myRG -n mySnapshot --source myDataDisk --incremental true` |
| Create image of a VM | `az image create -g myRG --source myVM --name myImage` |
| Create VM from image | `az vm create -g myRG -n myNewVM --image myImage` |

PowerShell versions: see [VMs: plan and deploy](../05-compute/03-vms-plan-deployment.md#powershell-equivalents).

## Related
- [Command recognition (governance)](command-recognition-gov.md)
- [How to study hands-on](how-to-study-hands-on.md)
