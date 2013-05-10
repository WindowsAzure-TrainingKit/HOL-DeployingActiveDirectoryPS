<a name="Deploy-AD-in-Windows-Azure" />
# Deploy Active Directory in Windows Azure using PowerShell#

---
<a name="Overview" /></a>
## Overview ##

In this lab, you will provision a newly created Windows Server 2012 Virtual Machine called DC01 in Windows Azure using the Windows Azure PowerShell Cmdlets and then deploy Active Directory using Server Manager on DC01. DC01 will be the first domain controller in a new forest.

When deploying Active Directory in Windows Azure, two aspects are important to point out.

The first one is the networking configuration. Domain members and domain controllers need to find the DNS server hosting the domain DNS information. You will use the Azure network configuration, to set up the DNS service.

Secondly, it is important to prevent Active Directory database corruption. Active Directory assumes that it can write its database updates directly to disk. That means that you should place the Active Directory database files on a data disk that does not have write caching enabled.

<a name="Objectives" /></a>
### Objectives ###

In this hands-on lab, you will learn how to:

- Provision a data disk to a Virtual Machine
- Deploy a Domain Controller in Windows Azure

<a name="Prerequisites"></a> 
### Prerequisites ###

The following is required to complete this hands-on lab:

- [Windows PowerShell 2.0][1]
- [Windows Azure PowerShell CmdLets][2]
- A Windows Azure subscription - [sign up for a free trial](http://aka.ms/WATK-FreeTrial)

[1]: http://microsoft.com/powershell/
[2]: http://msdn.microsoft.com/en-us/library/windowsazure/jj156055

Additionally, you must complete the _Provisioning a Windows Azure Virtual Machine (PowerShell)_ HOL

>**Note:** In order to run through the complete hands-on lab, you must have network connectivity. 

<a name="Setup" />
### Setup ###

In order to execute the exercises in this hands-on lab you need to set up your environment.

1. Open Windows Explorer and browse to the lab's **Source** folder.

1. Execute the **Setup.cmd** file with Administrator privileges to launch the setup process that will configure your environment.

1. If the User Account Control dialog is shown, confirm the action to proceed.

> **Note:** Make sure you have checked all the dependencies for this lab before running the setup. 


<a name='gettingstarted' /></a>
### Getting Started: Obtaining Subscription's Credentials ###

In order to complete this lab, you will need your subscription’s secure credentials. Windows Azure lets you download a Publish Settings file with all the information required to manage your account in your development environment.

<a name='GSTask1' /></a>
#### Task 1 - Downloading and Importing a Publish Settings file ####

> **Note:** If you have done these steps in a previous lab on the same computer you can move on to Exercise 1.


In this task, you will log on to the Windows Azure Portal and download the Publish Settings file. This file contains the secure credentials and additional information about your Windows Azure Subscription that you will use in your development environment. Therefore, you will import this file using the Windows Azure Cmdlets in order to install the certificate and obtain the account information.

1.	Open Internet Explorer and browse to <https://windows.azure.com/download/publishprofile.aspx>.

1.	Sign in using the **Microsoft Account** associated with your **Windows Azure** account.

1.	**Save** the Publish Settings file to your local file system.

	![Downloading publish-settings file](Images/downloading-publish-settings-file.png?raw=true 'Downloading publish-settings file')

	_Downloading Publish Settings file_

	> **Note:** The download page shows you how to import the Publish Settings file using the Visual Studio Publish box. This lab will show you how to import it using the Windows Azure PowerShell Cmdlets instead.

1. Search for **Windows Azure PowerShell** in the Start screen and choose **Run as Administrator**.

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


1.	The following script imports your Publish Settings file and generates an XML file with your account information. You will use these values during the lab to manage your Windows Azure Subscription. Replace the placeholder with the path to your Publish Setting file and execute the script.

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

1. If the preceding command do NOT return a storage account, you should create one first.
  
	1. Run the following command to determine the data center to create your storage account in. Ensure you pick a data center that shows support for PersistentVMRole. 

		````PowerShell
		Get-AzureLocation  
		````

	1. Create your storage account: 


		````PowerShell
		New-AzureStorageAccount -StorageAccountName '[YOUR-STORAGE-ACCOUNT]' -Location '[DC-LOCATION]'
		````

1. Execute the following command to set your current storage account for your subscription.

	<!-- mark:1 -->
	````PowerShell
	Set-AzureSubscription -SubscriptionName '[YOUR-SUBSCRIPTION-NAME]' -CurrentStorageAccount '[YOUR-STORAGE-ACCOUNT]'
	````

<a name="Exercises" /></a>
## Exercises ##

This hands-on lab includes the following exercises:

1. [Add a new data disk to the virtual machine](#Exercise1)
1. [Deploy a new domain controller in Windows Server 2012](#Exercise2)

<a name="Exercise1" /></a>
### Exercise 1: Add a new data disk to the virtual machine ###

You will now modify the virtual machine you already created from the "Provisioning a Windows Azure Virtual Machine (PowerShell)" lab. We will call this VM DC01. We will create and provision a data disk to this existing VM which will be used in exercise 2 to place the AD database files.

Exercise 1 contains 2 tasks:

1. Attach a data disk to DC01
1. Configure a new data disk on DC01

<a name="Ex1Task1" /></a>
#### Task 1 - Attach a data disk to DC01####

1. Start **Windows Azure PowerShell**.

1. Run the following command to add a data disk to the existing virtual machine. Make sure you replace the plaholder accordingly, using the service name you provided when creating the virtual machine.

	````PowerShell
	$cloudSvcName = '[YOUR-SERVICE-NAME]'
	$vmname = 'DC01'

	Get-AzureVM -Name $vmname -ServiceName $cloudSvcName |
		Add-AzureDataDisk -CreateNew -DiskSizeInGB 10 -DiskLabel 'DC01-data' -HostCaching 'None' -LUN 0 |
		Update-AzureVM 
	````

	>**Note:** Notice the HostCaching option set to None. For use with the Active Directory database files, we need to use a data disk without caching. 

	![Adding data disk](./Images/add-data-disk.png?raw=true "Adding data disk")

	_Adding data disk_

<a name="Ex1Task2" /></a>
#### Task 2 - Configure a new data disk on DC01####

1. In the **Virtual Machines** section of the Windows Azure portal, select the **DC01** virtual machine, and then on the toolbar, click the **Connect** icon to connect using **Remote Desktop Connection**.

	![Connecting to the DC01 VM](./Images/connecting-to-the-dc01-vm.png?raw=true "Connecting to the DC01 VM")

	_Connecting to the DC01 virtual machine_

1. Open the DC01.rdp file, and connect to the virtual machine.

	>**Note:** use the credentials that you inserted when creating the virtual machine.

1. On DC01, in **Server Manager**, on the **Tools** menu, click **Computer Management**. The Computer Management console will open.

	![Opening the Computer Manager console](./Images/opening-the-computer-manager-console.png?raw=true "Opening the Computer Manager console")

	_Opening the Computer Manager console_

1. In the Computer Management console, in the left pane, select **Disk Management**. Disk Management recognizes that a new initialize disk is added to the computer, and it will show the Initialize Disk dialog box.

	![Selecting Disk Management](./Images/selecting-disk-management.png?raw=true "Selecting Disk Management")

	_Selecting Disk Management_

1. In the Initialize Disk dialog box, click **OK**.

	![Initializing the disk 2](./Images/initializing-the-disk-2.png?raw=true "Initializing the disk 2")

	_Initializing the disk 2_

1. On Disk 2, right-click the **Unallocated** space, and then click **New Simple Volume**.

	![Formating the unallocated space](./Images/formating-the-unallocated-space.png?raw=true "Formating the unallocated space")

	_Formating the unallocated space_

1. In the **New Simple Volume Wizard**, click **Next**.

	![Using the Simple Volume Wizard](./Images/using-the-simple-volume-wizard.png?raw=true "Using the Simple Volume Wizard")

	_Using the Simple Volume Wizard_

1. On the **Specify Volume Size** page, click **Next**. 

	>**Note**: This means that the entire available space (10237 MB) will become a new volume.

	![Specifing the volume size](./Images/specifing-the-volume-size.png?raw=true "Specifing the volume size")

	_Specifing the volume size_

1. On the **Assign Drive Letter or Path** page, ensure that the drive letter **F** is selected, and then click **Next**.

	![Assigning the drive letter](./Images/assigning-the-drive-letter.png?raw=true "Assigning the drive letter")

	_Assigning the drive letter_

1. On the **Format Partition** page, in the **Volume Label** text box, type **AD DS Data**, and then click **Next**.

	![Specifing the volume label](./Images/specifing-the-volume-label.png?raw=true "Specifing the volume label")

	_Specifing the volume label_

1. On the **Completing the New Simple Volume Wizard** page, click **Finish**. _Windows will quick format the disk, and assign it the drive letter F_.

	![Completing the wizard](./Images/completing-the-wizard.png?raw=true "Completing the wizard")

	_Completing the wizard_

	>**Note:** if you are prompted to format the new AD DS Data disk, click **OK** in the dialog box and format the disk as NTFS.

1. Close the Computer Management console.

<a name="Exercise2" /></a>
### Exercise 2: Deploy a new domain controller in Windows Server 2012 ###
You have just created a base virtual machine called DC01, attached the necessary data disk, and provisioned the disk. We are going to login to DC01 to install and configure active directory and then verify the install was successful.

Exercise 2 contains 3 tasks:

1. Install the Active Directory Domain Services Role 
1. Configure the Active Directory Domain Services Role
1. Verify the Domain Controller Installed Successfully

<a name="Ex2Task1" /></a>
#### Task 1 - Install the Active Directory Domain Services Role ####

1. Inside the remote desktop connection to DC01, open a PowerShell window, and type the following command:

	````PowerShell
	Add-WindowsFeature -Name AD-Domain-Services  -IncludeManagementTools
	````

	![Adding the AD feature](./Images/adding-the-ad-feature.png?raw=true "Adding the AD feature")

	_Windows is installing the Active Directory Domain Services role_

<a name="Ex2Task2" /></a>
#### Task 2 - Configure the Active Directory Domain Services Role ####
1. When the feature installation is completed, type the following single command to promote the domain controller:

	````PowerShell
	Install-ADDSForest  -DomainName "contoso.com" -InstallDns:$true  -DatabasePath "F:\NTDS"  -LogPath "F:\NTDS"  -SysvolPath "F:\SYSVOL"  -NoRebootOnCompletion:$false  -Force:$true
	````

	>**Note:** The C: disk is the OS disk, and has caching enabled. The Active Directory database should not be stored on a disk that has write caching enabled. The F: disk is the data disk that we added earlier, and does not have this feature enabled.

	![Promoting the domain controller with powershell ](./Images/promoting-the-domain-controller-with-powershell.png?raw=true "Promoting the domain controller with powershell")

1. At the **SafeModeAdministratorPassword** prompt and the **Confirm SafeModeAdministratorPassword** prompt, type **Passw0rd!**, and then press **Enter**. 

	>**Note:** The computer is promoted to domain controller. After a few moments, the DC01 Virtual Machine will restart. You will lose the connection to the restarting Virtual Machine.

	![Configuring the administrator password](./Images/configuring-the-administrator-password.png?raw=true "Configuring the administrator password")

	_Configuring the administrator password_

<a name="Ex2Task3" /></a>
#### Task 3 - Verify the Domain Controller Installed Successfully ####

1. Wait two minutes for the DC01 Virtual Machine to restart.
In the Azure portal, on the Virtual Machines page, select **DC01**, and on the toolbar, click the **Connect** icon.
	
	![Connecting to the DC01 VM](./Images/connecting-to-the-dc01-vm.png?raw=true "Connecting to the DC01 VM")

	_Connecting to the DC01 virtual machine_

1. Open the DC01.rdp file, and connect to the virtual machine.

	>**Note:** use the credentials that you inserted when creating the virtual machine.

1. To verify that DC01 is working properly, open a Powershell window, and run the following command:

	````PowerShell
	dcdiag.exe
	````

	>**Note:** The output of the command confirms that DC01 was successfully promoted to domain controller

<a name='Summary'/>
## Summary ##

In this lab, you went through the steps of deploying a new Active Directory Domain controller in a new forest using Windows Azure virtual machines.
