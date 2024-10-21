# Passkey rollout

This example will trigger a logic app to deliver a Temporary Access Pass with instructions to help guide the user through setting up a passkey in Microsoft Authenticator and set a Conditional Access policy that will enforce phishing resistant authentication. The logic app template currently shows how to send the TAP via Teams and via email using Azure Communication Services.

## 1. Create and update the Logic App

> [!IMPORTANT]
> Microsoft performs hardening of Logic Apps deployed through Entitlement management. It is recommended to create these from the catalog rather than an ARM template.

To do this, we go under the Catalog - Custom Extensions - Add a custom extension. During the wizard, select "Request workflow", "Launch and Continue", and then fill in the details to create the logic app and complete the wizard.

Once the Logic App has been provisioned, you can copy the JSON from the link below and paste into the Logic App code view. There will be errors due to the API connections (authentication to Teams, connection string for ACS), and you'll have to update those.

[Logic App code](passkey-rollout-la.json)

Once complete, run the following and select the custom extension you created to get the custom extension extension ID:

```powershell
$customExtensionId = (Get-MgBetaEntitlementManagementAccessPackageCatalogAccessPackageCustomWorkflowExtension -AccessPackageCatalogId $catalogId | Out-GridView -PassThru).Id

```

---

## 2. Granting the Logic App permissions

The next step is to enable a Managed Identity on the Logic App and grant it the permissions needed to read the user's devices and get the LAPS password for them.

```powershell
$MIObjectId = "c449c80d-0d9c-4e04-b340-00dcf7e2d878"
$permissions = "UserAuthenticationMethod.ReadWrite.All"

Connect-MgGraph -Scopes AppRoleAssignment.ReadWrite.All
$permissions | ForEach-Object {
	$PermissionName = $_
	$GraphSP = Get-MgServicePrincipal -Filter "startswith(DisplayName,'Microsoft Graph')" | Select-Object -first 1 #Graph App ID: 00000003-0000-0000-c000-000000000000
	$AppRole = $GraphSP.AppRoles | Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
	New-MgServicePrincipalAppRoleAssignment -AppRoleId $AppRole.Id -ServicePrincipalId $MIObjectId -ResourceId $GraphSP.Id -PrincipalId $MIObjectId
}

```

---

## 3. Get or create the security group

The Conditional Access policy used to require phishing resistant authentication uses a security group to target users. When a user requests or is assigned this Access Package, they will be placed in this group enforcing phishing resistant authentication.

To create a new group:

```powershell
Connect-MgGraph -Scopes Group.ReadWrite.All

$name = "em-passkey-rollout"
$groupId = (New-MgBetaGroup -DisplayName $name -MailEnabled:$false -MailNickname $name -SecurityEnabled:$true).Id

```

To use an existing group:

```powershell
Connect-MgGraph -Scopes Group.Read.All

$name = "GroupName"
$groupId = (Get-MgBetaGroup -Filter "DisplayName eq '$name'").Id

```

---

## 4. Create the Conditional Access policy

If you don't have a CA policy, the following script will create it for you:

```powershell
Connect-MgGraph -Scopes Policy.ReadWrite.ConditionalAccess

$body = (Invoke-WebRequest -Uri "https://raw.githubusercontent.com/nathanmcnulty/MMS2024FLL/refs/heads/main/entra-entitlement-management/examples/authentication/passkey-rollout/passkey-rollout-cap.json").Content

$policyId = (Invoke-MgGraphRequest -Uri "/beta/identity/conditionalAccess/policies" -Body $body -Method POST).Id

$params = @{
  "conditions" = @{
    "users" = @{
      "includeGroups" = @( $groupId )
    }
  }
}
Update-MgBetaIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $policyId -BodyParameter $params

```

> [!WARNING]  
> This poplicy may trap users in a login loop if they have not registered a usable MFA method. The following script checks that all members of the group have enabled MFA and will exclude them if they have not. Recommend also excluding emergency access accounts.

```powershell
Connect-MgGraph -Scopes GroupMember.Read.All,AuditLog.Read.All

[array]$excludeUsers = (Get-MgBetaGroupMember -GroupId $groupId).Id | ForEach-Object { 
    Get-MgBetaReportAuthenticationMethodUserRegistrationDetail -UserRegistrationDetailsId $_ | Where-Object { $_.IsMfaCapable -eq $false }
}

$params = @{
  "conditions" = @{
    "users" = @{
      "excludeUsers" = @( $excludeUsers.Id )
    }
  }
}
Update-MgBetaIdentityConditionalAccessPolicy -ConditionalAccessPolicyId $policyId -BodyParameter $params

```

---

## 5. Add the group to the catalog

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

## 6. Create the Access Package

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  catalogId = "$catalogId"
  displayName = "Passkey Rollout"
  description = "Requires users to register a passkey"
}
$packageId = (New-MgBetaEntitlementManagementAccessPackage -BodyParameter $params).Id

```

---

## 7. Add the group to the access package

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

## 8. Create the access package policy

```powershell
Connect-MgGraph -Scopes EntitlementManagement.ReadWrite.All

$params = @{
  accessPackageId = "$packageId"
  displayName = "All directory members"
  description = "Any non-guest user can make a request"
  accessReviewSettings = $null
  requestorSettings = @{
    scopeType = "AllExistingDirectoryMemberUsers"
    acceptRequests = $true
    allowedRequestors = @()
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

```
