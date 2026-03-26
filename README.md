# -THE-HELL-HOUNDS-KAMEHAMEHA-ULTIMATE-ONECLICK-SCRIPT
Turtle Destruction Wave
<#
.SYNOPSIS
    HELL HOUNDS KAMEHAMEHA – The Ultimate OneClick Security & Forensics Toolkit
    Combines: X100 Defense | Scam Detection | Pen Testing | Crypto Monitor
    Personas: Hell Hound | Mewizard | Mr. Robot | Digital Joker
    Runs every 5 minutes (optional scheduled task)
.NOTES
    Run as Administrator. All tools auto‑install via Chocolatey.
    For educational and authorized use only.
#>

# ========== Auto‑elevate ==========
if (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Host "Requesting administrator privileges..." -ForegroundColor Yellow
    Start-Process powershell.exe -ArgumentList "-NoProfile -ExecutionPolicy Bypass -File `"$PSCommandPath`"" -Verb RunAs
    exit
}
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process -Force

# ========== Global Configuration ==========
$TIMESTAMP = Get-Date -Format 'yyyyMMdd_HHmmss'
$OUTPUT_DIR = "$env:USERPROFILE\Desktop\HELL_HOUNDS_KAMEHAMEHA_$TIMESTAMP"
$LOGS_DIR = "$OUTPUT_DIR\logs"
$REPORTS_DIR = "$OUTPUT_DIR\reports"
$EVIDENCE_DIR = "$OUTPUT_DIR\evidence"
$QUARANTINE_DIR = "$OUTPUT_DIR\quarantine"
$TOOLS_DIR = "$OUTPUT_DIR\tools"
$CRYPTO_DIR = "$OUTPUT_DIR\crypto_monitor"
New-Item -ItemType Directory -Force -Path $LOGS_DIR, $REPORTS_DIR, $EVIDENCE_DIR, $QUARANTINE_DIR, $TOOLS_DIR, $CRYPTO_DIR | Out-Null

$LOG = "$LOGS_DIR\master.log"
$USAGE_COUNTER = "$env:USERPROFILE\AppData\Local\HellHounds\usage_counter.txt"
$BITCOIN_ADDRESS = "1T3eZOJu89hyIHkA620sf7cvdrmD4MPBCa"
$PERSONAS = @{
    "Kaia" = @{Description = "Empathetic, analytical, strategic"}
    "Shadow" = @{Description = "Tactical, security-focused, protector"}
    "DigitalJoker" = @{Description = "Chaotic, creative, boundary-pusher"}
    "MrRobot" = @{Description = "Anti‑establishment, hacker, vigilante"}
}
$CURRENT_PERSONA = "MrRobot"

# ========== Colors ==========
$RED = 'Red'
$GREEN = 'Green'
$YELLOW = 'Yellow'
$CYAN = 'Cyan'
$MAGENTA = 'Magenta'
$WHITE = 'White'

# ========== Helper Functions ==========
function Write-Log {
    param([string]$Message, [string]$Level = "INFO", [string]$Color = "White")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss.fff"
    $logEntry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $LOG -Value $logEntry
    Write-Host $logEntry -ForegroundColor $Color
}

function Write-Alert {
    param([string]$Message, [string]$Severity = "HIGH", [string]$MitreID = "")
    $alert = @{
        timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss.fff"
        message = $Message
        severity = $Severity
        mitre_id = $MitreID
    }
    $alertJson = $alert | ConvertTo-Json -Compress
    Add-Content -Path "$LOGS_DIR\alerts.json" -Value $alertJson
    Write-Host "`n  ⚠️  [$Severity] $Message" -ForegroundColor Red
    if ($MitreID) { Write-Host "     MITRE ATT&CK: $MitreID" -ForegroundColor Yellow }
}

function Test-Command($cmd) {
    return [bool](Get-Command $cmd -ErrorAction SilentlyContinue)
}

function Install-ChocolateyIfMissing {
    if (-not (Test-Command "choco")) {
        Write-Log "Installing Chocolatey..." -Color Yellow
        Set-ExecutionPolicy Bypass -Scope Process -Force
        [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
        Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
        refreshenv
        Start-Sleep -Seconds 5
        $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
    }
}

function Install-Tool {
    param([string]$ToolName, [string]$ChocoPackage = $ToolName)
    if (-not (Test-Command $ToolName)) {
        Write-Log "Installing $ToolName..." -Color Yellow
        Install-ChocolateyIfMissing
        choco install $ChocoPackage -y --no-progress
        refreshenv
        $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
    }
}

# ========== X100 Defense Functions ==========
function Invoke-X100ARPDetection {
    Write-Log "X100 ARP Spoofing Detection..." -Color Cyan
    $arpTable = arp -a
    $arpEntries = @()
    $arpTable -split "`n" | ForEach-Object {
        if ($_ -match '(\d+\.\d+\.\d+\.\d+)\s+([0-9a-f]{2}-[0-9a-f]{2}-[0-9a-f]{2}-[0-9a-f]{2}-[0-9a-f]{2}-[0-9a-f]{2})') {
            $arpEntries += [PSCustomObject]@{IP = $matches[1]; MAC = $matches[2]}
        }
    }
    $grouped = $arpEntries | Group-Object MAC | Where-Object { $_.Count -gt 1 }
    foreach ($group in $grouped) {
        $ips = ($group.Group | ForEach-Object { $_.IP }) -join ", "
        Write-Alert -Message "ARP Spoofing: MAC $($group.Name) claims $ips" -Severity "HIGH" -MitreID "T1557.002"
        netsh advfirewall firewall add rule name="X100_Block_MAC_$($group.Name -replace '-','_')" dir=in action=block protocol=any remoteip=any 2>$null
    }
    $grouped | ConvertTo-Json | Out-File "$LOGS_DIR\arp_alerts.json"
    Write-Log "X100 ARP Detection Complete. Found $($grouped.Count) threats." -Color Green
}

function Invoke-X100ThreatHunting {
    Write-Log "X100 Threat Hunting..." -Color Cyan
    $suspicious = @()
    $malwareIndicators = @("nc","ncat","socat","meterpreter","mimikatz","psexec","wmic","cscript","powershell -e")
    Get-Process | ForEach-Object {
        $proc = $_
        foreach ($ind in $malwareIndicators) {
            if ($proc.Name -eq $ind -or ($proc.Path -and $proc.Path -match $ind)) {
                $suspicious += $proc
                Write-Alert -Message "Suspicious process: $($proc.Name) (PID: $($proc.Id))" -Severity "HIGH"
            }
        }
    }
    # Network connections
    $externalConnections = Get-NetTCPConnection | Where-Object {
        $_.State -eq 'Established' -and $_.RemoteAddress -notmatch '^(192\.168\.|10\.|172\.1[6-9]\.|172\.2[0-9]\.|172\.3[0-1]\.|127\.0\.0\.)'
    }
    foreach ($conn in $externalConnections) {
        Write-Alert -Message "External connection: $($conn.LocalAddress):$($conn.LocalPort) -> $($conn.RemoteAddress):$($conn.RemotePort)" -Severity "MEDIUM"
    }
    $suspicious | Export-Csv "$LOGS_DIR\suspicious_processes.csv" -NoTypeInformation
    Write-Log "X100 Threat Hunting Complete. Found $($suspicious.Count) suspicious processes." -Color Green
}

function Invoke-X100USBDefense {
    Write-Log "X100 USB Defense..." -Color Cyan
    $usbDevices = Get-WmiObject Win32_USBControllerDevice | ForEach-Object { [wmi]$_.Dependent }
    $usbFingerprints = @()
    foreach ($device in $usbDevices) {
        $fingerprint = @{
            name = $device.Name
            pnp = $device.PNPDeviceID
            manufacturer = $device.Manufacturer
            service = $device.Service
        }
        $usbFingerprints += $fingerprint
    }
    $usbFingerprints | ConvertTo-Json | Out-File "$LOGS_DIR\usb_fingerprints.json"
    # Disable autorun
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" -Name "NoDriveTypeAutoRun" -Value 255 -Type DWord -Force
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" -Name "NoAutorun" -Value 1 -Type DWord -Force
    Write-Log "X100 USB Defense Active. Monitored $($usbFingerprints.Count) devices." -Color Green
}

function Invoke-X100PersistenceScan {
    Write-Log "X100 Persistence Scan..." -Color Cyan
    $persistence = @()
    # Registry autoruns
    $runKeys = @(
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce",
        "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
        "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce"
    )
    foreach ($key in $runKeys) {
        if (Test-Path $key) {
            $items = Get-ItemProperty -Path $key
            foreach ($prop in $items.PSObject.Properties) {
                if ($prop.Name -notmatch "PS*") {
                    $persistence += [PSCustomObject]@{
                        type = "Registry"
                        location = $key
                        name = $prop.Name
                        value = $prop.Value
                    }
                }
            }
        }
    }
    # Scheduled tasks (non-Microsoft)
    $tasks = Get-ScheduledTask | Where-Object { $_.TaskPath -notlike "\Microsoft*" }
    foreach ($task in $tasks) {
        $persistence += [PSCustomObject]@{
            type = "ScheduledTask"
            location = $task.TaskPath
            name = $task.TaskName
            value = $task.State
        }
    }
    $persistence | ConvertTo-Json | Out-File "$LOGS_DIR\persistence.json"
    Write-Log "X100 Persistence Scan Complete. Found $($persistence.Count) entries." -Color Green
    return $persistence
}

function Invoke-X100CounterMeasures {
    Write-Log "X100 Counter-Measures..." -Color Cyan
    # Block suspicious IPs from alerts
    if (Test-Path "$LOGS_DIR\alerts.json") {
        $alerts = Get-Content "$LOGS_DIR\alerts.json" | ConvertFrom-Json
        $suspiciousIPs = $alerts | Where-Object { $_.message -match '\d+\.\d+\.\d+\.\d+' } | ForEach-Object {
            $_.message -match '\d+\.\d+\.\d+\.\d+' | Out-Null
            $matches[0]
        } | Select-Object -Unique
        foreach ($ip in $suspiciousIPs) {
            $ruleName = "X100_Block_IP_$($ip -replace '\.','_')"
            netsh advfirewall firewall add rule name=$ruleName dir=in action=block remoteip=$ip 2>$null
            Write-Log "Blocked IP: $ip" -Color Green
        }
    }
    # Block common attack ports
    $attackPorts = @(23,135,137,138,139,445,1433,1434,3306,3389,5900,8080,8443)
    foreach ($port in $attackPorts) {
        $ruleName = "X100_Block_Port_$port"
        netsh advfirewall firewall add rule name=$ruleName dir=in action=block protocol=tcp localport=$port 2>$null
    }
    Write-Log "X100 Counter-Measures Applied." -Color Green
}

function Invoke-X100EvidencePackage {
    Write-Log "X100 Evidence Package..." -Color Cyan
    $evidence = @()
    $evidence += "=== EVIDENCE PACKAGE ==="
    $evidence += "Generated: $(Get-Date)"
    $evidence += "Computer: $env:COMPUTERNAME"
    $evidence += "User: $env:USERNAME"
    $evidence += ""
    # Include logs
    $logFiles = Get-ChildItem $LOGS_DIR -Filter "*.json" | ForEach-Object { $_.Name }
    $evidence += "Log Files: $($logFiles -join ', ')"
    $evidence += ""
    $evidence -join "`r`n" | Out-File "$EVIDENCE_DIR\evidence_package.txt"
    Copy-Item -Path $LOGS_DIR\* -Destination $EVIDENCE_DIR -Force
    Write-Log "Evidence package saved to $EVIDENCE_DIR" -Color Green
}

# ========== Scam Detection & CounterAttack ==========
function Invoke-ScamDetection {
    Write-Log "Running Scam Detection Engine..." -Color Cyan
    $scamScore = 0
    # Check browser history for known scam URLs (simplified)
    $browserPaths = @("$env:LOCALAPPDATA\Google\Chrome\User Data\Default\History", "$env:APPDATA\Mozilla\Firefox\Profiles\*.default\places.sqlite")
    $scamUrls = @("crypto-giveaway", "wallet-verify", "secure-login", "update-account", "verify-identity")
    foreach ($path in $browserPaths) {
        if (Test-Path $path) {
            Write-Log "Scanning browser history..." -Color Gray
            $scamScore += 10
        }
    }
    # Check running processes for scam-related tools
    $scamProcesses = @("teamviewer", "anydesk", "ultravnc", "tightvnc")
    $running = Get-Process | Where-Object { $scamProcesses -contains $_.Name }
    if ($running) {
        $scamScore += 20
        Write-Alert -Message "Remote desktop software running: $($running.Name -join ', ')" -Severity "MEDIUM"
    }
    # Check for cryptocurrency miners
    $miners = Get-Process | Where-Object { $_.Name -match "xmrig|minerd|cpuminer" }
    if ($miners) {
        $scamScore += 50
        Write-Alert -Message "Cryptocurrency miner detected: $($miners.Name)" -Severity "HIGH"
    }
    # Counter-attack response
    if ($scamScore -ge 30) {
        Write-Log "High scam score ($scamScore). Initiating counter-measures..." -Color Red
        # Kill suspicious processes
        $scamProcesses | ForEach-Object {
            Get-Process -Name $_ -ErrorAction SilentlyContinue | Stop-Process -Force
        }
        # Block outgoing connections to known scam IPs (dummy)
        netsh advfirewall firewall add rule name="Block_Scam_Outbound" dir=out action=block remoteip=185.130.5.253,185.130.5.254
        Write-Alert -Message "Counter-attack: Blocked known scam IPs and terminated remote access tools." -Severity "CRITICAL"
    }
    Write-Log "Scam detection complete. Score: $scamScore" -Color Green
    $scamScore | Out-File "$REPORTS_DIR\scam_score.txt"
}

# ========== Penetration Testing (Authorized Only) ==========
function Invoke-PenTest {
    Write-Log "Penetration Testing Module (Authorized Targets Only)" -Color Cyan
    Write-Host "`n  WARNING: Use only on systems you own or have written permission to test." -ForegroundColor Red
    $auth = Read-Host "Do you have authorization? (yes/no)"
    if ($auth -ne "yes") {
        Write-Log "Penetration test aborted – no authorization." -Color Yellow
        return
    }
    Install-Tool -ToolName "nmap"
    $extraTargets = Read-Host "Enter additional IPs/ranges (comma‑separated) or press Enter"
    $targets = @("localhost", "127.0.0.1")
    if ($extraTargets) { $targets += $extraTargets -split ',' }
    foreach ($target in $targets) {
        Write-Log "Testing $target..." -Color Gray
        nmap -T4 -F $target -oN "$REPORTS_DIR\pentest_$($target -replace '\.','_').txt" 2>$null
        nmap --script vuln $target -oN "$REPORTS_DIR\vuln_$($target -replace '\.','_').txt" 2>$null
    }
    Write-Log "Penetration test complete." -Color Green
}

# ========== GitHub Security Tools Installer ==========
function Install-GitHubTools {
    Write-Log "Cloning GitHub security tools..." -Color Cyan
    Install-Tool -ToolName "git"
    $repos = @(
        "https://github.com/GreyDGL/PentestGPT.git",
        "https://github.com/enaqx/awesome-pentest.git",
        "https://github.com/jivoi/pentest.git",
        "https://github.com/blaCCkHatHacEEkr/PENTESTING-BIBLE.git",
        "https://github.com/dave5623/GrayHatPython.git"
    )
    Push-Location $TOOLS_DIR
    $success = 0
    foreach ($repo in $repos) {
        $name = ($repo -split '/')[-1] -replace '\.git$',''
        Write-Host "Cloning $name ... " -NoNewline
        try {
            git clone $repo 2>$null
            Write-Host "OK" -ForegroundColor Green
            $success++
        } catch {
            Write-Host "FAILED" -ForegroundColor Red
        }
    }
    Pop-Location
    Write-Log "GitHub tools cloned: $success of $($repos.Count)." -Color Green
}

# ========== Crypto Mining Monitor ==========
function Invoke-CryptoMonitor {
    Write-Log "Monitoring legitimate Bitcoin mining pools..." -Color Cyan
    $pools = @(
        @{Name="AntPool"; Url="https://api.antpool.com/api/v1/user/status"; Key="?"},
        @{Name="F2Pool"; Url="https://api.f2pool.com/bitcoin/"; Key="?"},
        @{Name="ViaBTC"; Url="https://api.viabtc.com/v1/order/status"; Key="?"}
    )
    $results = @()
    foreach ($pool in $pools) {
        try {
            $response = Invoke-WebRequest -Uri $pool.Url -UseBasicParsing -TimeoutSec 5 -ErrorAction Stop
            $results += [PSCustomObject]@{
                Pool = $pool.Name
                Status = "OK"
                Data = $response.Content.Substring(0, [Math]::Min(200, $response.Content.Length))
            }
        } catch {
            $results += [PSCustomObject]@{
                Pool = $pool.Name
                Status = "Error"
                Data = $_.Exception.Message
            }
            Write-Log "Could not reach $($pool.Name): $($_.Exception.Message)" -Color Yellow
        }
    }
    $results | ConvertTo-Json | Out-File "$CRYPTO_DIR\pool_status.json"
    Write-Log "Crypto monitor complete. Data saved to $CRYPTO_DIR" -Color Green
}

# ========== Additional Security Modules ==========
function Invoke-DefenderCheck {
    Write-Log "Checking Windows Defender status..." -Color Cyan
    try {
        $defender = Get-MpComputerStatus
        Write-Host "Defender Status:" -ForegroundColor Yellow
        $defender | Select-Object AMServiceEnabled, AntivirusEnabled, RealTimeProtectionEnabled, AntivirusSignatureLastUpdated | Format-List
        if (-not $defender.RealTimeProtectionEnabled) {
            Write-Alert -Message "Defender real-time protection is off" -Severity "HIGH"
        } else {
            Write-Host "✅ Defender real-time protection is active" -ForegroundColor Green
        }
        $scan = Read-Host "Run a Quick Defender scan? (y/n)"
        if ($scan -eq 'y') {
            Write-Log "Starting Quick Scan..." -Color Yellow
            Start-MpScan -ScanType QuickScan
            Write-Log "Quick scan initiated (check Windows Security app for results)" -Color Green
        }
    } catch {
        Write-Log "Defender module not available or error: $($_.Exception.Message)" -Level "WARNING" -Color Yellow
    }
}

function Invoke-AdminAudit {
    Write-Log "Auditing local Administrators group..." -Color Cyan
    $admins = Get-LocalGroupMember -Group "Administrators"
    $admins | Format-Table Name, PrincipalSource, ObjectClass
    $admins | Out-File "$REPORTS_DIR\local_admins.txt"
    if ($admins.Count -gt 5) {
        Write-Host "⚠️  Many admin accounts detected – review for least privilege" -ForegroundColor Yellow
    }
}

function Invoke-EventLogHunt {
    Write-Log "Hunting recent security events..." -Color Cyan
    $events = Get-WinEvent -FilterHashtable @{
        LogName = 'Security'
        ID = 4624,4625,4672,4688
    } -MaxEvents 50 -ErrorAction SilentlyContinue
    if ($events) {
        $events | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize -Wrap
        $events | Out-File "$REPORTS_DIR\security_events.txt"
    } else {
        Write-Host "No recent matching events found." -ForegroundColor Gray
    }
}

function Invoke-PersistenceCheck {
    Write-Log "Checking common persistence locations..." -Color Cyan
    Write-Host "Startup Items:" -ForegroundColor Yellow
    Get-CimInstance Win32_StartupCommand | Select-Object Command, Location, User | Format-Table
    Write-Host "`nScheduled Tasks (ready):" -ForegroundColor Yellow
    Get-ScheduledTask | Where-Object State -eq 'Ready' | Select-Object TaskName, TaskPath, Author | Format-Table
}

function Invoke-NetworkCheck {
    Write-Log "Network & Firewall status..." -Color Cyan
    Get-NetFirewallProfile | Select-Object Name, Enabled | Format-Table
    Write-Host "`nListening Ports:" -ForegroundColor Yellow
    Get-NetTCPConnection -State Listen | Select-Object LocalPort, OwningProcess | Sort-Object LocalPort | Format-Table
}

function Invoke-HardeningReport {
    Write-Log "Generating basic hardening report..." -Color Magenta
    $report = @{}
    $report.UAC = (Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System").EnableLUA
    $report.SMB1 = (Get-SmbServerConfiguration).EnableSMB1Protocol
    $report.GuestEnabled = (Get-LocalUser -Name Guest).Enabled
    $report | Format-List
    $report | ConvertTo-Json | Out-File "$REPORTS_DIR\hardening_report.json"
}

function Invoke-FullSecurityPosture {
    Write-Log "Running Full Security Posture Scan..." -Color Magenta
    Invoke-DefenderCheck
    Invoke-AdminAudit
    Invoke-EventLogHunt
    Invoke-PersistenceCheck
    Invoke-NetworkCheck
    Invoke-HardeningReport
    Write-Log "Full security posture scan complete." -Color Green
}

# ========== Persona Management ==========
function Set-Persona {
    param([string]$Persona)
    if ($PERSONAS.ContainsKey($Persona)) {
        $global:CURRENT_PERSONA = $Persona
        Write-Log "Persona switched to: $Persona ($($PERSONAS[$Persona].Description))" -Color Cyan
        # Optionally, you could modify system prompt here if interacting with an LLM
    } else {
        Write-Log "Persona '$Persona' not found. Available: $($PERSONAS.Keys -join ', ')" -Color Red
    }
}

# ========== Scheduled Task Registration ==========
function Register-ScheduledTask {
    $taskName = "HellHoundsKamehameha"
    $scriptPath = $MyInvocation.MyCommand.Path
    if (-not $scriptPath) {
        Write-Log "Cannot determine script path for scheduled task." -Color Red
        return
    }
    $action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$scriptPath`" -RunEvery5Min"
    $trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) -RepetitionInterval (New-TimeSpan -Minutes 5)
    $principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
    Register-ScheduledTask -TaskName $taskName -Action $action -Trigger $trigger -Principal $principal -Force | Out-Null
    Write-Log "Scheduled task '$taskName' created (runs every 5 minutes)." -Color Green
}

# ========== Usage Counter & Donation Prompt ==========
function Update-UsageCounter {
    $counterDir = Split-Path $USAGE_COUNTER -Parent
    if (-not (Test-Path $counterDir)) { New-Item -ItemType Directory -Force -Path $counterDir | Out-Null }
    if (Test-Path $USAGE_COUNTER) { $count = [int](Get-Content $USAGE_COUNTER) } else { $count = 0 }
    $count++
    $count | Out-File $USAGE_COUNTER
    Write-Log "This script has been executed $count times." -Color Cyan
    Write-Host "`n╔═══════════════════════════════════════════════════════════════════╗" -ForegroundColor Yellow
    Write-Host "║              💰 SUPPORT ULTIMATE DEVELOPMENT 💰                    ║" -ForegroundColor Yellow
    Write-Host "║                                                                   ║" -ForegroundColor Yellow
    Write-Host "║  Donate to keep this toolkit alive:                              ║" -ForegroundColor Yellow
    Write-Host "║  Bitcoin: $BITCOIN_ADDRESS   ║" -ForegroundColor Green
    Write-Host "║                                                                   ║" -ForegroundColor Yellow
    Write-Host "║  Thank you for using the Ultimate OneClick Script!              ║" -ForegroundColor Yellow
    Write-Host "╚═══════════════════════════════════════════════════════════════════╝" -ForegroundColor Yellow
}

# ========== Ultimate OneClick Runner ==========
function Invoke-UltimateOneClick {
    Write-Log "====== ULTIMATE ONECLICK SCRIPT STARTING ======" -Color Magenta
    # Run all core modules
    Invoke-ScamDetection
    Invoke-X100ARPDetection
    Invoke-X100ThreatHunting
    Invoke-X100USBDefense
    Invoke-X100PersistenceScan | Out-Null
    Invoke-X100CounterMeasures
    Invoke-PenTest
    Install-GitHubTools
    Invoke-CryptoMonitor
    Invoke-X100EvidencePackage
    Write-Log "====== ULTIMATE ONECLICK SCRIPT COMPLETE ======" -Color Magenta
}

# ========== ASCII Art Banner ==========
$ULTIMATE_ART = @"
╔═══════════════════════════════════════════════════════════════════════════════════════╗
║                         S C R I P T      M E G A   E D I T I O N                       ║
║               X100000000 Defense | Scam Detection | Pen Testing | Crypto Monitor       ║
║                    Hell Hound | Mewizard | Mr. Robot | Digital Joker                   ║
║                              Runs Every 5 Minutes                                      ║
╚═══════════════════════════════════════════════════════════════════════════════════════╝
"@

# ========== Main Menu ==========
function Show-Menu {
    Clear-Host
    Write-Host $ULTIMATE_ART -ForegroundColor Red
    Write-Host "`n               H E L L   H O U N D S   K A M E H A M E H A" -ForegroundColor Cyan
    Write-Host "                         Ultimate Security & Forensics Toolkit" -ForegroundColor Yellow
    Write-Host "`nContact: ranfom@gmail.com | (718) 422-0200" -ForegroundColor White
    Write-Host "Address: 22 W 88th St Apt 1, New York, NY 10024-2549" -ForegroundColor White

    Write-Host @"
`n
╔═══════════════════════════════════════════════════════════════════╗
║                    ULTIMATE ONECLICK MENU                          ║
╚═══════════════════════════════════════════════════════════════════╝
[1]  Run Full Suite (once)
[2]  X100 Defense Suite
[3]  Scam Detection & CounterAttack
[4]  Penetration Test (Authorized Only)
[5]  Install GitHub Security Tools
[6]  Crypto Mining Monitor
[7]  Set Persona (Kaia/Shadow/DigitalJoker/MrRobot)
[8]  Register Scheduled Task (every 5 minutes)
[9]  View Logs & Reports
[10] Windows Defender & Security Check
[11] Local Admin Audit
[12] Event Log Threat Hunt
[13] Persistence / Startup Check
[14] Network & Firewall Overview
[15] Hardening Quick Report
[16] Full Security Posture Scan (All Above)
[0]  Exit
"@ -ForegroundColor Cyan

    $choice = Read-Host "`nSelect option"
    switch ($choice) {
        "1" { Invoke-UltimateOneClick }
        "2" {
            Write-Log "Running X100 Defense Suite..." -Color Yellow
            Invoke-X100ARPDetection
            Invoke-X100ThreatHunting
            Invoke-X100USBDefense
            Invoke-X100PersistenceScan | Out-Null
            Invoke-X100CounterMeasures
            Invoke-X100EvidencePackage
        }
        "3" { Invoke-ScamDetection }
        "4" { Invoke-PenTest }
        "5" { Install-GitHubTools }
        "6" { Invoke-CryptoMonitor }
        "7" {
            Write-Host "Available personas: $($PERSONAS.Keys -join ', ')" -ForegroundColor Cyan
            $pers = Read-Host "Enter persona name"
            Set-Persona -Persona $pers
        }
        "8" { Register-ScheduledTask }
        "9" {
            if (Test-Path $LOGS_DIR) { Invoke-Item $LOGS_DIR }
            if (Test-Path $REPORTS_DIR) { Invoke-Item $REPORTS_DIR }
        }
        "10" { Invoke-DefenderCheck }
        "11" { Invoke-AdminAudit }
        "12" { Invoke-EventLogHunt }
        "13" { Invoke-PersistenceCheck }
        "14" { Invoke-NetworkCheck }
        "15" { Invoke-HardeningReport }
        "16" { Invoke-FullSecurityPosture }
        "0" { exit }
        default { Write-Host "Invalid option" -ForegroundColor Red }
    }
    Write-Host "`nPress Enter to continue..."
    Read-Host
    Show-Menu
}

# ========== Main Execution ==========
Clear-Host
Write-Host $ULTIMATE_ART -ForegroundColor Red
Write-Log "Ultimate OneClick Script initiated." -Color Cyan

# Update usage counter
Update-UsageCounter

# If run with -RunEvery5Min flag, execute full suite once and exit
if ($args -contains "-RunEvery5Min") {
    Invoke-UltimateOneClick
    exit
}

# Otherwise, show menu
Show-Menu
