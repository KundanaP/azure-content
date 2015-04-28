<properties
   pageTitle="Custom Script extension with Azure Resource Manager Templates"
   description="Using Custom Script Extension with templates"
   services="virtual-machines"
   documentationCenter=""
   authors="kundanap"
   manager="madhana"
   editor=""/>

<tags
   ms.service="virtual-machines"
   ms.devlang=""
   ms.topic="article"
   ms.tgt_pltfrm=""
   ms.workload="infrastructure-services"
   ms.date="04/28/2015"
   ms.author="kundanap"/>

# Using Custom Script Extension with templates
This article gives an overview of writing Azure Resource Manager Templates with Custom Script Extension for bootstrapping workloads on a Linux or a Windows VM.

For an overview of Custom Script Extension please refer to the article here <a href="http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-extensions-customscript" target="_blank">here</a>.

Ever since its launch Custom Script Extension has been used widely to configure workloads on both Windows and Linux VMs. With the introduction of Azure Resource Manager Templates, users can now create a single template that not only provisions the VM but also configures the workloads on it.

## Overview of Azure Resource Manager Templates.
Azure Resource Manager Template allow you to declaratively specify the Azure IaaS infrastructure  in Json language  by defining  the dependencies between resources. For a detailed overview of Azure Resource Manager Templates, please refer to the article here:

<a href="https://github.com/Azure/azure-content-pr/blob/release-build/articles/resource-group-overview.md" target="_blank">Template Overview</a>.
<a href="https://github.com/Azure/azure-content-pr/blob/release-build/articles/resource-group-authoring-templates.md" target="_blank">Authoring Templates</a>.
<a href="https://github.com/Azure/azure-content-pr/blob/release-build/articles/resource-group-advanced-template.md" target="_blank">Advanced Template Concepts</a>.
<a href="https://github.com/Azure/azure-content-pr/blob/release-build/articles/resource-group-template-functions.md" target="_blank">Template Functions</a>.


## Pre-Requistes for using Templates:
1. Access to the Azure Resource Manager APIs.

2. If you are using Azure Powershell, install and configure the latest PowerShell by following the instructions <a href="https://github.com/Azure/azure-content-pr/blob/release-build/articles/virtual-machines-deploy-rmtemplates-powershell.md" target="_blank">Powershell for Azure Resource Manager</a>.

3. If you are using Azure CLI, install and configure Azure CLI by following instructions <a href="https://github.com/Azure/azure-content-pr/blob/release-build/articles/virtual-machines-deploy-rmtemplates-powershell.md" target="_blank">Azure CLI for Templates</a>.

## Overview of Custom Script Extension with Templates:
The Custom Script for the templates is the same as the extension thats been used with Azure Service Management APIs. The extension supports the same parameters and scenarios like uploading files to Azure Storage account or Github location. The key difference with templates is the additional meta data thats required to define the extension in a template. Another key difference is specifying the exact version of the extension, as opposed to specifying the version in the format '1.*'. This changed has been added to ensure users are not accidentally auto-upgraded.

### Template Snippet for Custom Script Extension:

Here is the template fragment for using Custom Script Extension:
// Define the following variables in the 'Variables' section of the template

"variables": {
  "scriptUrl" : "https://storageaccountname.blob.core.windows.net/scripts/InstallApplication.ps1",
  "scriptName" : "InstallApplication.ps1",
  "customScriptExtensionVersion" : "1.4",
  "scriptArg" : "[parameters('applicationToInstall')]",
}

//Define the following resource in the Resource section of the template
{
            "type": "Microsoft.Compute/virtualMachines/extensions",
             "name": "[concat(parameters('uniqueDNSNameBase'),'/', variables('vmExtensionName'))]",
             "apiVersion": "2014-05-01-preview",
             "location": "[parameters('location')]",
              "dependsOn": [
               "[concat('Microsoft.Compute/virtualMachines/', parameters('uniqueDNSNameBase'))]"
             ],
             "properties": {
                 "publisher": "Microsoft.Compute",
                 "type": "CustomScriptExtension", 
                 "typeHandlerVersion": "[variables('customScriptExtensionVersion')]",
                 "settings":
                 {
                    "fileUris": ["[variables('scriptUrl')]"],
                    "commandToExecute": "[concat('powershell -ExecutionPolicy Unrestricted -file ',variables('scriptName'),' ',variables('scriptArg'))]"
                  }
           }
}


 ### Upload files to the default container:
If you have your scripts in the storage container of the default account of your subscription, then the cmdlet snippet below shows how you can run them on the VM. The ContainerName in the sample below is where you upload the scripts to. The default storage account can be verified by using the cmdlet ‘Get-AzureSubscription –Default’

Note: This use case creates a new VM but the same operations can be done on an existing VM as well.

    # create a new VM in Azure.
    $vm = New-AzureVMConfig -Name $name -InstanceSize Small -ImageName $imagename
    $vm = Add-AzureProvisioningConfig -VM $vm -Windows -AdminUsername $username -Password $password
    // Add Custom Script Extension to the VM. The container name refer to the storage container which contains the file.
    $vm = Set-AzureVMCustomScriptExtension -VM $vm -ContainerName $container -FileName 'start.ps1'
    New-AzureVM -ServiceName $servicename -Location $location -VMs $vm
    #  After the VM is created, the extension downloads the script from the storage location and executes it on the VM.

    # Viewing the  script execution output.
    $vm = Get-AzureVM -ServiceName $servicename -Name $name
    # Use the position of the extension in the output as index.
    $vm.ResourceExtensionStatusList[i].ExtensionSettingStatus.SubStatusList

### Upload files to a non default storage containers.

This use case shows how to use a non-default storage either within the same subscription or in a different subscription for uploading scripts/files. Here we’ll use an existing VM but the same operations can be done while creating a new VM.

        Get-AzureVM -Name $name -ServiceName $servicename | Set-AzureVMCustomScriptExtension -StorageAccountName $storageaccount -StorageAccountKey $storagekey -ContainerName $container -FileName 'file1.ps1','file2.ps1' -Run 'file.ps1' | Update-AzureVM
### Upload scripts to multiple containers across different storage accounts.
  If the script files are stored across multiple containers, then currently to run those scripts, you have to provide the full SAS URL of these files.

      Get-AzureVM -Name $name -ServiceName $servicename | Set-AzureVMCustomScriptExtension -StorageAccountName $storageaccount -StorageAccountKey $storagekey -ContainerName $container -FileUri $fileUrl1, $fileUrl2 -Run 'file.ps1' | Update-AzureVM

### Using Github location for downloading files.
Starting with version 1.4, Github URLs can be specified as input location for downloading files. These should be public URLs. In future versions we will add support for private URLs. The file URLs can be specified using the '-FileUri' option as specified above.

### Add Custom Script Extension from the Portal.
Browse to the Virtual Machine in the <a href="https://portal.azure.com/ " target="_blank">Azure Preview Portal </a> and add the Extension by specifying the script file to run.
  ![][5]

  ### UnInstalling Custom Script Extension.

Custom Script Extension can be uninstalled from the VM using the cmdlet below

      get-azureVM -ServiceName KPTRDemo -Name KPTRDemo | Set-AzureVMCustomScriptExtension -Uninstall | Update-AzureVM

For Linux Custom Script Extension, please refer to the documentation <a href="http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-script-lamp/" target="_blank">here</a>.

<!--Image references-->
[5]: ./media/virtual-machines-extensions-customscript/addcse.png
