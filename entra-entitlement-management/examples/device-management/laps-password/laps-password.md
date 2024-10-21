# Send LAPS password

This example allows specific users to request the LAPS password for their devices and will send it to them via Teams.

## 1. Create and update the Logic App

> !IMPORTANT
> Microsoft performs hardening of Logic Apps deployed through Entitlement management. It is recommended to create these from the catalog rather than an ARM template.

To create the Logic App, go under the Catalog - Custom Extensions, and click Add a custom extension. During the wizard, select "Request workflow", "Launch and Continue", and then fill in the details to create the logic app and complete the wizard.

Once the Logic App has been provisioned, you can copy the JSON from the link below and paste into the Logic App code view. There will be errors due to the API connections (authentication to Teams), and you'll have to update it.

[Logic App code](laps-password-la.json)

Once complete, run the following and select the custom extension you created to get the custom extension extension ID:

```powershell
$customExtensionId = (Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageCustomWorkflowExtension -AccessPackageCatalogId $catalogId | Out-GridView -PassThru).Id

```

---

## 2. Granting the Logic App permissions

The next step is to enable the Managed Identity on the Logic App and grant it the permissions needed to create a Temporary Access Pass for the user.

On the Logic App, go under Settings - Identity and enable the System Assigned Managed Identity. Copy the object ID, update $MIObjectId, and run the below script to grant it permissions to create read the user's devices and the LAPS passwords.

```powershell
$MIObjectId = "c449c80d-0d9c-4e04-b340-00dcf7e2d878"
$permissions = "User.Read.All","DeviceLocalCredential.Read.All"

Connect-MgGraph -Scopes AppRoleAssignment.ReadWrite.All
$permissions | ForEach-Object {
    $PermissionName = $_
    $GraphSP = Get-MgServicePrincipal -Filter "startswith(DisplayName,'Microsoft Graph')" | Select-Object -first 1 #Graph App ID: 00000003-0000-0000-c000-000000000000
    $AppRole = $GraphSP.AppRoles | Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
    New-MgServicePrincipalAppRoleAssignment -AppRoleId $AppRole.Id -ServicePrincipalId $MIObjectId -ResourceId $GraphSP.Id -PrincipalId $MIObjectId
}

```

---

## 3. Create the Access Package

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "Get LAPS password"
  description = "Gets the LAPS passwords for owned devices"
}
$packageId = (New-MgBetaEntitlementManagementAccessPackage -BodyParameter $params).Id

```

---

## 5. Create the access package policy

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$name = "it-all"
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
  customExtensionStageSettings = @(
        @{
            stage = "assignmentRequestCreated"
            customExtension = @{
                "@odata.type" = "#microsoft.graph.accessPackageAssignmentRequestWorkflowExtension"
                id = "$customExtensionId"
            }
        }
  )
}
New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params

# Update expiration (not supported on beta API)

$params = @{
  id = "$packageId"
  displayName = "Members of $name"
  description = "Members of $name can request assignment"
  allowedTargetScope = "specificDirectoryUsers"
  automaticRequestSettings = $null
  canExtend = $true
  specificAllowedTargets = @(
    @{
        "@odata.type" = "#microsoft.graph.groupMembers"
        groupId = "$requestorsId"
        description = "$name"
      }
  )
  expiration = @{
    type = "afterDuration"
    duration = "PT1H"
  }
  requestorSettings = @{
    enableTargetsToSelfAddAccess = $true
    enableTargetsToSelfUpdateAccess = $false
    enableTargetsToSelfRemoveAccess = $true
    allowCustomAssignmentSchedule = $false
    enableOnBehalfRequestorsToAddAccess = $false
    enableOnBehalfRequestorsToUpdateAccess = $false
    enableOnBehalfRequestorsToRemoveAccess = $false
    onBehalfRequestors = @()
  }
  requestApprovalSettings = @{
    isApprovalRequiredForAdd = $false
    isApprovalRequiredForUpdate = $false
    stages = @()
  }
  accessPackage = @{
    id = "$packageId"
  }
}

Invoke-MgGraphRequest -Uri "/v1.0/identityGovernance/entitlementManagement/assignmentPolicies/$($policy.Id)" -Method PUT -Body $params

```
