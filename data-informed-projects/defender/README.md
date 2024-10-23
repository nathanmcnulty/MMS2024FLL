# Defender XDR

## MDE hack!

This is a cool little trick to get extra data in to MDE. Create a Scheduled Task triggered by whatever you want and have it launch PowerShell to capture whatever you need and echo it back out to the console. MDE PowerShell block logging will pick up the results for you :)

![img](./img/powershell-hack.jpg)

## Advanced Hunting

```kql
DeviceNetworkEvents
| where ActionType == 'ConnectionSuccess'
| where RemoteIPType == 'Public'
| where InitiatingProcessVersionInfoOriginalFileName in~ ('wscript.exe','mshta.exe','cscript.exe','conhost.exe','runScriptHelper.exe')
```
```kql
DeviceNetworkEvents
| where RemoteIPType == "Public"
| where InitiatingProcessVersionInfoOriginalFileName in (( externaldata ( Name:string ) [ "https://lolbas-project.github.io/api/lolbas.csv" ] with (format=csv, ignoreFirstRecord=true)| distinct Name ))
```

```kql
DeviceFileEvents
| where FileName has_any ("credential","password") or FolderPath has_any ("credential","password")
```

https://kqlsearch.com

https://firewalliplists.gypthecat.com/kusto-tables/

## Defender APIs

Threat Hunting API is part of Graph, the future of hunting our data via API

[Defender for Endpoint API](https://learn.microsoft.com/en-us/defender-endpoint/api/apis-intro)  
[Defender for Cloud Apps API](https://learn.microsoft.com/en-us/defender-cloud-apps/api-introduction)  
[Defender for Office 365 API](https://learn.microsoft.com/en-us/graph/api/security-collaborationroot-list-analyzedemails?view=graph-rest-beta)  
[Defender for Identity API - Health Issues](https://learn.microsoft.com/en-us/graph/api/resources/security-healthissue?view=graph-rest-beta)  
[Defender for Identity API - Security sensor](https://learn.microsoft.com/en-us/graph/api/resources/security-sensor?view=graph-rest-beta)

### MDE API examples

[MDE API Docs](https://learn.microsoft.com/en-us/defender-endpoint/api/exposed-apis-list)

Get token and set headers

```powershell
## Service Principal Info
$tenantId = '847b3907-ca15-b0f4-b171-eb18319dbfab'
$appId = '6b3b624f-99bb-4ad7-bd38-16a6b3a476a2'
$appSecret = 'wjA8Q~vTRm3XChp_IzD2odWYKWZcaoA9a4Ozldrl'

## Get Token
$resourceAppIdUri = 'https://api.securitycenter.microsoft.com'
$oAuthUri = "https://login.microsoftonline.com/$TenantId/oauth2/token"
$authBody = [Ordered] @{
    resource = "$resourceAppIdUri"
    client_id = "$appId"
    client_secret = "$appSecret"
    grant_type = 'client_credentials'
}
$token = (Invoke-RestMethod -Method Post -Uri $oAuthUri -Body $authBody -ErrorAction Stop).access_token

## Set the WebRequest headers
$headers = @{
    'Content-Type' = 'application/json'
    Accept = 'application/json'
    Authorization = "Bearer $token"
}
```

Get list of devices

```powershell
# List all devices
(Invoke-RestMethod -Method GET -Uri "https://api.securitycenter.microsoft.com/api/machines" -Headers $headers).value

# Get all software
(Invoke-RestMethod -Method GET -Uri "https://api.securitycenter.microsoft.com//api/Software" -Headers $headers).value

# Get machines a user logged into
(Invoke-RestMethod -Method GET -Uri "https://api.securitycenter.microsoft.com/api/users/nathanmcnulty/machines" -Headers $headers).value

# AV health report
(Invoke-RestMethod -Method GET -Uri "https://api.securitycenter.microsoft.com/api/deviceavinfo" -Headers $headers).value

# Isolate a machine
(Invoke-RestMethod -Method POST -Uri "https://api.securitycenter.microsoft.com/api/machines/63be9aba8978cfdec1f33517c7f54f2a71778aba/isolate" -Headers $headers).value
```

Live response!
```powershell
$body = @{
   "Commands":[
      {
         "type":"RunScript",
         "params":[
            {
               "key":"ScriptName",
               "value":"minidump.ps1"
            },
            {
               "key":"Args",
               "value":"OfficeClickToRun"
            }

         ]
      },
      {
         "type":"GetFile",
         "params":[
            {
               "key":"Path",
               "value":"C:\\windows\\TEMP\\OfficeClickToRun.dmp.zip"
            }
         ]
      }
   ],
   "Comment":"Testing Live Response API"
}
(Invoke-RestMethod -Method POST -Uri "https://api.securitycenter.microsoft.com/API/machines/63be9aba8978cfdec1f33517c7f54f2a71778aba/runliveresponse" -Headers $headers).value
```

Tags
```powershell
# Get devices by tag
[array]$VDI = (Invoke-RestMethod -Method GET -Uri "https://api.securitycenter.microsoft.com/api/machines/findbytag?tag=VDI" -Headers $headers).value

# Add MDE-Management tag
$body = @{
    "Value" = "MDE-Management"
    "Action" = "Add"
    "MachineIds" = @($serverIDs)
} | ConvertTo-Json
Invoke-RestMethod -Method POST -Uri "https://api.securitycenter.microsoft.com/api/machines/AddOrRemoveTagForMultipleMachines" -Headers $headers -Body $body

# Remove MDE-Management tag
$body = @{
    "Value" = "MDE-Management"
    "Action" = "Remove"
    "MachineIds" = @($VDI)
} | ConvertTo-Json
Invoke-RestMethod -Method POST -Uri "https://api.securitycenter.microsoft.com/api/machines/AddOrRemoveTagForMultipleMachines" -Headers $headers -Body $body
```

Advanced Hunting (Graph API)
```powershell
$params = @{
	Query = 'DeviceProcessEvents | where InitiatingProcessFileName =~ "powershell.exe" | project Timestamp, FileName, InitiatingProcessFileName | order by Timestamp desc | limit 5'
}

(Start-MgSecurityHuntingQuery -BodyParameter $params).ToJsonString()

(Invoke-MgGraphRequest -Method POST -Uri "https://graph.microsoft.com/v1.0/security/runHuntingQuery" -Body $params -OutputType PSObject).results | fl *

```

## Sentinel integration

[Free data sources](https://learn.microsoft.com/en-us/azure/sentinel/billing?tabs=simplified%2Ccommitment-tiers#free-data-sources)  

[Onboarding Sentinel to Defender](https://learn.microsoft.com/en-us/defender-xdr/microsoft-sentinel-onboard)  
