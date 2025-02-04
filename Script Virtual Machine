param (
    [Parameter()]
    [hashtable]
    $OperatingSystems = @{
        Win10      = 'https://www.microsoft.com/en-us/evalcenter/download-windows-10-enterprise' #Adapt for ltsc
        Win11      = 'https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise'
        Server2016 = 'https://www.microsoft.com/en-us/evalcenter/download-windows-server-2016'
        Server2019 = 'https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019'
        Server2022 = 'https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022'
        Server2025 = 'https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025'
    },

    [Parameter()]
    [string]
    $BasePath = 'C:\Virtual Machines',

    [Parameter()]
    [array]
    $Paths = @('', 'Iso', 'Vhd')
)

Clear-host
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Install-Module 'windowsimagetools'
$paths.ForEach({
        if ( -not (Test-Path ($path = Join-Path $BasePath $_))) {
            New-Item -Path $Path -ItemType Directory 
        }
    })

function get-Iso {
    [cmdletbinding()]
    param(
        [String]$IsoPath = (Join-Path $BasePath $Paths[1]),
        [String]$OS = 'Server2022'
    )
    try {
        $url = $OperatingSystems[$OS]
        $content = Invoke-WebRequest -Uri $url -UseBasicParsing -ErrorAction Stop
        $link = $content.links | Where-Object { $_.'aria-label' -match 'Iso' -and $_.'aria-label' -match 'en-us' -and $_.'aria-label' -match '64-bit' } # Replace with regex
        $header = Invoke-WebRequest $link[0].href -UseBasicParsing -Method Head 
        $output = @{
            Uri      = $link[0].href
            Filename = ([System.Net.Mime.ContentDisposition]::new($header.headers.'content-disposition').filename)
            Length   = [int64]$($header.headers.'Content-Length')
            Path     = [string]
        }
        $count = ($output.Uri).Count
        Write-host ("Processing {0}, Found {1} Download" -f $url, $count + "`n") 
    }
    catch {
        Write-Warning ("{0} is not accessible" -f $url)
    }
    $output['Path'] = (Join-Path -Path $IsoPath -ChildPath $output.filename)
    if (-not ((Test-Path $output.Path) -and (get-item $output.Path).Length -eq $output.Length)) {
        curl.exe -JLo $output.path $output.uri

    }
    return $output
}
function new-VMFast {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [string]
        $VMName,

        [Parameter()]
        [string]
        $Memory = 4GB,

        [Parameter()]
        [ValidateRange(1, 2)]
        [int]$Generation = 2,

        [Parameter()]
        [string]
        $Switch = 'default switch'     

    )
    if (-not (Get-VM -Name $VMName -ErrorAction 'SilentlyContinue')) {
        New-VM -Name $VMName -MemoryStartupBytes $Memory -Generation $Generation -SwitchName $Switch -Path $basepath 
        Set-VM -Name $VMName -AutomaticStartAction "Nothing" -AutomaticStopAction "TurnOff" -CheckpointType "Disabled"  -ProcessorCount 4
        ADD-VMHardDiskDrive -VMName $VMName -ControllerNumber 0 -ControllerLocation 0 
    }
    return Get-VM -Name $VMName
}

function set-VMHD {
    param (
        [Parameter(Mandatory, ValueFromPipeline)]
        [System.Object]
        $VM,

        [Parameter()]
        [string]
        $OS = 'Server2022'
    )  

    if (get-vm -Name $VM.name -ErrorAction 'SilentlyContinue') {
        $VHD = Join-Path (Join-Path $BasePath $Paths[2]) "$OS.vhdx"
        if (-not (Test-Path $VHD )) {
            $iso = get-Iso -OS $OS
            $unattend = New-UnattendXml -AdminCredential 'Administrator' -TimeZone 'W. Europe Standard Time' -InputLocale fr-BE -UserLocale fr-BE -enableAdministrator 
            Convert-Wim2VHD -SourcePath $iso.Path -Path $VHD -Size 40GB -Dynamic -Index 2 -DiskLayout UEFI -Unattend $unattend #index=1 for win10 11 admin 
        }
        new-item (Join-Path $vm.Path 'Virtual Hard Disks' ) -ItemType Directory -ErrorAction SilentlyContinue
        $VMHardDiskDrive = (Join-Path -Path (Join-Path $vm.Path 'Virtual Hard Disks') -ChildPath ($VM.name + '.vhdx') )
        copy-item -Path $VHD -Destination  $VMHardDiskDrive
        Set-VMHardDiskDrive -VMName $VM.Name -Path $VMHardDiskDrive  -ControllerNumber 0 -ControllerLocation 0 -ControllerType SCSI
        Set-VMFirmware -VMName $VM.name -FirstBootDevice (Get-VMHardDiskDrive -VMName $VM.name)
    }
}
