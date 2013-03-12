<a name="Deploy-AD-in-Windows-Azure" />
# Deploy Active Directory in Windows Azure using PowerShell#

---
<a name="Overview" /></a>
## Overview ##

In this lab, you will create a new Windows Server 2012 VM called DC01 in Windows Azure using the Windows Azure management console in your web browser and then deploy Active Directory using Server Manager on DC01. DC01 will be the first domain controller in a new forest.

When deploying Active Directory in Windows Azure, two aspects are important to point out.

The first one is the networking configuration. Domain members and domain controllers need to find the DNS server hosting the domain DNS information. You will configure the Azure network configuration, so that it the correct DNS server is configured.

Secondly, it is important to avoid the possibility of Active Directory database corruption. Active Directory assumes that it can write its database updates directly to disk. That means that you should place the Active Directory database files on a data disk that does not have write caching enabled.

<a name="Objectives" /></a>
### Objectives ###

In this hands-on lab, you will learn how to:

- Configure Virtual Networking
- Create new Virtual Machines from Gallery Images
- Deploy a Domain Controller


<a name="Prerequisites"></a> 
### Prerequisites ###

The following is required to complete this hands-on lab:

- [Windows PowerShell 2.0][1]
- [Windows Azure PowerShell CmdLets][2]
- A Windows Azure subscription with the Virtual Machines Preview enabled - [sign up for a free trial](http://aka.ms/WATK-FreeTrial)

[1]: http://microsoft.com/powershell/
[2]: http://msdn.microsoft.com/en-us/library/windowsazure/jj156055

>**Note:** In order to run through the complete hands-on lab, you must have network connectivity. 

<a name='gettingstarted' /></a>
### Getting Started: Obtaining Subscription's Credentials ###

In order to complete this lab, you will need your subscription’s secure credentials. Windows Azure lets you download a Publish Settings file with all the information required to manage your account in your development environment.

#### Task 1 - Downloading and Importing a Publish-settings File ####

> **Note:** If you have done these steps in a previous lab on the same computer you can move on to Exercise 1.


In this task, you will log on to the Windows Azure Portal and download the publish-settings file. This file contains the secure credentials and additional information about your Windows Azure Subscription to use in your development environment. Then, you will import this file using the Windows Azure Cmdlets in order to install the certificate and obtain the account information.

1.	Open an Internet Explorer browser and go to <https://windows.azure.com/download/publishprofile.aspx>.

1.	Sign in using the **Microsoft Account** associated with your **Windows Azure** account.

1.	**Save** the publish-settings file to your local machine.

	![Downloading publish-settings file](images/downloading-publish-settings-file.png?raw=true 'Downloading publish-settings file')

	_Downloading publish-settings file_

	> **Note:** The download page shows you how to import the publish-settings file using Visual Studio Publish box. This lab will show you how to import it using the Windows Azure PowerShell Cmdlets instead.

1. In the start menu under **Windows Azure**, right-click **Windows Azure PowerShell** and choose **Run as Administrator**.

1.	Change the PowerShell execution policy to **RemoteSigned**. When asked to confirm press **Y** and then **Enter**.
	
	<!-- mark:1 -->
	````PowerShell
	Set-ExecutionPolicy RemoteSigned
	````

	> **Note:** The Set-ExecutionPolicy cmdlet enables you to determine which Windows PowerShell scripts (if any) will be allowed to run on your computer. Windows PowerShell has four different execution policies:
	>
	> - _Restricted_ - No scripts can be run. Windows PowerShell can be used only in interactive mode.
	> - _AllSigned_ - Only scripts signed by a trusted publisher can be run.
	> - _RemoteSigned_ - Downloaded scripts must be signed by a trusted publisher before they can be run.
	> - _Unrestricted_ - No restrictions; all Windows PowerShell scripts can be run.
	>
	> For more information about Execution Policies refer to this TechNet article: <http://technet.microsoft.com/en-us/library/ee176961.aspx>


1.	The following script imports your publish-settings file and generates an XML file with your account information. You will use these values during the lab to manage your Windows Azure Subscription. Replace the placeholder with your publish-setting file’s path and execute the script.

	<!-- mark:1 -->
	````PowerShell
	Import-AzurePublishSettingsFile '[YOUR-PUBLISH-SETTINGS-PATH]'   
	````

1. Execute the following commands and take note of the Subscription name and the storage account name you will use for the exercise.

	<!-- mark:1-3 -->
	````PowerShell
	Get-AzureSubscription | select SubscriptionName
	Get-AzureStorageAccount | select StorageAccountName 
	````

1. If you do NOT have a storage account returned above you should create one first.
  
	1. Run the following to determine the data center to create your storage account in. Ensure you pick a data center that shows support for PersistentVMRole. 

		````PowerShell
		Get-AzureLocation  
		````

	1. Create your storage account: 


		````PowerShell
		New-AzureStorageAccount -StorageAccountName '[YOUR-SUBSCRIPTION-NAME]' -Location '[DC-LOCATION]'
		````

1. Execute the following command to set your current storage account for your subscription.

	<!-- mark:1 -->
	````PowerShell
	Set-AzureSubscription -SubscriptionName '[YOUR-SUBSCRIPTION-NAME]' -CurrentStorageAccount '[YOUR-STORAGE-ACCOUNT]'
	````

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

1. On the Start menu, start typing **ise**, and then click **Windows PowerShell ISE**. _For this task and most of this HOL, we will use the PowerShell Integrated Scripting Environment._

	![Opening Windows Powershell ISE](./images/opening-windows-powershell-ise.png?raw=true "Opening Windows Powershell ISE")

	_Opening Windows Powershell ISE_

1. In the PowerShell ISE window, type the following command:
	
	````PowerShell
	# Creates the affinity group
	New-AzureAffinityGroup -Location "_LOCATION_" -Name agdomain

	> **Note:** For the _LOCATION_ variable above, please replace it with the exact text below (minus the number) from the datacenter closest to you:
	1. West US
	2. East US
	3. East Asia
	4. South East Asia
	5. North Europe
	6. West Europe

<a name="Ex1Task2" /></a>
#### Task 2 - Creating a new Virtual Network ####

The next step is to create a new virtual network to your subscription.

1. First create an XML file called **domainnet.xml** on your local host where you are running the PowerShell ISE with the following contents:

	````XML
	# Creates the domainvnet xml file
	```<?xml version="1.0" encoding="utf-8"?>
	<NetworkConfiguration xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/ServiceHosting/2011/07/NetworkConfiguration">
	  <VirtualNetworkConfiguration>
	    <Dns>
	      <DnsServers>
	        <DnsServer name="DC01" IPAddress="10.0.0.4" />
	      </DnsServers>
	    </Dns>
	    <VirtualNetworkSites>
	      <VirtualNetworkSite name="domainvnet" AffinityGroup="agdomain">
	        <AddressSpace>
	          <AddressPrefix>10.0.0.0/16</AddressPrefix>
	        </AddressSpace>
	        <Subnets>
	          <Subnet name="Subnet-1">
	            <AddressPrefix>10.0.0.0/24</AddressPrefix>
	          </Subnet>
	        </Subnets>
	        <DnsServersRef>
	          <DnsServerRef name="DC01" />
	        </DnsServersRef>
	      </VirtualNetworkSite>
	    </VirtualNetworkSites>
	  </VirtualNetworkConfiguration>
	</NetworkConfiguration>
	```

1. In the PowerShell ISE window, type the following command:

	````PowerShell
	# Creates the virtual network from XML file
	Set-AzureVNetConfig -ConfigurationPath _c:\yourpath_\domainvnet.xml

	> **Note:** Replace _c:\yourpath_ above with the path to where you saved the domainnet.xml file in the previous step.

1. Open a browser a go to [https://manage.windowsazure.com/](https://manage.windowsazure.com/). When prompted, login with your **Windows Azure** credentials. In the Windows Azure portal, click **Networks**, and then click **domainvnet**.  You can see the virtual network has been added that uses the affinity group you created earlier.


	![Verify The Virtual Network Creation](./images/verify-the-virtual-network-creation.png?raw=true "Verify The Virtual Network Creation")

	_Verify The Virtual Network Creation_

<a name="Exercise2" /></a>
### Exercise 2: Create a new virtual machine from the gallery image ###

You will now create a new virtual machine from a Windows Server 2012 gallery image called DC01.  This virtual machine will be used to create a domain controller in the next exercise. We will then create and provision a data disk which will be used in exercise 3 to place the AD database files.

Exercise 2 contains 2 tasks:

1. Create a new virtual machine
1. Configure a new data disk on DC01

<a name="Ex2Task1" /></a>
#### Task 1 - Create a new Virtual Machine with a data disk ####

1. In the PowerShell ISE window, type the following command:

	````PowerShell
	# Get a list of available gallery images
	Get-AzureVMImage  |  ft ImageName
	````

	![Retriving the VM images](./images/retriving-the-vm-images.png?raw=true "Retriving the VM images")

	_Retriving the virtual machine images_

	> **Note:** _The list of available image files is displayed. In this task, we will use the latest Windows Server 2012 image, which is a699494373c04fc0bc8f2bb1389d6106__Windows-Server-2012-Datacenter-201301.01-en.us-30GB.vhd._

1. In the PowerShell ISE window, type or copy the following commands.

	> **Note:** Make sure to replace NNNN in the $svcname variable below with the actual hosted service name you used earlier.

	````PowerShell
	# Defines image name
	$imgname = "a699494373c04fc0bc8f2bb1389d6106__Windows-Server-2012-Datacenter-201301.01-en.us-30GB.vhd"

	# Defines configuration settings
	$vmname = "DC01"
	$svcname = "NNNN"
	$password = "Passw0rd!"

	# Defines network settings
	$subnetname = "Subnet-1"

	# Defines VM configuration, including size, attached datadisk, and subnet 
	$dcvm = New-AzureVMConfig  -Name  $vmname  -ImageName  $imgname -InstanceSize "Small"  |
		Add-AzureProvisioningConfig  -Windows  -Password  $password  |
		Add-AzureDataDisk  -CreateNew  -DiskSizeInGB  10  -DiskLabel "ADdisk"  -LUN 0  |
		Set-AzureSubnet  -SubnetName  $subnetname

	# Creates new VM in existing hosted service
	New-AzureVM  -ServiceName  $svcname  -AffinityGroup 'agdomain' -VNetName 'domainvnet' -VMs  $dcvm
	````

	![Executing the powershell commands](./images/executing-the-powershell-commands.png?raw=true "Executing the powershell commands")

	_Executing the powershell commands_

	> **Note:** The **New-AzureVM** cmdlet will create a new hosted service when the -Location or -AffinityGroup parameter is specified, or will use an existing hosted service when those parameters are not specified. 

1. In the Windows Azure console, in the **Virtual Machines** section, wait a few minutes until the **DC01** virtual machine appears (or press F5 to refresh the display).

1. Click the **DC01** name.


	![Connecting to the DC01 VM](./images/connecting-to-the-dc01-vm.png?raw=true "Connecting to the DC01 VM")

	_Connecting to the DC01 virtual machine_

1. On the DC01 page, select the **Endpoints** tab. _Notice that by default the PowerShell cmdlet will add an RDP endpoint for port 3389. Therefore, we did not specify this endpoint configuration in the PowerShell commands._

<a name="Ex2Task2" /></a>
#### Task 2 - Configure a new data disk ####

1. In the Windows Azure console, in the **Virtual Machines** section, wait a few moments until the status of **DC01** is **Running**.

1. Select the **DC01** virtual machine, and then on the toolbar, click the **Connect** icon.

	![Connecting to the DC01 VM](./images/connecting-to-the-dc01-vm.png?raw=true "Connecting to the DC01 VM")

	_Connecting to the DC01 virtual machine_

1. Open the DC01.rdp file, and connect to the virtual machine. To log on, use credentials:

	| Field | Value |
	|--------|--------|
	| Account | **Administrator** |
	| Password | **Passw0rd!** |

	![Logging on to the DC01 VM](./images/logging-on-to-the-dc02-vm.png?raw=true "Logging on to the DC01 VM")

	_Logging on to the DC01 virtual machine_

1. On DC01, in **Server Manager**, on the **Tools** menu, click **Computer Management**. _The Computer Management console opens._

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
You have just created a base virtual machine called DC01, attached the necessary data disk, and provisioned the disk. We are going to login to DC01 to install and configure active directory and then verify the install was successful.

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
	Install-ADDSForest  -DomainName "contoso.com"  -SiteName "Default-First-Site-Name" -InstallDns:$true  -NoGlobalCatalog:$false  -Credential (Get-Credential)  -DatabasePath "F:\NTDS"  -LogPath "F:\NTDS"  -SysvolPath "F:\SYSVOL"  -NoRebootOnCompletion:$false  -Force:$true
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

1. At the **SafeModeAdministratorPassword** prompt and the **Confirm SafeModeAdministratorPassword** prompt, type **Passw0rd!**, and then press **Enter**. _The computer is promoted to domain controller. After a few moments, the DC01 VM will restart. You will lose the connection to the restarting VM._

	![Configuring the administrator password](./images/configuring-the-administrator-password.png?raw=true "Configuring the administrator password")

	_Configuring the administrator password_

<a name="Ex3Task3" /></a>
#### Task 3 - Verify the Domain Controller Installed Successfully ####

1. Wait two minutes for the DC01 VM to restart.
In the Azure portal, on the Virtual Machines page, select **DC01**, and on the toolbar, click the **Connect** icon.
	
	![Connecting to the DC01 VM](./images/connecting-to-the-dc01-vm.png?raw=true "Connecting to the DC01 VM")

	_Connecting to the DC01 virtual machine_

1. Open the DC01.rdp file, and connect to the virtual machine. To log on, use credentials:

	| Field | Value |
	|--------|--------|
	| User account | **Contoso\Administrator** |
	| Password | **Passw0rd!** |

	![Connecting to the VM](./images/connecting-to-the-vm.png?raw=true "Connecting to the VM")

	_Connecting to the virtual machine_

1. To verify that DC01 is working properly, run the following command:

	````PowerShell
	dcdiag.exe
	````

	_This output of the command confirms that DC01 was successfully promoted to domain controller._

<a name='Summary'/>
## Summary ##

In this lab, you walked through the steps of deploying a new Active Directory Domain using Windows Azure virtual machines and a simple virtual network. This lab also demonstrated how once in place virtual machines could be provisioned joined to the domain at boot time.
