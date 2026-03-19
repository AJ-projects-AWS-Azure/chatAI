The GitLab repository has been created and populated with a PowerShell script and pipeline. One step left—I need help ensuring that am1apit002 can access https://stphotprodgwcb5f8.blob.core.windows.net/images/portrait.
stages:
  - sync

sync_photos:
  stage: sync
  tags:
    - local-runner
  script:
    - powershell -ExecutionPolicy Bypass -File scripts/sync.ps1  
'''

########
$ErrorActionPreference = "Stop"

$source = "\\ny139f02\shares\SocialPhotos\Portrait"
$dest   = $env:AZCOPY_STORAGE_URL

$logFile = "C:\logs\azcopy-sync.log"
New-Item -ItemType Directory -Force -Path "C:\logs" | Out-Null

function Write-Log {
    param([string]$message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$timestamp - $message" | Out-File -Append -FilePath $logFile
}

Write-Log "Starting sync..."

$env:AZCOPY_AUTO_LOGIN_TYPE="SPN"
$env:AZCOPY_SPA_APPLICATION_ID=$env:AZCOPY_SPA_APPLICATION_ID
$env:AZCOPY_SPA_CLIENT_SECRET=$env:AZCOPY_SPA_CLIENT_SECRET
$env:AZCOPY_TENANT_ID=$env:AZCOPY_TENANT_ID

azcopy sync $source $dest --recursive --delete-destination=true --log-level=INFO

Write-Log "Sync completed."
-------------

After quite search i thing i will go with easy way which is automate this using a PowerShell script with AzCopy authenticated via a service principal, running through a GitLab pipeline with a local runner that has access to the file share.
