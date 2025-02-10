# Windows System Health Monitoring Script
# Author: Al3ks

# Set log file paths (Fixed to user-accessible location)
$LogFile = "$env:USERPROFILE\Documents\SystemHealthLog.txt"
$CSVFile = "$env:USERPROFILE\Documents\SystemHealthData.csv"

# Set monitoring interval in seconds
$Interval = 60  

# Function to get CPU usage
function Get-CPUUsage {
    $cpuLoad = Get-Counter '\Processor(_Total)\% Processor Time' | Select-Object -ExpandProperty CounterSamples | Select-Object -ExpandProperty CookedValue
    return [math]::Round($cpuLoad, 2)
}

# Function to get memory usage
function Get-MemoryUsage {
    $mem = Get-CimInstance Win32_OperatingSystem
    $totalMem = $mem.TotalVisibleMemorySize / 1MB
    $freeMem = $mem.FreePhysicalMemory / 1MB
    $usedMem = $totalMem - $freeMem
    $usagePercent = ($usedMem / $totalMem) * 100
    return @{UsedMB=[math]::Round($usedMem,2); FreeMB=[math]::Round($freeMem,2); UsagePercent=[math]::Round($usagePercent,2)}
}

# Function to get disk usage
function Get-DiskUsage {
    $disks = Get-PSDrive -PSProvider FileSystem | Select-Object Name, @{Name="FreeGB"; Expression={$_.Free/1GB}}, @{Name="UsedGB"; Expression={($_.Used/1GB)}}
    return $disks
}

# Function to get network status (Fixed `if` statement)
function Get-NetworkStatus {
    $net = Get-NetAdapter | Where-Object { $_.Status -eq "Up" }
    if ($net) {
        return "Connected"
    } else {
        return "Disconnected"
    }
}

# Function to get system uptime
function Get-SystemUptime {
    $uptime = (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
    $uptimeSpan = (Get-Date) - $uptime
    return @{Days=$uptimeSpan.Days; Hours=$uptimeSpan.Hours; Minutes=$uptimeSpan.Minutes}
}

# Function to get top 5 memory-consuming processes
function Get-TopProcesses {
    $processes = Get-Process | Sort-Object -Descending WS | Select-Object -First 5 Name, ID, @{Name="MemoryMB"; Expression={[math]::Round($_.WS/1MB,2)}}
    return $processes
}

# Create log files if they don’t exist
if (!(Test-Path $LogFile)) { New-Item -Path $LogFile -ItemType File -Force }
if (!(Test-Path $CSVFile)) { "Timestamp,CPU Usage,Memory Usage,Network Status" | Out-File -FilePath $CSVFile }

# Monitoring Loop
Write-Host "System Health Monitoring started. Logging to $LogFile every $Interval seconds."

while ($true) {
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $cpuUsage = Get-CPUUsage
    $memoryUsage = Get-MemoryUsage
    $diskUsage = Get-DiskUsage
    $networkStatus = Get-NetworkStatus
    $uptime = Get-SystemUptime
    $topProcesses = Get-TopProcesses

    # Create log entry
    $logEntry = @"
-------------------------------
Timestamp: $timestamp
CPU Usage: $cpuUsage %
Memory Usage: Used $($memoryUsage.UsedMB) MB / Free $($memoryUsage.FreeMB) MB ($($memoryUsage.UsagePercent) %)
Network Status: $networkStatus
System Uptime: $($uptime.Days) days, $($uptime.Hours) hours, $($uptime.Minutes) minutes
Top 5 Processes (Memory Usage):
$($topProcesses | Format-Table -AutoSize | Out-String)
-------------------------------
"@

    # Append to log file (Fixed access issues)
    $logEntry | Out-File -Append -FilePath $LogFile -Encoding UTF8

    # Append CSV data for Graphing
    "$timestamp,$cpuUsage,$($memoryUsage.UsagePercent),$networkStatus" | Out-File -Append -FilePath $CSVFile -Encoding UTF8

    # Wait before next check
    Start-Sleep -Seconds $Interval
}
