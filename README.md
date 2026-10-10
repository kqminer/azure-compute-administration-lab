# Azure Compute Administration Lab

## Project Overview

This project documents a hands-on Azure compute administration lab built around a Linux virtual machine in Central US. The goal was to practice the work a junior Azure administrator may handle: deploying a VM, securing administrative access, managing disks, controlling permissions, automating software installation, configuring backup, and rebuilding the environment with Bicep.

The lab was completed with both the Azure portal and Azure Cloud Shell. I also documented the problems I ran into, including subscription quota limits and regional capacity restrictions.

## Skills Demonstrated

- Linux virtual machine deployment and administration
- Azure VM size and quota troubleshooting
- Network security group configuration
- SSH key authentication
- Managed disks, mounting, and persistent storage
- Disk snapshots
- VM resizing
- Azure RBAC at resource scope
- Custom Script Extension
- Nginx installation and service verification
- System-assigned managed identity
- Azure Storage data-plane RBAC
- Recovery Services vault and Azure Backup
- Infrastructure as Code with Bicep
- Resource validation and cost-conscious cleanup

## Environment

| Resource | Configuration |
|---|---|
| Resource group | `rg-compute-lab` |
| Region | Central US |
| Virtual machine | `vm-compute-dev-cus-001` |
| Operating system | Ubuntu Server 22.04 LTS |
| Final VM size | `Standard_D2s_v6` |
| Virtual network | `vnet-compute-dev-cus-001` |
| Subnet | `subnet-compute-001` |
| Network security group | `nsg-compute-dev-cus-001` |
| Data disk | `disk-compute-data-cus-001` |
| Snapshot | `snap-compute-data-cus-001` |
| Recovery Services vault | `rsv-compute-lab-cus-001` |

## Architecture

The VM was placed in a dedicated virtual network and subnet. SSH access was restricted through an NSG rule instead of being left open to the internet. A separate managed data disk was attached and mounted at `/data`. Azure Backup protected the VM through a Recovery Services vault.

The Bicep version of the environment deployed a separate test VM, network, NSG, public IP, NIC, data disk, managed identity, and Custom Script Extension.

## Implementation

### 1. VM Deployment and Availability Troubleshooting

I first checked available VM SKUs in Central US. The initial B-series v2 deployment failed because the subscription had a quota limit of zero for that VM family. A later attempt with `Standard_B1ms` passed the quota check but failed because the size did not have regional capacity.

I changed to an available D-series size and completed the VM deployment.

### 2. Secure SSH Access

I configured SSH key authentication instead of a reusable password. The NSG allowed TCP 22 only from my authorized source address. This reduced exposure while still allowing remote administration.

### 3. Managed Data Disk

I attached a 32 GiB Standard SSD data disk to the VM, identified it in Linux, created a filesystem, and mounted it at `/data`. I added the disk UUID to `/etc/fstab` so the mount would survive a reboot.

I created a test file on the disk before taking a snapshot to prove that the disk contained data.

### 4. Snapshot and Resize

I created `snap-compute-data-cus-001` from the managed data disk. I also resized the VM to practice changing compute capacity and later returned it to `Standard_D2s_v6`.

### 5. Resource-Scoped RBAC

I assigned permissions directly at the VM scope:

- `grp-lab-developers`: Virtual Machine Contributor
- `grp-lab-compliance`: Reader

This demonstrated least privilege by limiting access to the VM rather than granting permissions across the entire subscription.

### 6. Nginx Automation

I used the Azure Custom Script Extension to install and start Nginx. I verified the result from inside the VM:

```text
Service: active
HTTP: 200
```

### 7. Managed Identity and Storage Access

I enabled a system-assigned managed identity on the VM and granted it the Storage Blob Data Reader role. From the VM, I requested a token from the Azure Instance Metadata Service and used that identity to reach Azure Storage without storing account keys in the script.

### 8. Azure Backup

I created a Recovery Services vault, enabled VM protection, and started an on-demand backup. Azure showed that backup configuration completed successfully and the backup job entered `InProgress`.

### 9. Infrastructure as Code

I recreated the main compute environment in a separate resource group using [main.bicep](main.bicep). The template deploys:

- Virtual network and subnet
- Network security group with restricted SSH
- Static public IP
- Network interface
- Ubuntu virtual machine
- 32 GiB managed data disk
- System-assigned managed identity
- Custom Script Extension that installs Nginx

The Bicep file accepts the SSH public key and allowed SSH source as parameters rather than hard-coding personal values.

After deployment, I verified every resource and confirmed that Nginx returned HTTP 200. I then deleted the temporary IaC resource group to stop unnecessary charges.

## Validation Evidence

Screenshots are stored in the `screenshots` folder. Key evidence includes:

- SKU restriction and quota failures
- Successful VM deployment
- Restricted SSH NSG rule
- SSH public key configuration
- Data disk attachment, detection, formatting, and mounting
- Snapshot creation
- VM resizing
- VM-scoped RBAC assignments
- Nginx extension deployment and HTTP 200 verification
- Azure Backup configuration
- Managed identity access to Azure Storage
- Bicep validation and deployment
- Bicep resource inventory
- Nginx verification on the Bicep-deployed VM

## Troubleshooting

### VM Family Quota Was Zero

The first deployment failed because the subscription had no approved Bsv2-family vCPU quota in Central US. I checked regional quota instead of repeatedly retrying the same deployment.

### VM Size Had No Regional Capacity

`Standard_B1ms` appeared in the SKU list but failed during deployment because Azure did not have available capacity in the region. SKU visibility does not guarantee that a deployment will succeed at that moment.

### SSH Public Key Was Missing

The portal required a public key, but I did not initially have one available on the local computer. I generated an Ed25519 key pair and added only the public key to the VM.

### Cloud Shell Reset

Cloud Shell reset during the lab, which cleared session variables and a temporary SSH key. I recreated the required variables and generated a new key before rerunning the Bicep deployment.

### Backup Took Longer Than Expected

Backup configuration completed quickly, but the first on-demand backup remained in progress. I treated configuration status and job status as separate checks instead of assuming the operation had failed.

## What I Learned

This lab made the difference between availability, quota, and capacity much clearer. A VM size can exist in a region but still be blocked by subscription quota or temporary regional capacity.

I also learned that a VM deployment is only the starting point. A usable Azure compute workload also needs secure access, persistent storage, permissions, backup, monitoring of deployment results, and a repeatable way to rebuild it.

The strongest part of this project was rebuilding the environment with Bicep and validating the deployed service. That turned a one-time portal build into reusable infrastructure.

## Cost Management

The temporary Bicep deployment was deleted after validation. The original lab resources were retained only long enough to finish documentation and backup testing before cleanup.
