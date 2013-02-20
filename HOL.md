<a name="Deploy-AD-in-Windows-Azure" />
# Deploy Active Directory in Windows Azure #

---
<a name="Overview" />
## Overview ##

In this lab, you will create a new VM named DC01 from a Windows Server 2012 gallery image in Windows Azure and then deploy Active Directory on DC01. 

When deploying Active Directory in Windows Azure, two aspects are important to point out.

The first one is the networking configuration. Domain members and domain controllers need to find the DNS server hosting the domain DNS information. You will configure the Azure network configuration, so that it the correct DNS server is configured.

Secondly, it is important to avoid the possibility of Active Directory database corruption. Active Directory assumes that it can write its database updates directly to disk. That means that you should place the Active Directory database files on a data disk that does not have write caching enabled.

<a name='Objectives' />
### Objectives ###

In this hands-on lab, you will learn how to:
- Create a new Virtual Machine from a gallery image
- Configure Virtual Networking
- Promote the Virtual Machine to the first Domain Controller in a new forest. 

<a name="Prerequisites"></a> 
### Prerequisites ###

The following is required to complete this hands-on lab:

- [Windows PowerShell 2.0][1]
- [Windows Azure PowerShell CmdLets][2]
- A Windows Azure subscription with the Virtual Machines Preview enabled - [sign up for a free trial](http://aka.ms/WATK-FreeTrial)

[1]: http://microsoft.com/powershell/
[2]: http://msdn.microsoft.com/en-us/library/windowsazure/jj156055

<a name="Setup"></a> 
### Setup ###

The Windows Azure PowerShell Cmdlets are required for this lab. If you have not configured them yet, please see the **Automating VM Management** hands-on lab in the **Automating Windows Azure with PowerShell** module. 

>**Note:** In order to run through the complete hands-on lab, you must have network connectivity. 

<a name="Exercises" />
## Exercises ##

<a name="Exercise1" />
### Exercise 1: Deploy Active Directory in Windows Azure ###

<a name="Ex1Task1" />
#### Task 1 – Examine the current Domain Controller configuration ####

1. In the Windows Azure portal, in the **Virtual Machines** section, click the **DC01** name. _The DC01 page appears._

	![Selecting the DC01 vm](./images/selecting-the-dc01-vm.png?raw=true "Selecting the DC01 vm")

	_Selecting the DC01 virtual machine_

1. On the DC01 page, select the **Configure** tab. _The DC01 VM is configured with Virtual Network named **domainvnet**, and a Subnet named **Subnet-1**, containing the IP range 10.0.0.0/24._

	![Selecting the configure tab](./images/dc01-configure-tab.png?raw=true "Selecting the configure tab")

	_Selecting the configure tab_

1. On the Virtual Machines page, select the **DC01** virtual machine, and then on the toolbar, click the **Connect** icon. _The RDP file for DC01 is downloaded._

	![Downloading the RDP file](./images/downloading-the-rdp-file.png?raw=true "Downloading the RDP file")

	_Downloading the RDP file_

1. In the Windows Security dialog box, enter the credentials for the DC01 virtual machine:

	| Field | Value |
	|--------|--------|
	| User account | **Contoso\Administrator** |
	| Password | **Passw0rd!** |
 

	If a Remote Desktop Connection warning appears about the certificate name, click **Yes**.

	![Entering the credentials](./images/entering-the-credentials.png?raw=true "Entering the credentials")

	_Entering the credentials_

1. In the DC01 VM, use the **Windows-X C** command to open a Command Prompt window. 

1. In the Command Prompt window, type, **ipconfig /all**, and then press **Enter**. _The domain controller uses DHCP to receive a dynamically assigned IP address (10.0.0.4) with a lifelong lease time. The DNS server is also assigned as 10.0.0.4._

	![Executing ipconfig](./images/executing-ipconfig.png?raw=true "Executing ipconfig")

	_Executing ipconfig_


<a name="Ex1Task2" />
#### Task 2 – Configure Virtual Networking ####

1. In the Windows Azure portal, on the **Networks** page, click the **domainvnet** name.

	![Clicking the domainvnet](./images/clicking-the-domainvnet.png?raw=true "Clicking the domainvnet")
	
	_Clicking the domainvnet network_

1. On the **domainvnet** page, select the **Configure** tab. _The domainvnet network currently has one subnet (Subnet-1 - 10.0.0.0/24), and uses DNS server DC01 10.0.0.4._

	![domainvnet network information](./images/domainvnet-network-information.png?raw=true "domainvnet network information")

	_domainvnet network information_

1. Click the **add subnet** button. _A new subnet is created: Subnet-2 - 10.0.1.0/24. You can add additional virtual machines in the first subnet. For the lab exercise, we will define a second subnet._

	![Clicking the add subnet button](./images/clicking-the-add-subnet-button.png?raw=true "Clicking the add subnet button")

	_Clicking the add subnet button_

1. On the toolbar, click the **Save** button.
_The network configuration is saved. Virtual machines that are assigned to different subnets within the same virtual network can connect to each other._

	![Clicking the save button](./images/clicking-the-save-button.png?raw=true "Clicking the save button")

	_Clicking the save button_

<a name="Ex1Task3" />
#### Task 3 - Create a new Virtual Machine with a data disk ####

1. On the Start menu, start typing **ise**, and then click **Windows PowerShell ISE**. _For these steps, we will use the PowerShell Integrated Scripting Environment._

	![Opening Windows Powershell ISE](./images/opening-windows-powershell-ise.png?raw=true "Opening Windows Powershell ISE")

	_Opening Windows Powershell ISE_

1. In the PowerShell ISE window, type the following command:

	````PowerShell
	Get-AzureVMImage  |  ft ImageName
	````

	![Retriving the VM images](./images/retriving-the-vm-images.png?raw=true "Retriving the VM images")

	_Retriving the virtual machine images_

	> **Note:** _The list of available image files is displayed. In this task, we will use the latest Windows Server 2012 image, which is a699494373c04fc0bc8f2bb1389d6106__Windows-Server-2012-Datacenter-201301.01-en.us-30GB.vhd._

1. In the PowerShell ISE window, type or copy the following commands.

	````PowerShell
	# Defines image name
	$imgname = "a699494373c04fc0bc8f2bb1389d6106__Windows-Server-2012-Datacenter-201301.01-en.us-30GB.vhd"

	# Defines configuration settings
	$vmname = "DC02"
	$svcname = "itcampNNNN"
	$password = "Passw0rd!"

	# Defines network settings
	$subnetname = "Subnet-2"

	# Defines VM configuration, including size, attached datadisk, and subnet 
	$dcvm = New-AzureVMConfig  -Name  $vmname  -ImageName  $imgname -InstanceSize "Small"  |
		Add-AzureProvisioningConfig  -Windows  -Password  $password  |
		Add-AzureDataDisk  -CreateNew  -DiskSizeInGB  10  -DiskLabel "ADdisk"  -LUN 0  |
		Set-AzureSubnet  -SubnetName  $subnetname

	# Creates new VM in existing hosted service
	New-AzureVM  -ServiceName  $svcname  -VMs  $dcvm
	````

	![Executing the powershell commands](./images/executing-the-powershell-commands.png?raw=true "Executing the powershell commands")

	_Executing the powershell commands_

	> **Note:** Do not forget to replace itcampNNNN, with the actual hosted service name you used earlier.

	> **Note:** The **New-AzureVM** cmdlet will create a new hosted service when the -Location or -AffinityGroup parameter is specified, or will use an existing hosted service when those parameters are not specified. 

1. In the Windows Azure console, in the **Virtual Machines** section, wait a few minutes until the **DC02** virtual machine appears (or press F5 to refresh the display).

1. Click the **DC02** name.

	![Selecting the DC02 VM](./images/selecting-the-dc02-vm.png?raw=true "Selecting the DC02 VM")

	_Selecting the DC02 Virtual Machine_

1. On the DC02 page, select the **Endpoints** tab. _Notice that by default the PowerShell cmdlet will add an RDP endpoint for port 3389. Therefore, we did not specify this endpoint configuration in the PowerShell commands._

	![Selecting the endpoints tab](./images/selecting-the-endpoints-tab.png?raw=true "Selecting the endpoints tab")

	_Selecting the endpoints tab_

<a name="Ex1Task4" />
#### Task 4 - Configure a new data disk ####

1. In the Windows Azure console, in the **Virtual Machines** section, wait a few moments until the status of **DC02** is **Running**.

1. Select the **DC02** virtual machine, and then on the toolbar, click the **Connect** icon.

	![Connecting to the DC02 VM](./images/connecting-to-the-dc02-vm.png?raw=true "Connecting to the DC02 VM")

	_Connecting to the DC02 virtual machine_

1. Open the DC02.rdp file, and connect to the virtual machine. To log on, use credentials:

	| Field | Value |
	|--------|--------|
	| Account | **Administrator** |
	| Password | **Passw0rd!** |

	![Logging on to the DC02 VM](./images/logging-on-to-the-dc02-vm.png?raw=true "Logging on to the DC02 VM")

	_Logging on to the DC02 virtual machine_

1. On DC02, in **Server Manager**, on the **Tools** menu, click **Computer Management**. _The Computer Management console opens._

	![Opening the Computer Manager console](./images/opening-the-computer-manager-console.png?raw=true "Opening the Computer Manager console")

	_Opening the Computer Manager console_

1. In the Computer Management console, in the left pane, select **Disk Management**. _Disk Management recognizes that a new initialize disk is added to the computer, and it will show the Initialize Disk dialog box._

	![Selecting Disk Management](./images/selecting-disk-management.png?raw=true "Selecting Disk Management")

	_Selecting Disk Management_

1. In the Initialize Disk dialog box, click **OK**. _The new Disk 2 is initialized._

	![Initializing the disk 2](./images/initializing-the-disk-2.png?raw=true "Initializing the disk 2")

	_Initializing the disk 2_

1. On Disk 2, right-click the **Unallocated** space, and then click **New Simple Volume**. _The New Simple Volume Wizard opens._

	![Formating the unallocated space](./images/formating-the-unallocated-space.png?raw=true "Formating the unallocated space")

	_Formating the unallocated space_

1. In the new Simple Volume Wizard, click **Next**.

	![Using the Simple Volume Wizard](./images/using-the-simple-volume-wizard.png?raw=true "Using the Simple Volume Wizard")

	_Using the Simple Volume Wizard_

1. On the Specify Volume Size page, click **Next**. _This means that the entire available space (10237 MB) will become a new volume._

	![Specifing the volume size](./images/specifing-the-volume-size.png?raw=true "Specifing the volume size")

	_Specifing the volume size_

1. On the Assign Drive Letter or Path page, ensure drive letter **F** is selection, and then click **Next**.

	![Assigning the drive letter](./images/assigning-the-drive-letter.png?raw=true "Assigning the drive letter")

	_Assigning the drive letter_

1. On the Format Partition page, in the **Volume Label** text box, type **AD DS Data**, and then click **Next**.

	![Specifing the volume label](./images/specifing-the-volume-label.png?raw=true "Specifing the volume label")

	_Specifing the volume label_

1. On the Completing the New Simple Volume Wizard page, click **Finish**. _Windows will quick format the disk, and assign drive letter F:._

	![Completing the wizard](./images/completing-the-wizard.png?raw=true "Completing the wizard")

	_Completing the wizard_

1. Close the Computer Management console.

<a name="Ex1Task5" />
#### Task 5 - Create a Domain Controller ####

1. In the DC02 VM, use the Windows X C command to open a Command Prompt window. In the Command Prompt Window, type **ipconfig /all**, and then press **Enter**. _The new VM uses IP address 10.0.1.4, and DNS server address 10.0.0.4._

	![Executing ipconfig](./images/executing-ipconfig.png?raw=true "Executing ipconfig")

	_Executing ipconfig_

1.  In the Command Prompt window, type **ping 10.0.0.4**, and then press **Enter**. _The replies from 10.0.0.4 confirm that the new VM can connect to DC01._

	![Pinging the DC01 VM](./images/pinging-the-dc01-vm.png?raw=true "Pinging the DC01 VM")

	_Pinging the DC01 virtual machine_

1. Open a PowerShell window, and type the following command:

	````PowerShell
	Add-WindowsFeature -Name AD-Domain-Services  -IncludeManagementTools
	````

	_Windows is installing the Active Directory Domain Services role._

	![Adding the AD feature](./images/adding-the-ad-feature.png?raw=true "Adding the AD feature")

	_Adding the Active Directory feature_

1. When the feature installation has completed, type the following single command to promote the domain controller:

	````PowerShell
	Install-ADDSDomainController  -DomainName "contoso.com"  -SiteName "Default-First-Site-Name" -InstallDns:$false  -NoGlobalCatalog:$false  -Credential (Get-Credential)  -DatabasePath "F:\NTDS"  -LogPath "F:\NTDS"  -SysvolPath "F:\SYSVOL"  -NoRebootOnCompletion:$false  -Force:$true
	````

	_The C: disk is the OS disk, and has caching enabled. The Active Directory database should not be stored on a disk that has write caching enabled. The F: disk is a data disk we added earlier, and does not have caching enabled._

	![Promoting the domain controller with powershell ](./images/promoting-the-domain-controller-with-powershell.png?raw=true "Promoting the domain controller with powershell")

1. In the Credential dialog box, use the following credentials:

	| Field | Value |
	|--------|--------|
	| User account | **Contoso\Administrator** |
	| Password | **Passw0rd!** |

	![Using administrator credentials](./images/using-administrator-credentials.png?raw=true "Using administrator credentials")

	_Using administrator credentials_

1. At the **SafeModeAdministratorPassword** prompt and the **Confirm SafeModeAdministratorPassword** prompt, type **Passw0rd!**, and then press **Enter**. _The computer is promoted to domain controller. After a few moments, the DC02 VM will restart. You will lose the connection to the restarting VM._

	![Configuring the administrator password](./images/configuring-the-administrator-password.png?raw=true "Configuring the administrator password")

	_Configuring the administrator password_

1. Wait two minutes for the DC02 VM to restart.
In the Azure portal, on the Virtual Machines page, select **DC02**, and on the toolbar, click the **Connect** icon.

	![Downloading the DC02 RDP file](./images/downloading-dc02-rdp-file.png?raw=true "Downloading the DC02 RDP file")

	_Downloading the DC02 RDP file_

1. Open the DC02.rdp file, and connect to the virtual machine. To log on, use credentials:

	| Field | Value |
	|--------|--------|
	| User account | **Contoso\Administrator** |
	| Password | **Passw0rd!** |

	![Connecting to the VM](./images/connecting-to-the-vm.png?raw=true "Connecting to the VM")

	_Connecting to the virtual machine_

1. To verify that DC02 is working properly, run the following command:

	````PowerShell
	repadmin.exe /syncall
	````

	_This output of the command confirms that DC02 was successfully promoted to domain controller._


---

<a name='Summary'/>
## Summary ##

In this lab, you walked through the steps of deploying a new Active Directory Domain using Windows Azure virtual machines and a simple virtual network. This lab also demonstrated how once in place virtual machines could be provisioned joined to the domain at boot time.