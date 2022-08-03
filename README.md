
# Azure Virtual Desktop with FSLogix

Deploy a fully functioning AVD (session and remote apps) with FSlogix for profiles and on prem domain controller 

## Step 1: Create a Domain Controller

1. Create a Virtual Network called AVDVnet
2. Create a Windows VM in Azure with a Windows Server 2019 image.
3. Go to Azure Active Directory and create a global admin avdga@<domain>.onmicrosoft.com. This admin credential will be required to set up Azure AD Connect. 
4. On the VM open Server Manager > Add roles and features > Server Roles > AD DS
5. Promote to DC from notifications > Add new forest > Enter Domain name ex. <domain>.onmicrosoft.com > restart
6. Install AD Connect > Configure Express 
7. Open Active Directory Users and Computers > Create new user who will be the domain controller admin and keep password never expires - avdadmin@<domain>.onmicrosoft.com
   Double click on newly created user > Add member of > Domain Admin and Enterprise Admin 
8. Install AD Connect with Express Settings and log in using Azure AD global admin from step 3.
9. Create more users and a new container Groups > add a new Group called AVDUsers and add all users from step 8.
10. Wait for all users and groups to sync to Azure AD

## Step 2: Add Groups in Azure AD

1. Add new groups like PoolUsers and RemoteAppUsers and assign some users that you created in DC and synced to AD previously

## Step 3: Set up storage account for FSLogix file share

ref: https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-ad-ds-enable

1. Create a storage account 
2. Create a new file share called "avdfs"
3. Go back to DC and create a new OU called Storage under Active Directory Users and Computers
4. Follow https://docs.microsoft.com/en-us/azure/storage/files/storage-files-identity-ad-ds-enable and download AzFleshybrid module
5. Cd to the AzFilesHybrid path and run the following powershell script on Windows Powershell ISE

```bash
# Change the execution policy to unblock importing AzFilesHybrid.psm1 module
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser

# Navigate to where AzFilesHybrid is unzipped and stored and run to copy the files into your path
.\CopyToPSPath.ps1 

# Import AzFilesHybrid module
Import-Module -Name AzFilesHybrid

Install-Module Az

# Login with an Azure AD credential that has either storage account owner or contributor Azure role assignment
Connect-AzAccount (do not use DC admin credentials here user subscription contributor or owner that you use to log into azure portal)

# Define parameters
$SubscriptionId = "<your-subscription-id-here>"
$ResourceGroupName = "<resource-group-name-here>"
$StorageAccountName = "<storage-account-name-here>"
$DomainName = "<domain-name-here>"

# Select the target subscription for the current session
Select-AzSubscription -SubscriptionId $SubscriptionId 


# Select the target subscription for the current session
Select-AzSubscription -SubscriptionId $SubscriptionId 


Join-AzStorageAccount `
        -ResourceGroupName $ResourceGroupName 
        -StorageAccountName $StorageAccountName 
        -DomainAccountType "ComputerAccount"
        -OrganizationUnitName "Storage"
        -Domain $DomainName


#Test
# List the directory service of the selected service account
$storageAccount.AzureFilesIdentityBasedAuth.DirectoryServiceOptions

# List the directory domain information if the storage account has enabled AD DS authentication for file shares
$storageAccount.AzureFilesIdentityBasedAuth.ActiveDirectoryProperties


```
6. Refresh Active Directory Users and Computers and check the storage AU to see if file share is connected


## Step 4: Set up file share permissions in DC 

ref: https://docs.microsoft.com/en-us/azure/virtual-desktop/fslogix-profile-container-configure-azure-files-active-directory?tabs=adds
   
1. Under Active Directory Users and Computers > Add new groups > FS Elevated Contributor (DC admin avdadmin@<domain>.onmicrosoft.com) and FS Contributor (AVDUsers group
   i.e. all users)
2. After these new groups are synced to Azure AD go to the azure storage account > file share > IAM > Add roles > FS Contributor as Storage File Data SMB Share Contributor and
   FS Elevated Contributor as Storage File Data SMB Share Elevated Contributor
3. Log into DC using the avdadmin credential as it is part of Elevated Contributor group and is the only one that can create/mount share

```powershell

#mount file share on DC, get the access keys for storage account
net use z: \\<storage-account-name-here>.file.core.windows.net\avdfs <key1> /user:Azure\<storage-account-name-here>

```
4. Configure NTFS permission by clicking on newly created drive properties > security > Advanced and add FS Contributor and FS Elevated Contributor with full control permissions.
   Also add AVDUsers group so everyone has access to the file share.
5. Create a new folder called "Profiles"
6. Assign permission to this folder by disabling all inhertied share level permission and add > Creater Owner, FS Elevated Contributor with Full Control, AVDUsers with create folders/append data 
   permission along with the rest
7. Profile folders should now sync with Azure File Share
   
## Step 5 : Create a custom image for host pool

1. Create a new VM with Windows 10 multisession + M365 Apps
2. Install a custom application something like 7Zip
3. Install FSLogix Agent > Run the FSLogixAppsSetup
4. Open Registry Editor > Local Machine key > FSLogix > Profile > Create a new DWord named Enabled with value 1 and create a Multi-string value named "VHDLocations" with
   value "\\<storage-account-name>.file.core.windows.net\avdfs" (These commands enable Profile Container and configure the location of the share.)
5. Generalize image by going to C: > Windows > System32 > Sysprep > run sysprep as admin and select "Generalize" and Shutdown options as "Shut Down"
6. Once the VM shuts down go to Azure portal and stop the VM first to change status to Stopped and Deallocated.
7. Click on Capture and save image as managed image for now if you don't have a gallery 
   
## Step 6 : Create Host pool and Application Groups

1. Go to DC on azure portal and navigate to the VNET it is in (AVDVnet). Go to DNS Servers and change the Setting from Default to Custom. Add the IP (private of the DC) as a server
2. Look up AVD and create a host pool of type "Pooled" and load-balancing either Depth/Bread First
3. Under Virtual Machines add new VMs and use the image saved from Step 5. Ensure that you enter the right credentials for DC admin avdadmin@<domain>.onmicrosoft.com and
   choose the same vnet as that of the DC (AVDVnet)
4. Create a new Workspace

## Step 7 : Set up applications and permissions

1. Under the Pool Application Group > Create new Remote Desktop App Group > Add some apps like Excel/Paint
2. Under Application Group > Assignments add users to each of the application groups (Groups like PoolUsers and RemoteAppUsers, you can also assign individual users)

## Step 8: Test the AVD environment for different users and profile

Choose from https://docs.microsoft.com/en-us/azure/virtual-desktop/user-documentation/connect-windows-7-10

1. Navigate to web client https://rdweb.wvd.microsoft.com/arm/webclient and log in using different users 
2. You can navigate to Azure File share and check if different profiles were created for each user as well as sessions under application groups

# Credits
   https://www.youtube.com/watch?v=XhaCJ6JPM4Y&ab_channel=ITClouds9
   
