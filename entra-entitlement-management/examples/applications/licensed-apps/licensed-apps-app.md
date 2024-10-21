# Grant access to licensed applications via app

> Completed, probably need wordsmithing

This example covers scenarios where we have licensed apps in Entra that automatically provision accounts that incur billing. This can be used for non-licensed apps as well, but we might not need the same approval process as we would for licensed applications.

> !NOTE
> There are two ways to configure this scenario: 1) [Group membership assigned to the app](licensed-apps-group.md), 2) Directly assigned to the app (this example). You will need to use the group option if you want to pass the group as a claim via SSO.

## 1. Add the applicaiton to the catalog

```powershell
Connect-MgGraph -Scopes Application.Read.All,EntitlementManagement.ReadWrite.All

$name = "Docusign"
$appId = (Get-MgBetaServicePrincipal -Search "displayName:$name" -ConsistencyLevel eventual -CountVariable $count | Out-GridView -PassThru).Id

$catalogId = (Get-MgBetaEntitlementManagementAccessPackageCatalog -Filter "DisplayName eq 'General'").Id

$params = @{
	catalogId = "$catalogId"
	requestType = "AdminAdd"
	justification = ""
	accessPackageResource = @{
		resourceType = "Application"
		originId = "$appId"
		originSystem = "AadApplication"
	}
}
New-MgBetaEntitlementManagementAccessPackageResourceRequest -BodyParameter $params

$resourceId = (Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageResource -AccessPackageCatalogId $catalogId -Filter "originId eq '$appId'").Id

```

---

## 2. Create the Access Package

> !NOTE
> Remember to change the display name to match the application 

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "Docusign Access - Application"
  description = "Allows users to access Docusign"
}
$packageId = (New-MgBetaEntitlementManagementAccessPackage -BodyParameter $params).Id

```

---

## 3. Add the application to the access package

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$appRoleId = ($sp.AppRoles | Out-GridView -PassThru).Id

$role = @{
  originId = "$appRoleId"
  displayName = "DocuSign Sender"
  originSystem = "AadApplication"
  accessPackageResource = @{
    id = "$resourceId"
    resourceType = "Application"
    originId = "$appId"
    originSystem = "AadApplication"
  }
}
$scope = @{
  originId = "$appId"
  originSystem = "AadApplication"
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
