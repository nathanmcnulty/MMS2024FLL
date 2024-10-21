# Change app protection policy

Sometimes app protection policies may cause problems with certain user workflows, and we might want to temporarily allow a subset of users to have a less restrictive app protection policy. To facilitate this, we use a group that will be excluded from the primary app protection policy and included in a new, less restrictive policy.

## 1. Get or create the security group

This example uses a group to control which app protection policy is applied to users.

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$groupId = (New-MgBetaGroup -DisplayName 'em-change-app-protection-policy' -MailEnabled:$False  -MailNickName 'em-change-app-protection-policy' -SecurityEnabled).Id

```

---

## 2. Add the group to the catalog

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

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "Adjust app protection policy"
  description = "Allows users to request a less restrictive app protection policy"
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

> [!NOTE]
> The groups in the code below must already exist. If they do not, you will need to create them.

Create a policy that always provides access:
```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$name = "it-developers"
$requestorsId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

$name = "it-admins"
$approverId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

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
    approvalMode = "SingleStage"
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
            displayName = "$name"
            objectId = "$approverId"
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

---

## 6. Update App protection policies

At this point, we should have a group in an access package available to select users. The final step is to exclude this new group from the primary app protection policy in Intune and include it on a less strict app protection policy.
