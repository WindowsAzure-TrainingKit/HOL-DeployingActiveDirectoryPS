<a name="Deploy-AD-in-Windows-Azure" />
# Deploy Active Directory in Windows Azure #

---
<a name="Overview" />
## Overview ##

In this lab, you will create a new Windows Server 2012 VM called DC02 in Windows Azure using the Windows Azure management console in your web browser and then deploy Active Directory using Server Manager on DC02. DC02 will be the first domain controller in a new forest.

When deploying Active Directory in Windows Azure, two aspects are important to point out.

The first one is the networking configuration. Domain members and domain controllers need to find the DNS server hosting the domain DNS information. You will configure the Azure network configuration, so that it the correct DNS server is configured.

Secondly, it is important to avoid the possibility of Active Directory database corruption. Active Directory assumes that it can write its database updates directly to disk. That means that you should place the Active Directory database files on a data disk that does not have write caching enabled.

<a name="Objectives" /></a>
### Objectives ###

In this hands-on lab, you will learn how to:

- Configure Virtual Networking
- Deploy a Domain Controller 
- Create new Virtual Machines in 

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

<a name="Exercises" /></a>
## Exercises ##

This hands-on lab includes the following exercises:

1. [Configure Virtual Networking](#Exercise1)
1. [Create a new virtual machine from the gallery image](#Exercise2)
1. [Deploy a new domain controller in Windows Server 2012](#Exercise3)

<a name="Exercise1" /></a>
### Exercise 1: Configure Virtual Networking ###

Running an Active Directory Domain requires persistent IP addresses and for clients of the Active Directory Domain to point to an AD enabled DNS server. The default internal DNS service (iDNS) in Windows Azure is not an acceptable solution because the IP address assigned to each virtual machine is not persistent. For this solution you will define a virtual network where you can assign the virtual machines to specific subnets. 

The network configuration used for this lab defines the following:

- A Virtual Network Named domainvnet with an address prefix of: 10.0.0.0/16
- A subnet named Subnet-1 with an address prefix of: 10.0.0.0/24

Exercise 1 contains 2 tasks:

1. Creating an Affinity Group 
2. Creating a new Virtual Network

<a name="Ex1Task1" /></a>
#### Task 1 - Creating an Affinity Group ####

The first task is to create an affinity group for the Virtual Network. 

<h1>PLACEHOLDER<h1>

<a name="Ex1Task2" /></a>
#### Task 2 - Creating a new Virtual Network ####

The next step is to create a new virtual network to your subscription.

<h1>PLACEHOLDER<h1>

<a name="Exercise2" /></a>
### Exercise 2: Create a new virtual machine from the gallery image ###

You will now create a new virtual machine from a Windows Server 2012 gallery image called DC02.  This virtual machine will be used to create a domain controller in the next exercise. We will then create and provision a data disk which will be used in exercise 3 to place the AD database files.

Exercise 2 contains 2 tasks:

1. Create a new virtual machine
1. Configure a new data disk on DC02

<a name="Ex2Task1" /></a>
#### Task 1 - Create a new Virtual Machine with a data disk ####

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

<a name="Ex2Task2" /></a>
#### Task 2 - Configure a new data disk ####

1. In the Windows Azure console, in the **Virtual Machines** section, wait a few moments until the status of **DC02** is **Running**.

1. Select the **DC02** virtual machine, and then on the toolbar, click the **Connect** icon.

	![Connecting to the DC02 VM](./images/connecting-to-the-dc01-vm.png?raw=true "Connecting to the DC02 VM")

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

<a name="Exercise3" /></a>
### Exercise 3: Deploy a new domain controller in Windows Server 2012 ###
You have just created a base virtual machine called DC02, attached the necessary data disk, and provisioned the disk. We are going to login to DC02 to install and configure active directory and then verify the install was successful.

Exercise 3 contains 3 tasks:

1. Install the Active Directory Domain Services Role 
1. Configure the Active Directory Domain Services Role
1. Verify the Domain Controller Installed Successfully

<a name="Ex3Task1" /></a>
#### Task 1 - Install the Active Directory Domain Services Role ####

1. Open a PowerShell window, and type the following command:

	````PowerShell
	Add-WindowsFeature -Name AD-Domain-Services  -IncludeManagementTools
	````

	_Windows is installing the Active Directory Domain Services role._

	![Adding the AD feature](./images/adding-the-ad-feature.png?raw=true "Adding the AD feature")

	_Adding the Active Directory feature_

<a name="Ex3Task2" /></a>
#### Task 2 - Configure the Active Directory Domain Services Role ####
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

<a name="Ex3Task3" /></a>
#### Task 3 - Verify the Domain Controller Installed Successfully ####

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

<a name='Summary'/>
## Summary ##

In this lab, you walked through the steps of deploying a new Active Directory Domain using Windows Azure virtual machines and a simple virtual network. This lab also demonstrated how once in place virtual machines could be provisioned joined to the domain at boot time.