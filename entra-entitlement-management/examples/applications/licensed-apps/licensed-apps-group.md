# Grant access to licensed applications via group

> Completed, probably need wordsmithing

This example covers scenarios where we have licensed apps in Entra that automatically provision accounts that incur billing. This can be used for non-licensed apps as well, but we might not need the same approval process as we would for licensed applications.

> !NOTE
> There are two ways to configure this scenario: 1) Group membership assigned to the app (this example), 2) [Directly assigned to the app](licensed-apps-app.md). You will need to use the group option if you want to pass the group as a claim via SSO.

## 1. Get or create the security group

This example uses a group to allow access to licensed applications.

> !NOTE
> Remember to adjust the name of the group to match your application

To create a new group:
```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-allow-docusign-access' -MailEnabled:$False  -MailNickName 'em-allow-docusign-access' -SecurityEnabled).Id

```

To use an existing group:
```powershell
Connect-MgGraph -Scopes Group.Read.All

$name = "GroupName"
$groupId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

```

---

## 2. Require assignment on app and add group

This will search for your directory for the name you put in and present a GUI For you to select the correct application in the event there are more than one with that name (like dev, prod, etc.).

```powershell
Connect-MgGraph -Scopes Application.Read.All,AppRoleAssignment.ReadWrite.All

$name = "Docusign"

$spId = (et-MgBetaServicePrincipal -Search "displayName:$name" -ConsistencyLevel eventual -CountVariable $count | Out-GridView -PassThru

$appRoleId = ($sp.AppRoles | Out-GridView -PassThru).Id

Update-MgBetaServicePrincipal -ServicePrincipalId $sp.Id -AppRoleAssignmentRequired:$true

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
  displayName = "Docusign Access - Group"
  description = "Allows users to access Docusign"
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

For this example, we're going to look at how we can have multiple approval stages, first stage for the manager with fallback to a group you will need to define and a second stage for finance to sign off.

> !NOTE
> The fallback group and finance group in the code below must already exist. If you don't have existing groups you can use, you will need to create them first.

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$fallbackName = "legal-managers"
$fallbackApproverId = (Get-MgBetaGroup -Filter "DisplayName eq '$fallbackName'").Id

$secondStageName = "finance-approvers"
$secondStageApproverId = (Get-MgBetaGroup -Filter "DisplayName eq '$secondStageName'").Id

$params = @{
  accessPackageId = "$packageId"
  displayName = "All Directory Members"
  description = "Any non-guest user can make a request"
  accessReviewSettings = $null
  requestorSettings = @{
    scopeType = "AllExistingDirectoryMemberUsers"
    acceptRequests = $true
    allowedRequestors = @()
  }
  requestApprovalSettings = @{
        approvalMode = "Serial"
        isApprovalRequired = $true
        isApprovalRequiredForExtension = $false
        isRequestorJustificationRequired = $true
        approvalStages = @(
            @{
                "@odata.type" = "#microsoft.graph.approvalStage"
                approvalStageTimeOutInDays = "14"
                isApproverJustificationRequired = $true
                isApproverAllowedToModifyRequest = $false
                isEscalationEnabled = $false
                isApproverInformationVisibleToRequestor = $false
                escalationTimeInMinutes = "0"
                primaryApprovers = @(
                    @{
                        "@odata.type" = "#microsoft.graph.groupMembers"
                        displayName = "$fallbackName"
                        objectId = "$fallbackApproverId"
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
            @{
                "@odata.type" = "#microsoft.graph.approvalStage"
                approvalStageTimeOutInDays = "14"
                isApproverJustificationRequired = $true
                isApproverAllowedToModifyRequest = $false
                isEscalationEnabled = $false
                isApproverInformationVisibleToRequestor = $false
                escalationTimeInMinutes = "0"
                primaryApprovers = @(
                    @{
                        "@odata.type" = "#microsoft.graph.groupMembers"
                        displayName = "$secondStageName"
                        objectId = "$secondStageApproverId"
                        isBackup = $false
                    }
                )
                escalationApprovers = @()
            }
        )
        skipManagerApprovalForManagerAsRequestor = $false
    }
}
New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params

```
