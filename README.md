param (
    [string]$SourcePath = "\\ny139f02\shares\SocialPhotos\Portrait",
    [string]$DestinationURL = "https://stphotprodgwcb5f8.blob.core.windows.net/images/portrait",
    [string]$AzCopyPath = "C:\Tools\AzCopy\azcopy.exe"
)

Write-Host "===== Photo Sync Job Started ====="

# 🔹 Step 1 — Validate AzCopy exists
if (-not (Test-Path $AzCopyPath)) {
    Write-Error "AzCopy not found at $AzCopyPath"
    exit 1
}

# 🔹 Step 2 — Validate source path access
if (-not (Test-Path $SourcePath)) {
    Write-Error "Cannot access source path: $SourcePath"
    exit 1
}

Write-Host "Source path is accessible."

# 🔹 Step 3 — Check Azure connectivity (proxy detection)
Write-Host "Checking Azure connectivity..."

$connection = Test-NetConnection "stphotprodgwcb5f8.blob.core.windows.net" -Port 443

if (-not $connection.TcpTestSucceeded) {
    Write-Warning "Direct connection failed. Applying proxy..."

    # ⚠️ TEMPORARY proxy (safe for shared runner)
    $env:HTTPS_PROXY = "http://proxy:port"

    Write-Host "Proxy set for this session only."
} else {
    Write-Host "Direct connection successful. No proxy needed."
}

# 🔹 Step 4 — Set Azure authentication (Service Principal)
$env:AZCOPY_AUTO_LOGIN_TYPE = "SPN"

if (-not $env:AZCOPY_SPA_CLIENT_ID -or `
    -not $env:AZCOPY_SPA_CLIENT_SECRET -or `
    -not $env:AZCOPY_TENANT_ID) {

    Write-Error "Missing Azure authentication environment variables"
    exit 1
}

Write-Host "Azure authentication variables detected."

# 🔹 Step 5 — Dry run (safe validation)
Write-Host "Running AzCopy dry-run..."

& $AzCopyPath sync $SourcePath $DestinationURL --recursive --dry-run

if ($LASTEXITCODE -ne 0) {
    Write-Error "Dry-run failed"
    exit 1
}

# 🔹 Step 6 — Actual sync
Write-Host "Running actual sync..."

& $AzCopyPath sync $SourcePath $DestinationURL --recursive

if ($LASTEXITCODE -ne 0) {
    Write-Error "Sync failed"
    exit 1
}

Write-Host "✅ Photo sync completed successfully!"
exit 0
