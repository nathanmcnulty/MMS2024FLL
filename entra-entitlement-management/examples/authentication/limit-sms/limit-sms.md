# Limit SMS

Purpose of this example

## 1. Get or create the security group

This example uses a group...

To create a new group populated with all users who have previously registered SMS:

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All,AuditLog.Read.All

$groupId = (New-MgBetaGroup -DisplayName 'em-authentication-allow-sms' -MailEnabled:$False  -MailNickName 'em-authentication-allow-sms' -SecurityEnabled).Id

Get-MgBetaReportAuthenticationMethodUserRegistrationDetail -Filter "methodsRegistered/any(i:i eq 'mobilePhone')" | ForEach-Object {
    New-MgBetaGroupMember -GroupId $groupId -DirectoryObjectId $_.Id
}

```

To create a new, empty group:

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$name = "em-authentication-allow-sms"
$groupId = (New-MgBetaGroup -DisplayName $name -MailEnabled:$false -MailNickname $name -SecurityEnabled:$true).Id

```

To use an existing group:

```powershell
Connect-MgGraph -Scopes Group.Read.All

$name = "GroupName"
$groupId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

```

---

## 2. Limit SMS authentication method to the group

This will change the SMS Entra authentication method from "All users" to "Select groups" and specify the group created above.

```powershell
Connect-MgGraph -Scopes Policy.ReadWrite.AuthenticationMethod

$params = @{
  "@odata.type" = "#microsoft.graph.smsAuthenticationMethodConfiguration"
  id = "Sms"
  includeTargets = @(
    @{
      "id" = "$groupId"
      "isRegistrationRequired" = $false
      "targetType" = "group"
      "isUsableForSignIn" = $false
    }
  )
  "excludeTargets" = @(
  )
  "state" = "enabled"
}
Update-MgBetaPolicyAuthenticationMethodPolicyAuthenticationMethodConfiguration -AuthenticationMethodConfigurationId Sms -BodyParameter $params

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
  displayName = "Allow SMS"
  description = "Allow use of SMS for MFA"
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

$name = "Frontline Workers"
$requestorsId = (Get-MgGroup -Filter "DisplayName eq '$name'").Id

$params = @{
  accessPackageId = "$packageId"
  displayName = "Members of $name"
  description = "Members of $name can request assignment"
  accessReviewSettings = $null
  durationInDays = 365
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
    approvalStages = @(
    )
  }
}

New-MgBetaEntitlementManagementAccessPackageAssignmentPolicy -BodyParameter $params
```
