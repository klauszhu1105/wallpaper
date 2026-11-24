
<#
Monitors total CPU usage and restarts the LEAPWORK_CONTROLLER
service when CPU usage remains above the configured threshold.

Features:
- Sustained CPU threshold detection
- Restart cooldown protection
- CPU counter error handling
- Service status verification
- Throttled logging
- Automatic log rotation
#>

param(
    [string]$ServiceName       = "LEAPWORK_CONTROLLER",
    [double]$CpuThreshold      = 70,
    [int]$SustainedSeconds     = 60,
    [int]$PollSeconds          = 5,
    [int]$CooldownSeconds      = 300,
    [int]$LogEveryNSeconds     = 30,
    [int]$MaxLogSizeMB         = 5,
    [int]$MaxLogFiles          = 3,
    [int]$ServiceTimeout       = 60,
    [string]$LogPath           = "$PSScriptRoot\Monitor-LeapworkController.log"
)

# Rotate log files when the maximum size is reached.
function Rotate-LogIfNeeded {
    if (-not (Test-Path -LiteralPath $LogPath)) {
        return
    }

    $sizeMB = (Get-Item -LiteralPath $LogPath).Length / 1MB

    if ($sizeMB -lt $MaxLogSizeMB) {
        return
    }

    # Remove the oldest backup first.
    $oldest = "$LogPath.$MaxLogFiles"

    if (Test-Path -LiteralPath $oldest) {
        Remove-Item -LiteralPath $oldest -Force -ErrorAction Stop
    }

    # Shift existing backups.
    for ($i = $MaxLogFiles - 1; $i -ge 1; $i--) {
        $src = "$LogPath.$i"
        $dst = "$LogPath.$($i + 1)"

        if (Test-Path -LiteralPath $src) {
            Move-Item -LiteralPath $src `
                -Destination $dst -Force -ErrorAction Stop
        }
    }

    Move-Item -LiteralPath $LogPath `
        -Destination "$LogPath.1" -Force -ErrorAction Stop
}

# Write timestamped messages to the log.
function Write-Log {
    param([string]$Message)

    $line = "{0:yyyy-MM-dd HH:mm:ss} [{1}] {2}" -f `
        (Get-Date), $ServiceName, $Message

    try {
        Rotate-LogIfNeeded

        Add-Content -LiteralPath $LogPath `
            -Value $line -ErrorAction Stop
    }
    catch {
        Write-Warning "Log write failed: $($_.Exception.Message)"
    }

    Write-Host $line
}

# Read total system CPU usage.
function Get-CpuPercent {
    $counter = Get-Counter `
        '\Processor(_Total)\% Processor Time' `
        -ErrorAction Stop

    $value = $counter.CounterSamples[0].CookedValue

    if ([double]::IsNaN($value) -or
        [double]::IsInfinity($value) -or
        $value -lt 0 -or $value -gt 100) {

        throw "Invalid CPU counter value: $value"
    }

    return $value
}

# Restart the service and verify its status.
function Restart-LeapworkController {
    try {
        Write-Log "Restarting service '$ServiceName'..."

        Restart-Service -Name $ServiceName `
            -ErrorAction Stop

        $service = Get-Service -Name $ServiceName `
            -ErrorAction Stop

        $service.WaitForStatus(
            [System.ServiceProcess.ServiceControllerStatus]::Running,
            [TimeSpan]::FromSeconds($ServiceTimeout)
        )

        $service.Refresh()

        if ($service.Status -ne 'Running') {
            throw "Service status is $($service.Status)"
        }

        Write-Log "Service '$ServiceName' restarted successfully."
    }
    catch {
        Write-Log "ERROR: Service restart failed: $($_.Exception.Message)"
    }
}

# Validate configuration.
if ($CpuThreshold -le 0 -or
    $CpuThreshold -gt 100 -or
    $SustainedSeconds -le 0 -or
    $PollSeconds -le 0 -or
    $CooldownSeconds -le 0 -or
    $LogEveryNSeconds -le 0 -or
    $MaxLogSizeMB -le 0 -or
    $MaxLogFiles -lt 1 -or
    $ServiceTimeout -le 0) {

    throw "Invalid monitor configuration."
}

# Verify that the service exists before monitoring.
try {
    $service = Get-Service -Name $ServiceName -ErrorAction Stop
}
catch {
    Write-Log "ERROR: Service '$ServiceName' was not found."
    exit 1
}

Write-Log "Monitor started."
Write-Log "CPU threshold: $CpuThreshold%"
Write-Log "Sustained duration: ${SustainedSeconds}s"
Write-Log "Polling interval: ${PollSeconds}s"
Write-Log "Restart cooldown: ${CooldownSeconds}s"
Write-Log "Initial service status: $($service.Status)"

# Initialize monitoring timers.
$aboveSince           = $null
$lastRestart          = [DateTime]::MinValue
$lastLoggedAt         = [DateTime]::MinValue
$lastCooldownLoggedAt = [DateTime]::MinValue
$lastSampleAt         = $null
$thresholdLogged      = $false

# Continuous monitoring loop.
while ($true) {

    try {
        $cpu = Get-CpuPercent
    }
    catch {
        Write-Log "ERROR: Failed to read CPU: $($_.Exception.Message)"

        # Failed samples invalidate the sustained CPU streak.
        $aboveSince = $null
        $lastSampleAt = $null
        $thresholdLogged = $false

        Start-Sleep -Seconds $PollSeconds
        continue
    }

    $now = Get-Date

    # Reset the streak after an unexpected monitoring gap.
    if ($lastSampleAt -and
        ($now - $lastSampleAt).TotalSeconds -gt ($PollSeconds * 3)) {

        $aboveSince = $null
        $thresholdLogged = $false

        Write-Log "Monitoring gap detected. CPU streak reset."
    }

    $lastSampleAt = $now

    if ($cpu -ge $CpuThreshold) {

        # Start tracking sustained high CPU usage.
        if (-not $aboveSince) {
            $aboveSince = $now
            $thresholdLogged = $false
            $lastLoggedAt = [DateTime]::MinValue
        }

        $sustained = ($now - $aboveSince).TotalSeconds

        # Log high CPU usage at the configured interval.
        if (($now - $lastLoggedAt).TotalSeconds -ge $LogEveryNSeconds) {

            Write-Log ("CPU {0:N1}% (>= {1}%) for {2:N0}s" -f `
                $cpu, $CpuThreshold, $sustained)

            $lastLoggedAt = $now
        }

        # Check whether high CPU usage has been sustained.
        if ($sustained -ge $SustainedSeconds) {

            # Log the threshold crossing only once.
            if (-not $thresholdLogged) {
                Write-Log "CPU threshold sustained for ${SustainedSeconds}s."
                $thresholdLogged = $true
            }

            $elapsed = ($now - $lastRestart).TotalSeconds

            # Restart only when the cooldown has expired.
            if ($elapsed -ge $CooldownSeconds) {

                Restart-LeapworkController

                # Start cooldown after the restart attempt finishes.
                $lastRestart = Get-Date

                # Reset monitoring state.
                $aboveSince = $null
                $lastSampleAt = $null
                $thresholdLogged = $false
                $lastLoggedAt = [DateTime]::MinValue
                $lastCooldownLoggedAt = [DateTime]::MinValue
            }
            else {
                # Throttle cooldown messages.
                if (($now - $lastCooldownLoggedAt).TotalSeconds `
                    -ge $LogEveryNSeconds) {

                    $remaining = $CooldownSeconds - $elapsed

                    Write-Log ("Restart skipped. Cooldown: {0:N0}s remaining." -f `
                        $remaining)

                    $lastCooldownLoggedAt = $now
                }
            }
        }
    }
    else {
        # Reset the streak when CPU drops below the threshold.
        if ($aboveSince) {
            Write-Log ("CPU dropped to {0:N1}%. Streak reset." -f $cpu)
        }

        $aboveSince = $null
        $thresholdLogged = $false
    }

    Start-Sleep -Seconds $PollSeconds
}
