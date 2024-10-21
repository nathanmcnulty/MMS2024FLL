# Allow Authentication flows

Authentication flows can provide nice user experiences but are inherently risky. Microsoft recommends blocking device code flow and authentication transfer by default and only enabling users to use them when needed.

This example creates a Conditional Access policy (in report only) that enables users to request an access package that will exclude them from the policy. You can use use the following (slow) command to find out if any users are using device code flow before enabling the policy.

```powershell
Get-MgBetaAuditLogSignIn -Filter "AuthenticationProtocol eq 'deviceCode'"
```

## 1. Get or create the security group

This example uses a group to exclude users from a Conditional Access policy. To create a new group:

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$name = "em-ca-allow-authentication-flows"
$groupId = (New-MgBetaGroup -DisplayName $name -MailEnabled:$false -MailNickname $name -SecurityEnabled:$true).Id

```

To use an existing group:

```powershell
Connect-MgGraph -Scopes Group.Read.All

$name = "GroupName"
$groupId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

```

---

## 2. Create the Conditional Access policy

If you don't have a CA policy...

```powershell
Connect-MgGraph -Scopes Policy.ReadWrite.ConditionalAccess

$body = (Invoke-WebRequest -Uri "https://raw.githubusercontent.com/nathanmcnulty/MMS2024FLL/refs/heads/main/entra-entitlement-management/examples/conditional-access/authentication-flows/block-authentication-flows.json").Content

$policyId = (Invoke-MgGraphRequest -Uri "/beta/identity/conditionalAccess/policies" -Body $body -Method POST).Id

$params = @{
  "conditions" = @{
    "users" = @{
      "excludeGroups" = @( $groupId )
    }
  }
}
Update-MgBetaIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $policyId -BodyParameter $params

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

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "Allow authentication flows"
  description = "Allow use of device code flow or authentication transfer for up to 1 hour"
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

## 5. Create an access package policy

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$name = "it-developers"
$requestorsId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

$params = @{
  accessPackageId = "$packageId"
  displayName = "Members of $name"
  description = "Members of $name can request assignment"
  accessReviewSettings = $null
  expiration = @{
    type = "afterDuration"
    duration = "PT1H"
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
