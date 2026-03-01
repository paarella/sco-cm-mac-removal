# MAC Address Removal with Format — MECM Orchestrator Runbook

## Overview

This System Center Orchestrator (SCORCH) runbook automates the removal of a MAC address record from the Microsoft Endpoint Configuration Manager (MECM/SCCM) database. It is designed to be triggered from the Orchestrator Web Console, accepting one or two MAC addresses as input, and handles the full lifecycle of lookup, validation, deletion, and email notification.

## Runbook Name

**Delete Machine**

## Use Case

When a device needs to be re-imaged or redeployed via an OSD Task Sequence, a duplicate MAC address in the MECM database can block the process. This runbook allows a technician to submit the MAC address(es) through the Web Console, and the automation will:

1. Identify who initiated the request
2. Normalize the MAC address format
3. Search MECM for a matching system resource
4. Delete the matching record if found
5. Notify the requestor of the result via email

## Workflow

```
┌──────────────────────┐
│  Web Console Input   │  (MACAddress0, MACAddress1)
└─────────┬────────────┘
          │ success
          ▼
┌──────────────────────┐
│   Query Database     │  Get requesting user's SID from Orchestrator DB
└─────────┬────────────┘
          │ success
          ▼
┌──────────────────────┐
│ Get User Name from   │  Convert SID → SAM Account Name (PowerShell)
│       Query          │
└─────────┬────────────┘
          │ success
          ▼
┌──────────────────────┐
│     Get User (AD)    │  Look up user in Active Directory (email, etc.)
└─────────┬────────────┘
          │ success
          ▼
┌──────────────────────┐
│ Format MAC Addresses │  Normalize to XX:XX:XX:XX:XX:XX (PowerShell)
└─────────┬────────────┘
          │ success
          ▼
┌──────────────────────┐
│  Query ConfigMgr     │  WQL query for SMS_R_System by MACAddresses
└─────────┬────────────┘
          │
    ┌─────┴──────────┐
    │                │
    ▼ (found)        ▼ (not found / count = 0)
┌────────────┐   ┌─────────────────────┐
│ Delete MAC │   │ MAC Not Found Email  │
│  Address   │   └─────────────────────┘
└─────┬──────┘
      │ success
      ▼
┌──────────────┐
│  Send Email  │  "MAC Address Removed" notification
└──────────────┘
```

## Activities Detail

### 1. Web Console Input (Custom Start)

| Parameter    | Type   | Description                  |
|-------------|--------|------------------------------|
| MACAddress0 | String | First MAC address to remove  |
| MACAddress1 | String | Second MAC address to remove |

### 2. Query Database

Queries the Orchestrator SQL database to retrieve the `CreatedBy` SID of the user who most recently launched this runbook.

```sql
SELECT TOP 1 j.[CreatedBy]
FROM [Orchestrator].[Microsoft.SystemCenter.Orchestrator.Runtime.Internal].[Jobs] AS j
INNER JOIN [Orchestrator].[dbo].[POLICIES] AS p ON j.[RunbookId] = p.[UniqueID]
WHERE p.[Name] = '<RunbookName>'
ORDER BY j.[CreationTime] DESC
```

**Target Server:** `yourservername` — Database: `Orchestrator`

### 3. Get User Name From Query (PowerShell)

Translates the Windows SID returned from the database query into a SAM Account Name.

```powershell
$objSID = New-Object System.Security.Principal.SecurityIdentifier('<SID>')
$objUser = $objSID.Translate([System.Security.Principal.NTAccount])
$samaccountname = ($objUser.Value).split('\')[1]
```

**Published Data:** `UserName` (samaccountname)

### 4. Get User (Active Directory)

Queries Active Directory for the user matching the resolved SAM Account Name. Returns user attributes including email for notification routing.

**Configuration:** `youractivedirectoryname`  
**Filter:** `Sam Account Name` equals the resolved username

### 5. Format MAC Addresses (PowerShell)

Normalizes input MAC addresses (any format) into the `XX:XX:XX:XX:XX:XX` format expected by MECM.

```powershell
function Format-MacAddresses {
    param ([string[]]$MacAddresses)
    $formattedMacs = @()
    foreach ($mac in $MacAddresses) {
        $cleanMac = ($mac -replace '[^A-Fa-f0-9]', '')
        if ($cleanMac.Length -eq 12) {
            $formatted = ($cleanMac.ToUpper() -split '(.{2})' | Where-Object { $_ }) -join ':'
            $formattedMacs += $formatted
        } else {
            Write-Warning "Invalid MAC address: $mac"
        }
    }
    return $formattedMacs
}
```

**Published Data:** `FormattedMACAddresses`

### 6. Query Configuration Manager

Runs a WQL query against SCCM to find a system resource matching the formatted MAC address.

```sql
SELECT DISTINCT SMS_R_System.Name, SMS_R_System.ResourceId
FROM SMS_R_System
WHERE SMS_R_System.MACAddresses = "<FormattedMACAddress>"
```

**Connection:** `SiteCode` site  
**Looping:** Enabled — exits when Result Count = 0 or on failure.

### 7. Delete MAC Address (PowerShell)

Removes the matched system resource from MECM via WMI.

```powershell
$SCCMServer = "yourmcmservername"
$sitename = "yoursitecode"
$query = '<WQL from Query ConfigMgr>'

$resID = Get-WmiObject -ComputerName $SCCMServer -Query $query -Namespace "root\sms\site_$sitename"

foreach ($ResourceID in $resID) {
    $ResourceID.psbase.delete()
}
```

### 8. Send Email (Success)

Sends a notification to the requesting user upon successful deletion.

| Field   | Value |
|---------|-------|
| Subject | MAC Address Removed |
| Body    | Confirms the MAC address was removed and advises re-running the Task Sequence |
| To      | Requesting user's email (from AD lookup) |
| SMTP    | `smtp.office365.com:587` (TLS) |

### 9. MAC Not Found Email

Sends a notification when the MAC address was not found in MECM.

| Field   | Value |
|---------|-------|
| Subject | MAC Address Not Found |
| Body    | Reports the MAC was not found and includes query results for troubleshooting |
| To      | Requesting user's email (from AD lookup) |
| SMTP    | `smtp.office365.com:587` (TLS) |

## Global Variables

| Variable             | Value                    | Description                               |
|----------------------|--------------------------|-------------------------------------------|
| Primary Site Server  | `yourprimarysiteservername`   | SCCM Primary Site server             |
| Namespace SMS Root   | `root\sms\site_sitecode`      | WMI namespace for the SCCM site      |
| SCORCH Server        | `yourservername`              | Orchestrator runbook server          |
| MACAddresses         | `00:00:00:00:00:01`           | Default/test MAC address placeholder |
| ComputerName         | `windows11`                   | Default/test computer name           |

## Prerequisites

- **System Center Orchestrator** with the SCCM Integration Pack installed
- **Active Directory Integration Pack** for user lookup
- **MECM/SCCM** site (`sitecode`) accessible via WMI from the runbook server
- **SMTP relay** configured (`smtp.office365.com:587` with TLS)
- Service account (`yourserviceaccountname`) with permissions to:
  - Query and delete resources in MECM
  - Query the Orchestrator database
  - Read Active Directory user objects

## How to Use

1. Open the **Orchestrator Web Console**
2. Navigate to the **Delete_Machine_MAC_Format** folder
3. Start the **Delete Machine** runbook
4. Enter the MAC address(es) to remove (`MACAddress0` and optionally `MACAddress1`)
5. The runbook will execute and send an email notification with the result

## Notes

- MAC addresses can be entered in any common format (e.g., `AA-BB-CC-DD-EE-FF`, `AABBCCDDEEFF`, `AA:BB:CC:DD:EE:FF`) — the runbook normalizes them automatically.
- The runbook runs in **pipeline mode** with a max of 1 parallel request.
- If the MAC is not found in MECM, a "not found" email is sent instead and no deletion occurs.
