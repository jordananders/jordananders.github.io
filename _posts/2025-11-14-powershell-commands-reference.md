---
layout: default
title:  "PowerShell Commands I Always Forget"
date:   2025-11-14 18:00:00
categories: PowerShell Windows DevOps
---

I use PowerShell daily, but there are commands I have to look up every single time. This is my personal reference—the commands I actually use, with examples that work.

## File and Directory Operations

### Navigate and List

```powershell
# Current directory
Get-Location
# Or shorter
pwd

# Change directory
Set-Location C:\Users
# Or shorter
cd C:\Users

# List files and folders
Get-ChildItem
# Or shorter
ls
dir

# List with details
Get-ChildItem | Format-List

# List recursively
Get-ChildItem -Recurse

# List hidden files
Get-ChildItem -Force

# List only files
Get-ChildItem -File

# List only directories
Get-ChildItem -Directory

# Filter by extension
Get-ChildItem -Filter *.txt

# Search recursively for filename
Get-ChildItem -Recurse -Filter "web.config"
```

### Create, Copy, Move, Delete

```powershell
# Create directory
New-Item -Path "C:\Temp\NewFolder" -ItemType Directory
# Or shorter
mkdir C:\Temp\NewFolder

# Create file
New-Item -Path "C:\Temp\file.txt" -ItemType File

# Copy file
Copy-Item -Path "source.txt" -Destination "dest.txt"

# Copy directory recursively
Copy-Item -Path "C:\Source" -Destination "C:\Dest" -Recurse

# Move file
Move-Item -Path "old.txt" -Destination "new.txt"

# Rename file
Rename-Item -Path "old.txt" -NewName "new.txt"

# Delete file
Remove-Item -Path "file.txt"

# Delete directory recursively
Remove-Item -Path "C:\Folder" -Recurse -Force

# Delete all files matching pattern
Remove-Item -Path "C:\Temp\*.log"
```

### Read and Write Files

```powershell
# Read file content
Get-Content -Path "file.txt"
# Or shorter
cat file.txt

# Read first 10 lines
Get-Content -Path "file.txt" -TotalCount 10

# Read last 10 lines
Get-Content -Path "file.txt" -Tail 10

# Follow file (like tail -f)
Get-Content -Path "log.txt" -Wait

# Write to file (overwrite)
"Hello World" | Out-File -FilePath "file.txt"

# Append to file
"New line" | Out-File -FilePath "file.txt" -Append

# Write multiple lines
@"
Line 1
Line 2
Line 3
"@ | Out-File -FilePath "file.txt"
```

## String and Text Processing

### Search in Files (like grep)

```powershell
# Search for text in file
Select-String -Path "file.txt" -Pattern "error"

# Search recursively
Get-ChildItem -Recurse | Select-String -Pattern "TODO"

# Search multiple files
Select-String -Path "*.log" -Pattern "exception"

# Case-sensitive search
Select-String -Path "file.txt" -Pattern "Error" -CaseSensitive

# Show context (3 lines before/after)
Select-String -Path "file.txt" -Pattern "error" -Context 3,3

# Get only matching strings
Select-String -Path "file.txt" -Pattern "error" | Select-Object -ExpandProperty Line
```

### String Manipulation

```powershell
# Replace text
"Hello World" -replace "World", "PowerShell"

# Split string
"one,two,three" -split ","

# Join strings
("one", "two", "three") -join ", "

# Convert to uppercase
"hello".ToUpper()

# Convert to lowercase
"HELLO".ToLower()

# Trim whitespace
"  hello  ".Trim()

# Check if string contains
"Hello World" -like "*World*"

# Check if string matches regex
"test123" -match '\d+'
```

## Processes and Services

### Manage Processes

```powershell
# List all processes
Get-Process

# Get specific process
Get-Process -Name "chrome"

# Sort by CPU usage
Get-Process | Sort-Object CPU -Descending

# Sort by memory usage
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 10

# Kill process by name
Stop-Process -Name "notepad"

# Kill process by ID
Stop-Process -Id 1234

# Force kill
Stop-Process -Name "chrome" -Force

# Start process
Start-Process "notepad.exe"

# Start process with arguments
Start-Process "ping.exe" -ArgumentList "google.com"
```

### Manage Services

```powershell
# List all services
Get-Service

# Get specific service
Get-Service -Name "wuauserv"

# Get running services
Get-Service | Where-Object {$_.Status -eq "Running"}

# Start service
Start-Service -Name "wuauserv"

# Stop service
Stop-Service -Name "wuauserv"

# Restart service
Restart-Service -Name "wuauserv"

# Get service status
(Get-Service -Name "wuauserv").Status
```

## Network Commands

### Basic Network Info

```powershell
# IP configuration
Get-NetIPAddress

# All network adapters
Get-NetAdapter

# DNS client cache
Get-DnsClientCache

# Clear DNS cache
Clear-DnsClientCache

# Test connection (ping)
Test-Connection -ComputerName google.com

# Continuous ping
Test-Connection -ComputerName google.com -Count 10

# Test port
Test-NetConnection -ComputerName google.com -Port 443

# Trace route
Test-NetConnection -ComputerName google.com -TraceRoute

# Get public IP
(Invoke-WebRequest -Uri "https://api.ipify.org").Content
```

### Download Files

```powershell
# Download file
Invoke-WebRequest -Uri "https://example.com/file.zip" -OutFile "file.zip"

# Download with progress
$ProgressPreference = 'Continue'
Invoke-WebRequest -Uri "https://example.com/largefile.zip" -OutFile "largefile.zip"

# Make HTTP request and get content
$response = Invoke-WebRequest -Uri "https://api.example.com/data"
$response.Content

# REST API call
Invoke-RestMethod -Uri "https://api.github.com/repos/microsoft/powershell"

# POST request
Invoke-RestMethod -Uri "https://api.example.com/data" -Method POST -Body @{name="test"} -ContentType "application/json"
```

## System Information

### Get System Info

```powershell
# Computer name
$env:COMPUTERNAME

# Username
$env:USERNAME

# OS version
Get-ComputerInfo | Select-Object WindowsVersion, OSDisplayVersion

# Uptime
(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime

# Disk space
Get-PSDrive -PSProvider FileSystem

# Detailed disk info
Get-Volume

# Specific drive info
Get-Volume -DriveLetter C

# Memory info
Get-CimInstance Win32_PhysicalMemory | Measure-Object -Property Capacity -Sum

# CPU info
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
```

### Environment Variables

```powershell
# List all environment variables
Get-ChildItem Env:

# Get specific variable
$env:PATH

# Set environment variable (current session)
$env:MY_VAR = "value"

# Set environment variable (permanent)
[Environment]::SetEnvironmentVariable("MY_VAR", "value", "User")

# Append to PATH
$env:PATH += ";C:\NewPath"
```

## User and Permission Management

### User Accounts

```powershell
# List local users
Get-LocalUser

# Get current user
[System.Security.Principal.WindowsIdentity]::GetCurrent().Name

# Create local user
New-LocalUser -Name "testuser" -Password (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force)

# Delete local user
Remove-LocalUser -Name "testuser"

# Add user to group
Add-LocalGroupMember -Group "Administrators" -Member "testuser"
```

### File Permissions

```powershell
# Get ACL for file
Get-Acl -Path "C:\file.txt"

# Get ACL formatted
Get-Acl -Path "C:\file.txt" | Format-List

# Set owner
$acl = Get-Acl -Path "C:\file.txt"
$user = New-Object System.Security.Principal.NTAccount("DOMAIN\username")
$acl.SetOwner($user)
Set-Acl -Path "C:\file.txt" -AclObject $acl

# Grant permissions
$acl = Get-Acl -Path "C:\folder"
$permission = "DOMAIN\user","FullControl","Allow"
$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule $permission
$acl.SetAccessRule($accessRule)
Set-Acl -Path "C:\folder" -AclObject $acl
```

## Common Administrative Tasks

### Registry Operations

```powershell
# Read registry value
Get-ItemProperty -Path "HKLM:\Software\Microsoft\Windows\CurrentVersion" -Name "ProgramFilesDir"

# Set registry value
Set-ItemProperty -Path "HKCU:\Software\MyApp" -Name "Setting" -Value "Value"

# Create registry key
New-Item -Path "HKCU:\Software\MyApp"

# Delete registry key
Remove-Item -Path "HKCU:\Software\MyApp" -Recurse
```

### Windows Features

```powershell
# List Windows features
Get-WindowsOptionalFeature -Online

# Enable feature
Enable-WindowsOptionalFeature -Online -FeatureName "Microsoft-Windows-Subsystem-Linux"

# Disable feature
Disable-WindowsOptionalFeature -Online -FeatureName "FeatureName"
```

### Event Log

```powershell
# Get recent errors from System log
Get-EventLog -LogName System -EntryType Error -Newest 10

# Get events from Application log
Get-EventLog -LogName Application -Newest 20

# Get events after specific date
Get-EventLog -LogName System -After (Get-Date).AddDays(-1)

# Search for specific event ID
Get-EventLog -LogName System | Where-Object {$_.EventID -eq 1074}
```

## Scripting Essentials

### Variables and Arrays

```powershell
# Variable
$name = "John"

# Array
$fruits = @("apple", "banana", "orange")

# Access array element
$fruits[0]

# Array length
$fruits.Count

# Add to array
$fruits += "grape"

# Loop through array
foreach ($fruit in $fruits) {
    Write-Host $fruit
}

# Hash table (dictionary)
$person = @{
    Name = "John"
    Age = 30
    City = "New York"
}

# Access hash table
$person["Name"]
$person.Age
```

### Conditionals and Loops

```powershell
# If statement
if ($age -gt 18) {
    Write-Host "Adult"
} elseif ($age -gt 12) {
    Write-Host "Teen"
} else {
    Write-Host "Child"
}

# Switch statement
switch ($day) {
    "Monday" { Write-Host "Start of week" }
    "Friday" { Write-Host "End of week" }
    default { Write-Host "Middle of week" }
}

# For loop
for ($i = 0; $i -lt 10; $i++) {
    Write-Host $i
}

# While loop
$i = 0
while ($i -lt 10) {
    Write-Host $i
    $i++
}

# ForEach-Object (pipeline)
1..10 | ForEach-Object {
    Write-Host "Number: $_"
}
```

### Functions

```powershell
# Basic function
function Say-Hello {
    Write-Host "Hello, World!"
}

# Function with parameters
function Greet {
    param(
        [string]$Name
    )
    Write-Host "Hello, $Name!"
}

# Function with default parameters
function Greet-Advanced {
    param(
        [string]$Name = "Guest",
        [int]$Age = 0
    )
    Write-Host "Hello, $Name! You are $Age years old."
}

# Function with return value
function Add {
    param(
        [int]$a,
        [int]$b
    )
    return $a + $b
}

# Call functions
Say-Hello
Greet -Name "John"
Greet-Advanced -Name "Jane" -Age 25
$result = Add -a 5 -b 3
```

## Pipeline and Filtering

### Common Pipeline Operations

```powershell
# Where-Object (filter)
Get-Process | Where-Object {$_.CPU -gt 100}

# Select-Object (select properties)
Get-Process | Select-Object Name, CPU, WorkingSet

# Sort-Object
Get-Process | Sort-Object CPU -Descending

# Group-Object
Get-Process | Group-Object ProcessName

# Measure-Object (count, sum, average)
Get-Process | Measure-Object -Property CPU -Sum

# ForEach-Object (iterate)
Get-ChildItem | ForEach-Object {
    Write-Host $_.Name
}

# Tee-Object (output to file and pipeline)
Get-Process | Tee-Object -FilePath "processes.txt" | Where-Object {$_.CPU -gt 100}

# Out-File (export to file)
Get-Process | Out-File -FilePath "processes.txt"

# Export-Csv
Get-Process | Export-Csv -Path "processes.csv" -NoTypeInformation

# ConvertTo-Json
Get-Process | Select-Object Name, Id | ConvertTo-Json

# ConvertTo-Html
Get-Process | ConvertTo-Html | Out-File report.html
```

## Useful One-Liners

### Find Large Files

```powershell
Get-ChildItem -Recurse | Where-Object {$_.Length -gt 100MB} | Sort-Object Length -Descending | Select-Object FullName, @{Name="Size(MB)";Expression={[math]::Round($_.Length/1MB,2)}}
```

### Find Old Files

```powershell
Get-ChildItem -Recurse | Where-Object {$_.LastWriteTime -lt (Get-Date).AddDays(-90)} | Select-Object FullName, LastWriteTime
```

### Count Files by Extension

```powershell
Get-ChildItem -Recurse | Group-Object Extension | Sort-Object Count -Descending | Select-Object Count, Name
```

### Get Folder Size

```powershell
Get-ChildItem -Recurse | Measure-Object -Property Length -Sum | Select-Object @{Name="Size(GB)";Expression={[math]::Round($_.Sum/1GB,2)}}
```

### Find Duplicate Files

```powershell
Get-ChildItem -Recurse -File | Group-Object -Property Name | Where-Object {$_.Count -gt 1} | ForEach-Object {$_.Group | Select-Object FullName}
```

### Check if Running as Administrator

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

### Get Installed Software

```powershell
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, Publisher | Sort-Object DisplayName
```

### Kill All Processes by Name

```powershell
Get-Process -Name "chrome" | Stop-Process -Force
```

## Execution Policy

```powershell
# Check current execution policy
Get-ExecutionPolicy

# Set execution policy (run as admin)
Set-ExecutionPolicy RemoteSigned

# Bypass for single script
PowerShell.exe -ExecutionPolicy Bypass -File script.ps1

# Common policies:
# Restricted: No scripts allowed
# RemoteSigned: Local scripts allowed, downloaded scripts must be signed
# Unrestricted: All scripts allowed
```

## Modules

```powershell
# List installed modules
Get-Module -ListAvailable

# Import module
Import-Module ModuleName

# Find module in gallery
Find-Module -Name "Posh-SSH"

# Install module from gallery
Install-Module -Name "Posh-SSH"

# Update module
Update-Module -Name "Posh-SSH"

# Remove module
Uninstall-Module -Name "Posh-SSH"
```

## My Most-Used Aliases

```powershell
# Create alias
Set-Alias -Name ll -Value Get-ChildItem

# List all aliases
Get-Alias

# Common built-in aliases:
# ls, dir -> Get-ChildItem
# cd -> Set-Location
# pwd -> Get-Location
# cat -> Get-Content
# cp -> Copy-Item
# mv -> Move-Item
# rm -> Remove-Item
# ps -> Get-Process
# kill -> Stop-Process
# cls, clear -> Clear-Host
```

## Quick Tips

### Measure Command Execution Time

```powershell
Measure-Command {
    Get-ChildItem -Recurse
}
```

### Get Command Help

```powershell
# Basic help
Get-Help Get-Process

# Detailed help
Get-Help Get-Process -Detailed

# Examples
Get-Help Get-Process -Examples

# Online help
Get-Help Get-Process -Online
```

### Find Commands

```powershell
# Find commands with keyword
Get-Command -Name "*process*"

# Find commands in module
Get-Command -Module Microsoft.PowerShell.Management

# Find all cmdlets
Get-Command -CommandType Cmdlet
```

## Resources

- [PowerShell Documentation - Microsoft](https://learn.microsoft.com/en-us/powershell/)
- [PowerShell Gallery](https://www.powershellgallery.com/)
- [SS64 PowerShell Reference](https://ss64.com/ps/)

---

*Got a PowerShell command I should add? [Let me know](mailto:jordan@jordananderson.us).*
