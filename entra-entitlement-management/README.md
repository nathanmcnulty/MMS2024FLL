# Entitlement management

This is a collection of examples, scripts, and tips for Entra Entitlement management. There are currently about 35 examples I am working on publishing, so keep an eye on this repository as these will be completed in the coming weeks :)

## Resources

### Microsoft Docs

[Common scenarios](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-scenarios)  

[Creating an access package](https://learn.microsoft.com/en-us/entra/id-governance/entitlement-management-access-package-create)  

[Security best practices](https://learn.microsoft.com/en-us/entra/id-governance/best-practices-secure-id-governance)  

[Securing Custom Extensions(Logic Apps)](https://learn.microsoft.com/en-us/entra/id-governance/custom-extension-security)  

[Graph API Docs](https://learn.microsoft.com/en-us/graph/api/resources/entitlementmanagement-overview?view=graph-rest-1.0)  

[Graph PowerShell Tutorial](https://learn.microsoft.com/en-us/powershell/microsoftgraph/tutorial-entitlement-management?view=graph-powershell-1.0)  


### Blogs

[Pim Jacobs - Series](https://identity-man.eu/2023/02/15/using-the-hidden-gems-in-azure-ad-access-packages-all-you-need-to-know-part-1/)  


### Tips / Tricks

Groups, apps, and roles are referenced in commands by their object ID which is very easy to get. 


SharePoint sites are actually the web URL, but discovery can be lame due to Graph API only supporting LIST for Application permissions... Instead, we have to search which only works for some site types.

```powershell
# Get IDs for groups
Get-MgBetaGroup -All
(Get-MgBetaGroup -Filter "DisplayName eq 'Group'").Id

# Get IDs for apps
Get-MgBetaApplication -All
(Get-MgBetaApplication -Filter "DisplayName eq 'App'").AppId

# Get IDs for SP sites
Get-MgBetaSite -Search 'site'
(Get-MgBetaSite -Search 'Site').WebUrl

# Get IDs for Entra roles
Get-MgBetaRoleManagementDirectoryRoleDefinition
(Get-MgBetaRoleManagementDirectoryRoleDefinition -Filter "DisplayName eq 'Role'").Id

```

For custom extensions (Logic Apps), we usually need to grant Graph API permissions for them to perform functions. Simply set $MIObjectId to the Managede Identity object ID and change $permissions to match the set of permissions you need.

```powershell
# Modified from @AlexFilipin: https://gist.github.com/AlexFilipin/daace2f2d7989545e8ab0b969de2aaed
$MIObjectId = "c449c80d-0d9c-4e04-b340-00dcf7e2d878"
$permissions = "User.Read.All","Group.Read.All"

Connect-MgGraph -Scopes AppRoleAssignment.ReadWrite.All
$permissions | ForEach-Object {
	$PermissionName = $_
	$GraphSP = Get-MgServicePrincipal -Filter "startswith(DisplayName,'Microsoft Graph')" | Select-Object -first 1 #Graph App ID: 00000003-0000-0000-c000-000000000000
	$AppRole = $GraphSP.AppRoles | Where-Object {$_.Value -eq $PermissionName -and $_.AllowedMemberTypes -contains "Application"}
	New-MgServicePrincipalAppRoleAssignment -AppRoleId $AppRole.Id -ServicePrincipalId $MIObjectId -ResourceId $GraphSP.Id -PrincipalId $MIObjectId
}

```

More details on Managed Identities with Graph API and Logic Apps:  

[Jeff Brown](https://jeffbrown.tech/graph-api-managed-identity/)  

[Toon Vanhoutte](https://yourazurecoach.com/2023/05/08/authenticate-logic-apps-against-microsoft-graph-using-managed-identity/)  

[Alex Jaya](https://medium.com/@alex.jaya/id-governance-workflow-with-custom-extensions-logic-apps-using-logic-apps-custom-api-connector-c3e7e5d04281)  

## Examples

### Admin roles

> [!NOTE]
> Identity Governance licenses include the ability to put Entra roles in Access Packages. This builds on top of PIM and enables us to use Verified ID with Face Check, multiple approval steps, and multiple policies driven by more granular conditions.

:construction: [Grant eDiscovery admin role](./examples/admin-roles/ediscovery-admin/ediscovery-admin.md)

:construction: [Grant Exchange admin role](./examples/admin-roles/exchange-admin/exchange-admin.md)

---

### Entra applications

:white_check_mark: [Grant access to licensed applications via app](./examples/applications/licensed-apps/licensed-apps-app.md)

:white_check_mark: [Grant access to licensed applications via group](./examples/applications/licensed-apps/licensed-apps-group.md)

:construction: [SCIM provisioning](./examples/saas/scim-provisioning/scim-provisioning.md)

:white_check_mark: [Secure risky apps by requiring assignment](./examples/applications/secure-apps-require-assignment/secure-apps-require-assignment.md)

:construction: [Grant permissions to SSO apps through role assignment](./examples/applications/sso-role-assignment/sso-role-assignment.md)

---

### Entra authentication

:white_check_mark: [Limit SMS MFA use](./examples/authentication/limit-sms/limit-sms.md)

:construction: [Limit SSPR use](./examples/authentication/limit-sspr/limit-sspr.md)

:white_check_mark: [Passkey rollout](./examples/authentication/passkey-rollout/passkey-rollout.md)

---

### Azure

:construction: [Provide access to Azure storage](./examples/azure/azure-storage/azure-storage.md)

:construction: [Provide access to Log Anaytics](./examples/azure/log-analytics/log-analytics.md)

---

### Conditional Access

:construction: [App protection policy exceptions](./examples/conditional-access/app-protection-policies/app-protection-policies.md)

:white_check_mark: [Allow authentication flows](./examples/conditional-access/authentication-flows/authentication-flows.md)

:construction: [Allow device join/register](./examples/conditional-access/device-join-register/device-join-register.md)

:construction: [Limit guest account access](./examples/conditional-access/limit-guest-access/limit-guest-access.md)

:white_check_mark: [Allow access during travel](./examples/conditional-access/travel-exclusions/travel-exclusions.md)

---

### Defender

:construction: [Allow Live Response](./examples/defender/live-response/live-response.md)

:construction: [Assign RBAC roles](./examples/defender/rbac-roles/rbac-roles.md)

---

### Device management

:construction: [Grant admin rights](./examples/device-management/admin-rights/admin-rights.md)

:white_check_mark: [Change app protection policy](./examples/device-management/change-app-protection-policy/change-app-protection-policy.md)

:white_check_mark: [Change compliance policy](./examples/device-management/change-compliance-policy/change-compliance-policy.md)

:white_check_mark: [Change configuration profile](./examples/device-management/change-configuration-profile/change-configuration-profile.md)

:construction: [Intune device enrollment restrictions](./examples/device-management/device-restrictions/device-restrictions.md)

:white_check_mark: [Send LAPS password](./examples/device-management/laps-password/laps-password.md)

:construction: [Install licensed applications](./examples/device-management/licensed-apps/licensed-apps.md)

:construction: [Pilot LOB software upgrades](./examples/device-management/lob-software-upgrades/lob-software-upgrades.md)

:white_check_mark: [OS upgrade pilot](./examples/device-management/os-upgrades/os-upgrades.md)

:construction: [Disable PowerShell Constrained Language Mode](./examples/device-management/powershell-clm/powershell-clm.md)

---

### On-prem scenarios

> [!NOTE]  
> These require configuring group writeback through Entra Cloud Sync

:construction: [Access to file shares](./examples/on-prem/file-share-access/file-share-access.md)

:construction: [Include/Exclude from Group Policy](./examples/on-prem/group-policy/group-policy.md)

:construction: [Access IIS website](./examples/on-prem/iis-app-access/iis-app-access.md)

:construction: [Control printing](./examples/on-prem/printing-scenarios/printing-scenarios.md)

:construction: [Add to Administrators on Servers](./examples/on-prem/server-admin-group/server-admin-group.md)

---

### Purview

:construction: [Access MIP encryped data](./examples/purview/access-mip-encrypted-data/access-mip-encrypted-data.md)

:construction: [Control Endpoint DLP policy](./examples/purview/endpoint-dlp-policy/endpoint-dlp-policy.md)

---

### SharePoint

:construction: [Control guest access to sites](./examples/sharepoint/guest-access/guest-access.md)

:construction: [Access software library](./examples/sharepoint/software-library/software-library.md)
