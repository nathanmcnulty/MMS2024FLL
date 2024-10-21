# Travel exclusions

Geo-blocking policies are still surprisingly effective in many cases. For organizations that only operate in a few countries or a majority of users do not travel, geo-blocking is a great policy to have in place and alert on.

This example allows users who are travelling to blocked countries to be excluded from the policy for 7 days where the user can extend access if needed. You can customize the policy settings to fit your needs :)

## 1. Setting up the security group

This example uses a group excluded from a geo-blocking CA policy. To create a new group:

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$name = "em-ca-travel-exclusion"
$groupId = (New-MgBetaGroup -DisplayName $name -MailEnabled:$false -MailNickname $name -SecurityEnabled:$true).Id

```

To use an existing group:

```powershell
Connect-MgGraph -Scopes Group.Read.All

$name = "GroupName"
$groupId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

```

---

## 2. Create or update the geo-blocking CA policy

If you don't have a CA policy, the following code will create a policy, create a named location, and then update the policy to exclude the named location and user group.

```powershell
Connect-MgGraph -Scopes Policy.ReadWrite.ConditionalAccess

# Import policy
$body = (Invoke-WebRequest -Uri "https://raw.githubusercontent.com/nathanmcnulty/MMS2024FLL/refs/heads/main/entra-entitlement-management/examples/conditional-access/travel-exclusion/travel-exclusion.json").Content

$policyId = (Invoke-MgGraphRequest -Uri "/beta/identity/conditionalAccess/policies" -Body $body -Method POST).Id

# Create Allowed Countries Named Location
$params = @{
    displayName = "Allowed Countries"
    countryLookupMethod = "clientIpAddress"
    includeUnknownCountriesAndRegions = "false"
    countriesAndRegions = @(
        "CA","US"
    )
    "@odata.type" = "#microsoft.graph.countryNamedLocation"
}
$locationId = (New-MgBetaIdentityConditionalAccessNamedLocation -BodyParameter $params).Id

# Update policy for both location and exclusion group
$params = @{
    conditions = @{
        locations = @{
            excludeLocations = @(
                "$locationId"
            )
        }
        users = @{
            "excludeGroups" = @( 
                "$groupId"
            )
        }
    }
}
Update-MgBetaIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $policyId -BodyParameter $params

```

If you would like to use an existing CA policy, you will need to define named locations

```powershell
Connect-MgGraph -Scopes Policy.ReadWrite.ConditionalAccess

$policyId = "objectId of geo-blocking CA policy"

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
  displayName = "Travel exclusion"
  description = "Exclude from geo-blocking policies for access while traveling"
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
}
$policy = New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params

# Update expiration (not supported on beta API)

$params = @{
  id = "$packageId"
  displayName = "All Directory Members"
  description = "Any non-guest user can make a request"
  allowedTargetScope = "allDirectoryUsers"
  automaticRequestSettings = $null
  canExtend = $true
  specificAllowedTargets = @()
  expiration = @{
    type = "afterDuration"
    duration = "P7D"
  }
  requestorSettings = @{
    enableTargetsToSelfAddAccess = $true
    enableTargetsToSelfUpdateAccess = $true
    enableTargetsToSelfRemoveAccess = $true
    allowCustomAssignmentSchedule = $true
    enableOnBehalfRequestorsToAddAccess = $false
    enableOnBehalfRequestorsToUpdateAccess = $false
    enableOnBehalfRequestorsToRemoveAccess = $false
    onBehalfRequestors = @()
  }
  requestApprovalSettings = @{
    isApprovalRequiredForAdd = $true
    isApprovalRequiredForUpdate = $true
    stages = @(
      @{
        durationBeforeAutomaticDenial = "P2D"
        isApproverJustificationRequired = $false
        isEscalationEnabled = $false
        durationBeforeEscalation = "PT0S"
        primaryApprovers = @(
          @{
            "@odata.type" = "#microsoft.graph.requestorManager"
            managerLevel = 1
          }
        )
        fallbackPrimaryApprovers = @(
          @{
            "@odata.type" = "#microsoft.graph.groupMembers"
            groupId = "$approverId"
            description = "group"
          }
        )
        escalationApprovers = @()
        fallbackEscalationApprovers = @()
      }
      )
  }
  accessPackage = @{
    id = "$packageId"
  }
}

Invoke-MgGraphRequest -Uri "/v1.0/identityGovernance/entitlementManagement/assignmentPolicies/$($policy.Id)" -Method PUT -Body $params

```
