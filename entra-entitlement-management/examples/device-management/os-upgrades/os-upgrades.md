# OS Upgrade Pilot

This example allows users to opt-in to OS upgrades and can also be used to force upgrades through policy. It utilizes a Logic App to get the user's devices and adds them to the security group used to target a Feature update policy.

## 1. Create the security group

This example uses a group to enable devices for Feature Updates.

> !IMPORTANT
> The Logic App created later will make calls to add devices to a groupId that is part of the URI. Take note of this groupID as we will need it later.

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-os-upgrades' -MailEnabled:$False  -MailNickName 'em-os-upgrades' -SecurityEnabled).Id

Write-Output "The following group ID will be needed for updating the Logic App: $groupId"

```

---

## 2. Create and update the Logic App

> !IMPORTANT
> Microsoft performs hardening of Logic Apps deployed through Entitlement management. It is recommended to create these from the catalog rather than an ARM template.

To create the Logic App, go under the Catalog - Custom Extensions, and click Add a custom extension. During the wizard, select "Request workflow", "Launch and Continue", and then fill in the details to create the logic app and complete the wizard.

Once the Logic App has been provisioned, you can copy the JSON from the link below and paste into the Logic App code view. There will be errors due to the API connections (authentication to Teams), and you'll have to update it.

[Logic App code](os-upgrades-la.json)



Once complete, run the following and select the custom extension you created to get the custom extension extension ID:

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$catalogId = (Get-MgBetaEntitlementManagementAccessPackageCatalog -Filter "DisplayName eq 'General'").Id

$customExtensionId = (Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageCustomWorkflowExtension -AccessPackageCatalogId $catalogId | Out-GridView -PassThru).Id

```

---

## 2. Granting the Logic App permissions

The next step is to enable the Managed Identity on the Logic App and grant it the permissions needed to create a Temporary Access Pass for the user.

On the Logic App, go under Settings - Identity and enable the System Assigned Managed Identity. Copy the object ID, update $MIObjectId, and run the below script to grant it permissions to create read the user's devices and the LAPS passwords.

```powershell
$MIObjectId = "758fb58a-58a8-48dd-989c-f89f03a8e1c9"
$permissions = "User.Read.All","GroupMember.ReadWrite.All","Device.ReadWrite.All"

Connect-MgGraph -Scopes AppRoleAssignment.ReadWrite.All
$permissions | ForEach-Object {
    $PermissionName = $_
    $GraphSP = Get-MgServicePrincipal -Filter "startswith(DisplayName,'Microsoft Graph')" | Select-Object -first 1 #Graph App ID: 00000003-0000-0000-c000-000000000000
    $AppRole = $GraphSP.AppRoles | Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
    New-MgServicePrincipalAppRoleAssignment -AppRoleId $AppRole.Id -ServicePrincipalId $MIObjectId -ResourceId $GraphSP.Id -PrincipalId $MIObjectId
}

```

---

## 4. Create the Access Package

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "OS Upgrade Pilot"
  description = "Allows users to opt-in for OS upgrades"
}
$packageId = (New-MgBetaEntitlementManagementAccessPackage -BodyParameter $params).Id

```

---

## 5. Create access package policies

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$name = "it-admins"
$approverId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

$params = @{
  accessPackageId = "$packageId"
  displayName = "All Directory Members"
  description = "Any non-guest user can make a request"
  accessReviewSettings = $null
  requestorSettings = @{
    scopeType = "AllExistingDirectoryMemberUsers"
    acceptRequests = $true
    allowedRequestors = @()
    allowCustomAssignmentSchedule = $false
  }
  requestApprovalSettings = @{
    approvalMode = "SingleStage"
    isApprovalRequired = $true
    isApprovalRequiredForExtension = $false
    isRequestorJustificationRequired = $false
    approvalStages = @(
      @{
        "@odata.type" = "#microsoft.graph.approvalStage"
        approvalStageTimeOutInDays = "14"
        isApproverJustificationRequired = $false
        isApproverAllowedToModifyRequest = $false
        isEscalationEnabled = $false
        isApproverInformationVisibleToRequestor = $false
        escalationTimeInMinutes = "0"
                primaryApprovers = @(
                    @{
                        "@odata.type" = "#microsoft.graph.groupMembers"
                        displayName = "$name"
                        objectId = "$approverId"
                        isBackup = $true
                    }
                    @{
                        "@odata.type" ="#microsoft.graph.requestorManager"
                        isBackup = $false
                        managerLevel = "1"
                    }
                )
        escalationApprovers = @()
      }
    )
    skipManagerApprovalForManagerAsRequestor = $false
  }
  customExtensionStageSettings = @(
        @{
            stage = "assignmentRequestApproved"
            customExtension = @{
                "@odata.type" = "#microsoft.graph.accessPackageAssignmentRequestWorkflowExtension"
                id = "$customExtensionId"
            }
        }
  )
}
New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params

```

---
