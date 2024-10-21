# Secure apps by requiring assignment

> Completed, could use wordsmithing

Some applications, such as Azure CLI tools, AzureAD PowerShell, Graph PowerShell, and Exchange PowerShell, are common targets for abuse. Most users will never need access to these, and some users may only need occasional access.

The purpose of this Access Package is to remove default access and provide a way for those who don't always need access to request it when they do.

## 1. Get or create the security group

This example uses a group to allow access to powerful apps that we will block by default for most users.

> !WARNING
> Only create one group at a time. Complete the example, then come back and do it again for a different group. Some applications may be registered by default while others aren't. You can find the list of well-known Microsoft applications and their application IDs here: [Application IDs of commonly used Microsoft applications](https://learn.microsoft.com/en-us/troubleshoot/azure/entra/entra-id/governance/verify-first-party-apps-sign-in)  

To create a new group for Azure CLI:
```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-allow-azure-cli' -MailEnabled:$False  -MailNickName 'em-allow-azure-cli' -SecurityEnabled).Id

$appId = "04b07795-8ddb-461a-bbee-02f9e1bf7b46"

```

To create a new group for Azure PowerShell:
```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-allow-azure-powershell' -MailEnabled:$False  -MailNickName 'em-allow-azure-powershell' -SecurityEnabled).Id

$appId = "1950a258-227b-4e31-a9cf-717495945fc2"

```

To create a new group for AzureAD PowerShell:
```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-allow-azure-ad-powershell' -MailEnabled:$False  -MailNickName 'em-allow-azure-ad-powershell' -SecurityEnabled).Id

$appId = "1b730954-1685-4b74-9bfd-dac224a7b894"

```

To create a new group for Exchange PowerShell:
```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-allow-exchange-powershell' -MailEnabled:$False  -MailNickName 'em-allow-exchange-powershell' -SecurityEnabled).Id

$appId = "fb78d390-0c51-40cd-8e17-fdbfab77341b"

```

To create a new group for Graph PowerShell/CLI Tools:
```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-allow-graph-powershell-cli' -MailEnabled:$False  -MailNickName 'em-allow-graph-powershell-cli' -SecurityEnabled).Id

$appId = "14d82eec-204b-4c2f-b7e8-296a70dab67e"

```

To use an existing group and appId:
```powershell
Connect-MgGraph -Scopes Group.Read.All

$name = "GroupName"
$groupId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id
$appId = "8efb8aad-969b-4992-bb5f-f86cb24faa5f"

```

---

## 2. Require assignment on app and add group

This will register the application to your tenant if it has not already been regtistered, then it will set it to require assigment and assign the security group we just created.

```powershell
Connect-MgGraph -Scopes Application.ReadWrite.All,AppRoleAssignment.ReadWrite.All

New-MgServicePrincipal -AppId $appId -ErrorAction SilentlyContinue

$sp = Get-MgServicePrincipal -Filter "appId eq '$appId'"

$appRoleId = ($sp.AppRoles | Out-GridView -PassThru).Id

Update-MgServicePrincipal -ServicePrincipalId $sp.Id -AppRoleAssignmentRequired:$true

$params = @{
  appRoleId = $appRoleId
  resourceId = $sp.Id
  principalId = $groupId
}
New-MgBetaServicePrincipalAppRoleAssignedTo -ServicePrincipalId $sp.Id -BodyParameter $params

```

---

## 3. Add the group to the catalog

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$catalogId = (Get-MgBetaEntitlementManagementAccessPackageCatalog -Filter "DisplayName eq 'General'").Id

$params = @{
  catalogId = "$catalogId"
  requestType = "AdminAdd"
  justification = ""
  accessPackageResource = @{
    resourceType = "AadGroup"
    originId = "$groupId"
    originSystem = "AadGroup"
  }
}
New-MgBetaEntitlementManagementAccessPackageResourceRequest -BodyParameter $params

$resourceId = (Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageResource -AccessPackageCatalogId $catalogId -Filter "originId eq '$groupId'").Id

```

---

## 4. Create the Access Package

> !NOTE
> Remember to change the display name to match the application 

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "Allow Azure CLI"
  description = "Allows users to access the Azure CLI application"
}
$packageId = (New-MgBetaEntitlementManagementAccessPackage -BodyParameter $params).Id

```

---

## 5. Add the group to the access package

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$role = @{
  originId = "Member_$groupId"
  displayName = "Member"
  originSystem = "AadGroup"
  accessPackageResource = @{
    id = "$resourceId"
    resourceType = "Security Group"
    originId = "$groupId"
    originSystem = "AadGroup"
  }
}
$scope = @{
  originId = "$groupId"
  originSystem = "AadGroup"
}
New-MgBetaEntitlementManagementAccessPackageResourceRoleScope -AccessPackageId $packageId -AccessPackageResourceRole $role -AccessPackageResourceScope $scope

```

---

## 5. Create access package policies

For this example, we might want a couple of policies - one for those who should always have access and another that must activate their access.

> !NOTE
> The groups in the code below must already exist. If they do not, you will need to create them.

Create a policy that always provides access:
```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$name = "it-developers"
$requestorsId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

$params = @{
  accessPackageId = "$packageId"
  displayName = "Members of $name"
  description = "Members of $name can request assignment"
  accessReviewSettings = $null
  requestorSettings = @{
    scopeType = "SpecificDirectorySubjects"
    acceptRequests = $true
    allowedRequestors = @(
      @{
        "@odata.type" = "#microsoft.graph.groupMembers"
        isBackup = $false
        id = "$requestorsId"
        description = "$name"
      }
    )
  }
  requestApprovalSettings = @{
    isApprovalRequired = $false
    isApprovalRequiredForExtension = $false
    isRequestorJustificationRequired = $false
    approvalMode = "NoApproval"
    approvalStages = @()
  }
}
New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params

```

Create a policy that provides access for 12 hours:
```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$name = "azure-consultants"
$requestorsId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

$params = @{
  accessPackageId = "$packageId"
  displayName = "Members of $name"
  description = "Members of $name can request assignment"
  accessReviewSettings = $null
  expiration = @{
    type = "afterDuration"
    duration = "PT12H"
  }
  requestorSettings = @{
    scopeType = "SpecificDirectorySubjects"
    acceptRequests = $true
    allowedRequestors = @(
      @{
        "@odata.type" = "#microsoft.graph.groupMembers"
        isBackup = $false
        id = "$requestorsId"
        description = "$name"
      }
    )
  }
  requestApprovalSettings = @{
    isApprovalRequired = $false
    isApprovalRequiredForExtension = $false
    isRequestorJustificationRequired = $false
    approvalMode = "NoApproval"
    approvalStages = @()
  }
}
New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params

```