---
layout: default
title:  "Automating SharePoint Deployments with PowerShell"
date:   2025-11-14 12:00:00
categories: SharePoint PowerShell Automation
---

I used to manually create SharePoint sites. Click through the UI, configure settings, create lists, set permissions, upload templates. It took hours and was error-prone.

Then I discovered PowerShell automation. Now I can provision an entire site structure in minutes, consistently, repeatedly, and without forgetting that one setting that always breaks things.

Here's what I've learned about automating SharePoint with PowerShell.

## Why Automate SharePoint Deployments?

Before we dive into code, let's talk about why this matters.

### Consistency

Manual deployments lead to drift. Site A has one configuration, Site B has a slightly different one because you forgot a step. With PowerShell scripts, every deployment is identical.

### Speed

What takes 30 minutes manually takes 2 minutes with automation. For bulk operations (creating 50 project sites), automation isn't optional—it's the only practical approach.

### Documentation

Your PowerShell script *is* your documentation. Want to know how production sites are configured? Read the deployment script.

### Repeatability

Environment refreshes, disaster recovery, deploying to new tenants—automation makes these routine instead of panic-inducing events.

## PnP PowerShell: Your Best Friend

The **PnP PowerShell** module (Patterns and Practices) is the de facto standard for SharePoint automation. It works with both SharePoint Online and SharePoint Server (2013, 2016, 2019).

### Installing PnP PowerShell

```powershell
# Install PnP PowerShell (SharePoint Online)
Install-Module -Name PnP.PowerShell -Scope CurrentUser

# Verify installation
Get-Module -Name PnP.PowerShell -ListAvailable
```

### Connecting to SharePoint

```powershell
# Interactive login (prompts for credentials)
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/yoursite" -Interactive

# Using credentials
$cred = Get-Credential
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/yoursite" -Credentials $cred

# Using app registration (for automation)
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/yoursite" `
    -ClientId "your-client-id" `
    -Tenant "yourtenant.onmicrosoft.com" `
    -CertificatePath "C:\cert.pfx" `
    -CertificatePassword (ConvertTo-SecureString -String "password" -AsPlainText -Force)
```

For automated scripts (scheduled tasks, CI/CD), use app registration with certificates. For interactive work, `-Interactive` is fine.

## Real-World Automation Scripts

Let me show you the scripts I actually use. These aren't theoretical examples—these are copy-paste-and-modify scripts that solve real problems.

### Script 1: Provision a New Project Site

This script creates a consistent project site structure with predefined lists, libraries, and permissions.

```powershell
<#
.SYNOPSIS
    Creates a new SharePoint project site with standard structure
.PARAMETER ProjectName
    Name of the project (used for site title and URL)
.PARAMETER ProjectOwner
    Email address of the project owner
.EXAMPLE
    .\New-ProjectSite.ps1 -ProjectName "Website Redesign" -ProjectOwner "john@company.com"
#>

param(
    [Parameter(Mandatory=$true)]
    [string]$ProjectName,

    [Parameter(Mandatory=$true)]
    [string]$ProjectOwner
)

# Configuration
$TenantUrl = "https://yourtenant.sharepoint.com"
$SiteCollectionUrl = "$TenantUrl/sites/projects"

# Generate URL-friendly alias
$Alias = $ProjectName -replace '[^a-zA-Z0-9]', '-' -replace '-+', '-'
$SiteUrl = "$SiteCollectionUrl/$Alias"

Write-Host "Creating project site: $ProjectName" -ForegroundColor Green

# Connect to SharePoint
Connect-PnPOnline -Url $SiteCollectionUrl -Interactive

# Create the subsite
$site = New-PnPWeb -Title $ProjectName `
    -Url $Alias `
    -Template "STS#3" `
    -BreakInheritance

Connect-PnPOnline -Url $SiteUrl -Interactive

# Create document libraries
Write-Host "Creating document libraries..."
New-PnPList -Title "Project Documents" -Template DocumentLibrary
New-PnPList -Title "Deliverables" -Template DocumentLibrary
New-PnPList -Title "Meeting Notes" -Template DocumentLibrary

# Create lists
Write-Host "Creating lists..."
New-PnPList -Title "Tasks" -Template Tasks
New-PnPList -Title "Issues" -Template IssueTracking
New-PnPList -Title "Project Milestones" -Template GenericList

# Add columns to Milestones list
Add-PnPField -List "Project Milestones" -DisplayName "Due Date" -InternalName "DueDate" -Type DateTime -AddToDefaultView
Add-PnPField -List "Project Milestones" -DisplayName "Status" -InternalName "Status" -Type Choice -Choices "Not Started","In Progress","Completed","Delayed" -AddToDefaultView

# Create SharePoint groups
Write-Host "Creating SharePoint groups..."
$ownersGroup = New-PnPGroup -Title "$ProjectName Owners" -Owner $ProjectOwner
$membersGroup = New-PnPGroup -Title "$ProjectName Members"
$visitorsGroup = New-PnPGroup -Title "$ProjectName Visitors"

# Add owner to owners group
Add-PnPGroupMember -Group "$ProjectName Owners" -EmailAddress $ProjectOwner

# Set permissions
Write-Host "Configuring permissions..."
Set-PnPGroupPermissions -Identity "$ProjectName Owners" -AddRole "Full Control"
Set-PnPGroupPermissions -Identity "$ProjectName Members" -AddRole "Edit"
Set-PnPGroupPermissions -Identity "$ProjectName Visitors" -AddRole "Read"

# Create default folders
Write-Host "Creating folder structure..."
Add-PnPFolder -Name "Templates" -Folder "Project Documents"
Add-PnPFolder -Name "Draft" -Folder "Deliverables"
Add-PnPFolder -Name "Final" -Folder "Deliverables"

# Set site logo (if you have a standard project logo)
# Set-PnPWebTheme -Theme "Company Theme"

Write-Host "Project site created successfully!" -ForegroundColor Green
Write-Host "URL: $SiteUrl"
Write-Host "Owner: $ProjectOwner"
```

**Usage:**
```powershell
.\New-ProjectSite.ps1 -ProjectName "Website Redesign" -ProjectOwner "john@company.com"
```

This creates a complete project site in under 2 minutes, consistently, every time.

### Script 2: Bulk Upload Documents

You need to upload hundreds of files from a file share to SharePoint. Doing this manually is madness.

```powershell
<#
.SYNOPSIS
    Bulk upload files from local folder to SharePoint library
.PARAMETER SourcePath
    Local folder containing files to upload
.PARAMETER SiteUrl
    SharePoint site URL
.PARAMETER LibraryName
    Document library name
.EXAMPLE
    .\Upload-BulkFiles.ps1 -SourcePath "C:\Files" -SiteUrl "https://tenant.sharepoint.com/sites/site" -LibraryName "Documents"
#>

param(
    [Parameter(Mandatory=$true)]
    [string]$SourcePath,

    [Parameter(Mandatory=$true)]
    [string]$SiteUrl,

    [Parameter(Mandatory=$true)]
    [string]$LibraryName
)

# Validate source path
if (-not (Test-Path $SourcePath)) {
    Write-Error "Source path does not exist: $SourcePath"
    exit 1
}

Write-Host "Starting bulk upload..." -ForegroundColor Green
Write-Host "Source: $SourcePath"
Write-Host "Destination: $SiteUrl/$LibraryName"

# Connect to SharePoint
Connect-PnPOnline -Url $SiteUrl -Interactive

# Get all files recursively
$files = Get-ChildItem -Path $SourcePath -File -Recurse

$totalFiles = $files.Count
$currentFile = 0
$successCount = 0
$failCount = 0

foreach ($file in $files) {
    $currentFile++
    $percentComplete = [math]::Round(($currentFile / $totalFiles) * 100)

    # Calculate relative path for folder structure
    $relativePath = $file.DirectoryName.Replace($SourcePath, "").TrimStart("\")
    $targetFolder = if ($relativePath) { "$LibraryName/$($relativePath.Replace('\', '/'))" } else { $LibraryName }

    Write-Progress -Activity "Uploading files" `
        -Status "Processing $($file.Name) ($currentFile of $totalFiles)" `
        -PercentComplete $percentComplete

    try {
        # Ensure folder exists
        if ($relativePath) {
            $folderPath = $relativePath.Replace('\', '/')
            Resolve-PnPFolder -SiteRelativePath "$LibraryName/$folderPath" | Out-Null
        }

        # Upload file
        Add-PnPFile -Path $file.FullName -Folder $targetFolder -ErrorAction Stop | Out-Null
        $successCount++
        Write-Host "✓ Uploaded: $($file.FullName)" -ForegroundColor Green
    }
    catch {
        $failCount++
        Write-Host "✗ Failed: $($file.FullName) - $($_.Exception.Message)" -ForegroundColor Red
    }
}

Write-Progress -Activity "Uploading files" -Completed

Write-Host "`nUpload complete!" -ForegroundColor Green
Write-Host "Total files: $totalFiles"
Write-Host "Successful: $successCount" -ForegroundColor Green
Write-Host "Failed: $failCount" -ForegroundColor Red
```

This script preserves folder structure and provides progress feedback. I've used this to upload 1,000+ files at once.

### Script 3: Export and Apply Site Template

You've configured a site perfectly and want to replicate that configuration elsewhere.

```powershell
# Export site as template
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/template-site" -Interactive

Get-PnPSiteTemplate -Out "C:\Templates\ProjectSiteTemplate.xml" `
    -Handlers Lists, Fields, ContentTypes, Pages, Files

Write-Host "Template exported successfully!"

# Apply template to new site
Connect-PnPOnline -Url "https://yourtenant.sharepoint.com/sites/new-site" -Interactive

Invoke-PnPSiteTemplate -Path "C:\Templates\ProjectSiteTemplate.xml"

Write-Host "Template applied successfully!"
```

**Handlers you can include:**
- `Lists` - List and library definitions
- `Fields` - Site columns
- `ContentTypes` - Content types
- `Pages` - Pages and web parts
- `Files` - Documents and files
- `Navigation` - Site navigation
- `WebSettings` - Site settings
- `Features` - Activated features

### Script 4: Manage Permissions at Scale

You need to add a group to 50 document libraries across multiple sites.

```powershell
<#
.SYNOPSIS
    Grant permissions to a group across multiple sites
.PARAMETER SiteUrls
    Array of site URLs
.PARAMETER GroupName
    SharePoint group name or AD group
.PARAMETER PermissionLevel
    Permission level to grant (Read, Edit, Contribute, etc.)
#>

param(
    [Parameter(Mandatory=$true)]
    [string[]]$SiteUrls,

    [Parameter(Mandatory=$true)]
    [string]$GroupName,

    [Parameter(Mandatory=$true)]
    [ValidateSet("Read","Edit","Contribute","Full Control")]
    [string]$PermissionLevel
)

foreach ($siteUrl in $SiteUrls) {
    Write-Host "`nProcessing: $siteUrl" -ForegroundColor Cyan

    try {
        Connect-PnPOnline -Url $siteUrl -Interactive

        # Grant permissions
        Grant-PnPSiteDesignRights -Identity $GroupName -Rights $PermissionLevel

        Write-Host "✓ Permissions granted successfully" -ForegroundColor Green
    }
    catch {
        Write-Host "✗ Failed: $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

### Script 5: Audit Site Collection Administrators

You need to know who has admin rights across all site collections.

```powershell
<#
.SYNOPSIS
    Export all site collection administrators across tenant
#>

Connect-PnPOnline -Url "https://yourtenant-admin.sharepoint.com" -Interactive

$sites = Get-PnPTenantSite

$adminReport = @()

foreach ($site in $sites) {
    Write-Host "Checking: $($site.Url)"

    Connect-PnPOnline -Url $site.Url -Interactive

    $admins = Get-PnPSiteCollectionAdmin

    foreach ($admin in $admins) {
        $adminReport += [PSCustomObject]@{
            SiteUrl = $site.Url
            SiteTitle = $site.Title
            AdminName = $admin.Title
            AdminEmail = $admin.Email
            IsSiteOwner = $admin.IsSiteAdmin
        }
    }
}

# Export to CSV
$reportPath = "C:\Reports\SiteCollectionAdmins_$(Get-Date -Format 'yyyyMMdd').csv"
$adminReport | Export-Csv -Path $reportPath -NoTypeInformation

Write-Host "`nReport generated: $reportPath" -ForegroundColor Green
Write-Host "Total sites checked: $($sites.Count)"
Write-Host "Total administrators found: $($adminReport.Count)"
```

Run this quarterly to audit admin access.

## Best Practices for SharePoint Automation

### 1. Use Parameters, Not Hardcoded Values

**Bad:**
```powershell
Connect-PnPOnline -Url "https://contoso.sharepoint.com/sites/hr" -Interactive
New-PnPList -Title "Employee List" -Template GenericList
```

**Good:**
```powershell
param(
    [string]$SiteUrl,
    [string]$ListTitle
)
Connect-PnPOnline -Url $SiteUrl -Interactive
New-PnPList -Title $ListTitle -Template GenericList
```

Hardcoded values make scripts single-use. Parameters make them reusable.

### 2. Include Error Handling

**Bad:**
```powershell
New-PnPList -Title "Tasks" -Template Tasks
Add-PnPField -List "Tasks" -DisplayName "Priority" -Type Choice
```

**Good:**
```powershell
try {
    $list = New-PnPList -Title "Tasks" -Template Tasks -ErrorAction Stop
    Add-PnPField -List "Tasks" -DisplayName "Priority" -Type Choice -ErrorAction Stop
    Write-Host "✓ List created successfully" -ForegroundColor Green
}
catch {
    Write-Host "✗ Error: $($_.Exception.Message)" -ForegroundColor Red
    # Optionally: rollback, log, alert
}
```

Production scripts need error handling. Always.

### 3. Log Everything

```powershell
# Set up logging
$logPath = "C:\Logs\SharePoint\Deployment_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"

function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "$timestamp [$Level] $Message"
    Add-Content -Path $logPath -Value $logEntry
    Write-Host $logEntry
}

Write-Log "Starting deployment..."
Write-Log "Site URL: $siteUrl"

try {
    # Your code here
    Write-Log "Operation completed successfully" -Level "SUCCESS"
}
catch {
    Write-Log "Error: $($_.Exception.Message)" -Level "ERROR"
}
```

Future you will thank past you for good logs.

### 4. Test Before Production

Always test scripts in a dev/test environment first. Use whatif parameters when available:

```powershell
# This shows what would happen without actually doing it
Remove-PnPList -Identity "OldList" -WhatIf
```

### 5. Use Source Control

Your PowerShell scripts should be in Git. Version control for infrastructure as code is non-negotiable.

```
repo/
├── SharePoint/
│   ├── Provisioning/
│   │   ├── New-ProjectSite.ps1
│   │   └── New-DepartmentSite.ps1
│   ├── Maintenance/
│   │   ├── Audit-Permissions.ps1
│   │   └── Cleanup-OldFiles.ps1
│   └── Templates/
│       └── ProjectSiteTemplate.xml
```

## Common Pitfalls

### Pitfall 1: Not Disconnecting

```powershell
# Multiple connections can cause issues
Disconnect-PnPOnline  # Always disconnect when done
```

### Pitfall 2: Assuming Success

Always check return values:

```powershell
$list = Get-PnPList -Identity "Tasks"
if ($null -eq $list) {
    Write-Error "List not found!"
    exit 1
}
```

### Pitfall 3: Ignoring Rate Limiting

For bulk operations, SharePoint will throttle you. Add delays:

```powershell
foreach ($item in $largeCollection) {
    # Process item
    Start-Sleep -Milliseconds 500  # Avoid throttling
}
```

### Pitfall 4: Hardcoded Credentials

Never hardcode credentials in scripts:

```powershell
# BAD - credentials in script
$password = ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force

# GOOD - use certificate-based auth or credential manager
$cred = Get-Credential  # For interactive
# Or use certificate for automated
```

## My Automation Toolkit

Here's my standard setup for SharePoint automation:

```powershell
# Profile.ps1 - Load this in every PowerShell session
# Standard modules
Import-Module PnP.PowerShell

# Common variables
$Global:TenantUrl = "https://yourtenant.sharepoint.com"
$Global:TenantAdminUrl = "https://yourtenant-admin.sharepoint.com"

# Helper functions
function Connect-SPOSite {
    param([string]$Url)
    Connect-PnPOnline -Url $Url -Interactive
}

function Get-SPOLists {
    Get-PnPList | Select-Object Title, ItemCount, LastItemModifiedDate |
        Sort-Object Title
}

# Set up logging path
$Global:LogPath = "C:\Logs\SharePoint"
if (-not (Test-Path $LogPath)) {
    New-Item -Path $LogPath -ItemType Directory | Out-Null
}
```

Save this as your PowerShell profile and you'll have these functions available in every session.

## Wrapping Up

SharePoint automation with PowerShell transforms how you manage SharePoint. What takes hours manually takes minutes with automation, and you gain consistency and repeatability.

Start small:
1. Automate one repetitive task
2. Build a library of reusable scripts
3. Version control everything
4. Share with your team

The time investment pays off immediately.

## Further Reading

- [PnP PowerShell Documentation](https://pnp.github.io/powershell/)
- [Microsoft SharePoint Online Management Shell](https://learn.microsoft.com/en-us/powershell/sharepoint/sharepoint-online/introduction-sharepoint-online-management-shell)
- [PnP PowerShell GitHub Repository](https://github.com/pnp/powershell)
- [SharePoint Patterns and Practices](https://pnp.github.io/)

---

*Questions about SharePoint PowerShell automation? [Reach out](mailto:jordan@jordananderson.us).*
