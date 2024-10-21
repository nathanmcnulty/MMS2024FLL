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

> !NOTE
> Identity Governance licenses include the ability to put Entra roles in Access Packages. This builds on top of PIM and enables us to use Verified ID with Face Check, multiple approval steps, and multiple policies driven by more granular conditions.

[Grant eDiscovery admin role](admin-roles/ediscovery-admin/ediscovery-admin.md)

[Grant Exchange admin role](admin-roles/exchange-admin/exchange-admin.md)

### Entra applications

:x: [Grant access to licensed applications via app](applications/licensed-apps/licensed-apps-app.md)

:x: [Grant access to licensed applications via group](applications/licensed-apps/licensed-apps-group.md)

[SCIM provisioning](saas/scim-provisioning/scim-provisioning.md)

:x: [Secure risky apps by requiring assignment](applications/secure-apps-require-assignment/secure-apps-require-assignment.md)

[Grant permissions to SSO apps through role assignment](applications/sso-role-assignment/sso-role-assignment.md)

### Entra authentication

:x: [Limit SMS MFA use](authentication/limit-sms/limit-sms.md)

[Limit SSPR use](authentication/limit-sspr/limit-sspr.md)

:x: [Passkey rollout](authentication/passkey-rollout/passkey-rollout.md)

### Azure

[Provide access to Azure storage](azure/azure-storage/azure-storage.md)

[Provide access to Log Anaytics](azure/log-analytics/log-analytics.md)

### Conditional Access

[App protection policy exceptions](conditional-access/app-protection-policies/app-protection-policies.md)

:x: [Allow authentication flows](conditional-access/authentication-flows/authentication-flows.md)

[Allow device join/register](conditional-access/device-join-register/device-join-register.md)

[Limit guest account access](conditional-access/limit-guest-access/limit-guest-access.md)

:x: [Allow access during travel](conditional-access/travel-exclusions/travel-exclusions.md)

### Defender

[Allow Live Response](defender/live-response/live-response.md)

[Assign RBAC roles](defender/rbac-roles/rbac-roles.md)

### Device management

[Grant admin rights](device-management/admin-rights/admin-rights.md)

:x: [Change app protection policy](device-management/change-app-protection-policy/change-app-protection-policy.md)

:x: [Change compliance policy](device-management/change-compliance-policy/change-compliance-policy.md)

:x: [Change configuration profile](device-management/change-configuration-profile/change-configuration-profile.md)

[Intune device enrollment restrictions](device-management/device-restrictions/device-restrictions.md)

:x: [Send LAPS password](device-management/laps-password/laps-password.md)

[Install licensed applications](device-management/licensed-apps/licensed-apps.md)

[Pilot LOB software upgrades](device-management/lob-software-upgrades/lob-software-upgrades.md)

:x: [OS upgrade pilot](device-management/os-upgrades/os-upgrades.md)

[Disable PowerShell Constrained Language Mode](device-management/powershell-clm/powershell-clm.md)

### On-prem scenarios

> [!NOTE]  
> These require configuring group writeback through Entra Cloud Sync

[Access to file shares](on-prem/file-share-access/file-share-access.md)

[Include/Exclude from Group Policy](on-prem/group-policy/group-policy.md)

[Access IIS website](on-prem/iis-app-access/iis-app-access.md)

[Control printing](on-prem/printing-scenarios/printing-scenarios.md)

[Add to Administrators on Servers](on-prem/server-admin-group/server-admin-group.md)

### Purview

[Access MIP encryped data](purview/access-mip-encrypted-data/access-mip-encrypted-data.md)

[Control Endpoint DLP policy](purview/endpoint-dlp-policy/endpoint-dlp-policy.md)

### SharePoint

[Control guest access to sites](sharepoint/guest-access/guest-access.md)

[Access software library](sharepoint/software-library/software-library.md)
