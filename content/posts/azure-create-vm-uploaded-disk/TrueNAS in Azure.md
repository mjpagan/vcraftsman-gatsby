# Create a VM in Azure with a Uploaded Virtual Disk 

## Introduction

I recently had the chance to play around with IX Systems storage appliances and their storage operating system, TrueNAS. After installing it in a virtual machine and tinkering with some of the features, my interest was piqued. While scanning the product documentation, I noticed that there are instructions for deployment on AWS but none for Azure. I checked with support and they said there wasn't a technical limitation, they hadn't worked on getting it certified yet. That had me thinking about how how would I go about making this work in Azure.

## Goal

I took it upon myself to see if I could make a TrueNAS appliance in Azure. I've never created a custom image for Azure or really spent much time using Hyper-V so it was time to roll up my sleeves and learn some new things. This post details my process and the result. I hope you find my journey and the result useful.

## Journey 

Initially, I did some research on converting an existing VMware OVA image and getting that to work in Azure. After a few attempts, it became apparent that this was more complicated than it needed to be.

Next, I used Veeam to backup then restore a TrueNAS virtual machine from a VMware environment into Azure. This worked and I was able to connect to the OS and configure settings, but I didn't want everyone who wanted to run TrueNAS in Azure to go through the backup/restore process I needed something that was more portable.

I tried creating a TrueNAS Hyper-V image to simplify the conversion and was able to convert the Hyper-V virtual disk into a disk image that can be used in Azure. I uploaded the disk and then built a virtual machine around it. Then, I turned the virtual machine into an Azure Image for redeployment.

My idea to use an Azure image presented some deployment issues based on the script I'd cobbled together. The script would run, create the VM and then wait another 25 minutes before throwing an error message that ended up not affecting the VM. I worked on that for a while but couldn't find a solution so I simplified my process a bit.

This time around I took the virtual disk created from the Hyper-V virtual machine and instead of worrying about creating an image I just cloned the disk this allowed me to just create the VM with a PowerShell script into my already existing tenant. I can now reproduce creating a basic TrueNAS virtual machine in Azure rather easily with my base disk.

I originally created my Hyper-V image with the TrueNAS 12.X Core software which is built on FreeBSD. There were a couple of issues I ran into with FreeBSD (mostly my own limitations working in UNIX) so I switched the image to TrueNAS SCALE software which is based on Debian Linux.

Even though this code is currently an "alpha" I was able to install the [Azure Linux Agent](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/agent-linux) in the first release (20.10-ALPHA). The builds are progressing quickly and now the waagent is included in the 20.12-ALPHA code.

## Challenges

Uploading a Hyper-V virtual disk to Azure is a lot harder than I expected. One of the bigger issues I had was uploading the virtual disks to Azure. 

A virtual disk that is uploaded to Azure needs the following characteristics ([Convert the virtual disk to a fixed size VHD](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/prepare-for-upload-vhd-image#convert-the-virtual-disk-to-a-fixed-size-vhd)) :

- Disks in Azure must have a virtual size aligned to 1 MiB
- Disk "Size" must be a multiple of 1 MiB and Disk "FileSize" must be the "Size" plus 512 byte VHD footer
- Maximum size for a OS VHD is 2 TiB
- Maximum size for a data VHD is 32 TiB

I also wanted to keep the OS disk small to keep the virtual disk costs low. To create the proper sized VHD file I did the following steps:

1. Created a Hyper-V virtual machine with a 28 GiB OS (dynamic) disk

2. Installed TrueNAS SCALE and performed initial configuration

3. Shutdown the VM

4. Converted it to a fixed disk with PowerShell  (the Hyper-V GUI would've also worked)

   ```powershell
   Convert-VHD .\TrueNAS.vhdx D:\Azure\TrueNAS00.vhd -VHDType fixed
   ```

5. Resized the VM to 31GB (any size would do, I kept it small because the OS only needs 32GB)

   ```powershell
   Resize-VHD .\TrueNAS00.vhd -SizeBytes 31GB
   ```

6. Since that lead to a faction of a 1 MiB I resized it to a size divisible by 1024

   ```powershell
   Resize-VHD .\TrueNAS00.vhd -SizeBytes 33285996544
   ```

   Here you can see what the resize command and the get-vhd command output looks like: 

   ![step-1-resize-vhd](/images/step-1-resize-vhd.png)

   As you can see the Size is listed in bytes. For this to be a valid VHD that can be uploaded to Azure it needs to be divisible by 1 MiB (1 MiB = 1048576 bytes).
   $$
   33285996544~bytes / 1048576~bytes = 31,744~bytes = 31~GiB
   $$
   If the result was not a whole number we'd need to resize the VHD. In this case, it is a whole number so it is a valid VHD. Now we can use PowerShell or Azure Storage Explorer to upload the VHD to your Resource Group of choice as a managed disk.

## Creating the VM

Creating the Virtual machine is relatively straight forward. I used Microsoft documentation and PowerShell examples I found online to create the script below.

Here is my script: [Create TrueNAS VM from uploaded VHD.ps1](https://github.com/mjpagan/TrueNAS/blob/main/Create%20TrueNAS%20VM%20from%20uploaded%20VHD.ps1)

If you don't want to go through the process of creating your own VHD, you can use mine. The credentials are root / TrueNAS

My TrueNAS SCALE Image: [TrueNAS00.vhd](https://truenas2609.file.core.windows.net/truenas/TrueNAS00.vhd?st=2020-12-23T06%3A32%3A36Z&se=2021-12-31T06%3A32%3A00Z&sp=rl&sv=2018-03-28&sr=f&sig=4JuuZkSu1rfNXmvUA6HTvhakso1j%2FCrVZ8RPHkOmUIM%3D)

```powershell
# Create TrueNAS VM from uploaded VHD
#
# Assumptions: 
# You have the following resources created:
#  -Resourcegroup
#  -Virtual network w/subnet
#  -Virtual network gateway deployed and VPN tunnel created (if on-prem access is desired)
#
# Step 1: Upload the VHD to your subscription as a managed disk (not into a storage account). I used Azure Storage Explorer for this task. 
#
# Step 2: Fill in your information and copy and paste this into Cloud Shell or Azure PowerShell. It will prompt you to enter credentials, but they do not actually apply to the OS so they will not be used.
#
# Step 3: Wait for the VM to be created. You should not have to wait for the progress bar to complete as it not recognizing that the resoruces have been created. It will eventually timeout and throw and error.
#
# Step 4: Modify or delete the inbound NSG rule allowing HTTPS depending on your security requirements.
#
# Step 5: Connect to TrueNAS server via the "internal" IP address and configure to your needs.
#
# ---------------------------------------------------------------------------------------------------------------------------------------------------
#
# Setting the table

$vmName = "TN-Test01"
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize "Standard_B2ms"

$rgname = "Lab_RG"
$vnetName = "lab-ncus-vnet"
$location = "northcentralus"

# Name of your uploaded managed disk
$osDiskName = "TrueNAS01"

$vnet = Get-azvirtualnetwork -Name $vnetName

$nicName = $vmName+"-nic"
$nic = New-AzNetworkInterface -Name $nicName `
    -ResourceGroupName $rgname `
    -Location $location -SubnetId $vnet.Subnets[0].Id `

$osDisk = Get-AzDisk -Name $osDiskName

$vm = Add-AzVMNetworkInterface -VM $vmConfig -Id $nic.Id

$vm = Set-AzVMOSDisk -VM $vm -ManagedDiskId $osDisk.Id -StorageAccountType Standard_LRS `
    -DiskSizeInGB 32 -CreateOption Attach -Linux

New-AzVM -ResourceGroupName $rgname -Location $location -VM $vm
```

To create the VM, I used Visual Studio Code to edit and run the PowerShell script. 

![step-2-edit-run-script](/images/step-2-edit-run-script.png)

The script ran successfully and as we can see it created the VM with the attributes we defined. 

![step-3-vm-created](/images/step-3-vm-created.png)

From here you can you can add additional data or caching disk to create your storage pool. After that, connect to the management interface and login to configure your TrueNAS appliance settings using the credentials you defined when you created the VM.

![step-4-truenas-logon](/images/step-4-truenas-logon.png)

## Wrap up

This project required me to sharpen my skills in PowerShell and Azure. I was certainly frustrated a number of times, but I was able to work through it. In the future, I'd like to add more to my script to add data disks and be able to copy the VHD directly from my storage account into my resource group, but as it stands I achieved my goal of being able to create a basic TrueNAS appliance in Azure.  

Skilled used/learned:

- Creating a virtual machine using PowerShell
- Creating virtual machines in Hyper-V
- Converting and uploading Hyper-V virtual disks to Azure using [Microsoft Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/)
- Unix/Linux commands
- Creating virtual machines in Azure with PowerShell
- Using GitHub and Visual Studio Code

Links:

- [TrueNAS.com](https://www.truenas.com/)
- [Install FreeNAS in Hyper-V: Part 1 Basic Configuration (servethehome.com)](https://www.servethehome.com/install-freenas-hyperv-part-1-basic-configuration/)

