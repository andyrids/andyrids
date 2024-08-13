## Hi there, I'm Andy

I'm a linguist and analyst, who works in a data science & data mining role, which frequently involves the automation of technical analytic tradecraft. My personal projects are often geared towards professional development, but also frontend development and embedded projects with Raspberry Pi.

## My Dev Environment

I am currently experimenting with a custom installation of an Alpine Linux minimal root filesystem on Windows Subsystem for Linux (WSL2).

![Alpine Linux](https://img.shields.io/badge/Alpine_Linux-%230D597F.svg?style=for-the-badge&logo=alpine-linux&logoColor=white) &nbsp;
![Visual Studio Code](https://img.shields.io/badge/Visual%20Studio%20Code-0078d7.svg?style=for-the-badge&logo=visual-studio-code&logoColor=white)


## How I Install Alpine Linux on WSL2

For those interested, I'll detail my setup below:

### 1. Download & install Alpine Linux

The minimal root filesystem can be downloaded from [https://alpinelinux.org/downloads/](https://alpinelinux.org/downloads/). I create a WSL directory and a subdirectory for each distribution (e.g. `C:\WSL\Alpine`), where I keep my distribution images and where I have WSL create the vhdx image files. On administrator PowerShell:

```ps
cd C:\WSL\Alpine
wsl --import Alpine C:\WSL\Alpine .\alpine-minirootfs-3.20.1-x86_64.tar
wsl -l -v
```
### 2. Setup Alpine Linux

I use a profile within Windows Terminal (Microsoft Store), which is set to run Alpine with the following command `C:\windows\system32\wsl.exe -d Alpine`. See here [Alpine Linux post-install guide](https://docs.alpinelinux.org/user-handbook/0.1a/Working/post-install.html) for a useful post-install guide.

```sh
# new user setup - replace <username> with your username
adduser -h /home/username -s /bin/ash username
su -l root

# I use doas rather than sudo
apk update && apk upgrade
apk add doas

# add user to wheel
echo 'permit :wheel' > /etc/doas.d/doas.conf
adduser username wheel

# In Alpine - doas.conf is in /etc/doas.d/
su -l username
doas vi /etc/doas.d/doas.conf
```
As an example for `/etc/doas.d/doas.conf` settings:

```sh
# contents of /etc/doas.d/doas.conf
# permit apk update & apk upgrade without password for user <username>
permit nopass username as root cmd apk args update
permit nopass username as root cmd apk args upgrade
```

```sh
# test doas.conf nopass settings
doas apk update && doas apk upgrade
```

Below details installation & setup of openrc with mdev and hwdrivers on WSL2. As I am not using a full desktop environment, 
mdev is perfectly sufficient. It also allows me to facilitate access to microcontrollers through WSL for embedded projects:


```sh
# mdev is provided by the busybox package
doas apk add busybox-mdev-openrc

# enable the mdev & hwdrivers services
doas rc-update add mdev sysinit
doas rc-update add hwdrivers sysinit

# start services if stopped
doas rc-service mdev --ifstopped start
doas rc-service hwdrivers --ifstopped start

# seed /dev with the device nodes created while the system was booting
doas mdev -s

# view changes
ls -l /dev/

# microcontrollers get added under /dev/ttyACM[0-9] (dialout group)
doas adduser username dialout
```

Below is my init file for mdev at `/etc/init.d/mdev`, which I modified to run mdev as a deamon, which allowed me to
have mdev run as a hotplug manager similar to udev. Whenever I plug in a Raspberry Pi device and share and attach it 
to WSL, I want mdev to respond to a uevent and parse the `/etc/mdev.conf` looking for matching rules for the attached 
device, which is usually ttyACM[0-9]. The modified line is `mdev -d` in the `_start_service` function, which replaced
`mdev -s`.

```sh
#!/sbin/openrc-run

description="the mdev device manager"

depend() {
        provide dev
        need sysfs dev-mount
        before checkfs fsck
        keyword -containers -vserver -lxc
}

_start_service () {
        ebegin "Starting busybox mdev"
        mkdir -p /dev
        echo >/dev/mdev.seq
        echo "/sbin/mdev" > /proc/sys/kernel/hotplug
        eend $?
}

_start_coldplug () {
        ebegin "Scanning hardware for mdev"
        # mdev -s will not create /dev/usb[1-9] devices with recent kernels
        # so we manually trigger events for usb
        for i in $(find /sys/devices -name 'usb[0-9]*'); do
                [ -e $i/uevent ] && echo add > $i/uevent
        done
        # trigger the rest of the coldplug
        # mdev -s is replaced with mdev -d to have it run as a daemon
        mdev -d
        eend $?
}

start() {
        _start_service
        _start_coldplug
}

stop() {
        ebegin "Stopping busybox mdev"
        echo > /proc/sys/kernel/hotplug
        eend
}
```

I configure WSL with the following config file at `/etc/wsl.conf`. I set the default WSL username under the `[user]` 
settings and facilitate openrc starting with WSL, by using the command `command = "/sbin/openrc default"` under `[boot]`. 
The `appendWindowsPath = true` setting under `[interop]`, allows the addition of Windows tools to be added to the WSL distro 
`$PATH` automatically. The command `code .` would for example, be available in WSL and launch VSCode in the current working WSL 
directory.

```sh
# /etc/wsl.conf
[automount]
enabled = true
mountFsTab = true

[network]
generateHosts = true
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = true

[user]
default = username

[boot]
command = "/sbin/openrc default"
```

When we imported this distro of Alpine Linux manually, the `/etc/profile` file overwrites the `$PATH` enviroment variable and therefore 
overwrites the directories added by Windows from the aforementioned `appendWindowsPath = true` setting. I correct this by editing the 
`/etc/profile` file. You could also add anything else to the path here such as `export PATH="$PATH:/home/username/projects/.venv-poetry/bin"`,
which in my case, would add my Poetry package manager to the PATH.


```sh
# replace the existing PATH declaration:
export PATH="$PATH"

export PAGER=less
umask 022

# use nicer PS1 for bash and busybox ash
if [ -n "$BASH_VERSION" -o "$BB_ASH_VERSION" ]; then
        PS1='\h:\w\$ '
# use nicer PS1 for zsh
elif [ -n "$ZSH_VERSION" ]; then
        PS1='%m:%~%# '
# set up fallback default PS1
else
        : "${HOSTNAME:=$(hostname)}"
        PS1='${HOSTNAME%%.*}:$PWD'
        [ "$(id -u)" -eq 0 ] && PS1="${PS1}# " || PS1="${PS1}\$ "
fi

for script in /etc/profile.d/*.sh ; do
        if [ -r "$script" ] ; then
                . "$script"
        fi
done
unset script
```

I reboot and restart Alpine linux to let the new configs take effect:

```sh
# should see mdev & hwdrivers under syinit
rc-update show -v

# should see * status: started for both services
rc-service mdev status && rc-service hwdrivers status
```

### 3. Setup USB Device Sharing to WSL2 (Microcontrollers) 

I use usbipd-win to share locally connected USB devices to other machines, including Hyper-V guests and WSL 2.
By default devices are not shared with USBIP clients. To lookup and share devices, run the following commands with 
administrator privileges:

```ps1
# install usbipd with winget
winget install usbipd

# list devices with busid & state information
usbipd list

# For the Pi Pico - I am looking for something like:
# BUSID  VID:PID    DEVICE                    STATE
# 1-8    2e8a:0005  USB Serial Device (COM5)  shared

# bind the busid
usbipd bind --busid 1-8

# attach the busid to WSL2
usbipd attach --wsl --busid 1-8

# should see state 'shared'
usbipd list
```

In Alpine Linux you can test the device sharing using the following commands:

```sh
lsusb

# note the ID 2e8a:0005 matches the VID:PID on PowerShell
Bus 001 Device 001: ID 1d6b:0002
Bus 001 Device 002: ID 2e8a:0005
Bus 002 Device 001: ID 1d6b:0003

# should show something like ttyACM0 for this device
ls -l /dev/ | grep ttyACM

crw-rw----    1 root     dialout   166,   0 Jul 30 20:05 ttyACM0
```

I am currently working on a PowerShell script, which I am using to monitor Windows events related to Raspberry Pi device connections,
and automatically share and attach each device to WSL, if active. It's verymuch a WIP and I had to learn PowerShell specifically for this
task, but the code is below for those intersted:

```ps1
<#
For a full list of Raspberry Pi vendor ID (VID) & product ID (PID) values,
see https://github.com/raspberrypi/usb-pid.

Raspberry Pi VID: 0x2E8A

Raspberry Pi PID:

┌────────┬──────────────────────────────────────────────┐
│   PID  │                    Product                   │
├────────┼──────────────────────────────────────────────┤
│ 0x0003 │ Raspberry Pi Pico W                          │
├────────┼──────────────────────────────────────────────┤
│ 0x0004 │ Raspberry Pi PicoProbe                       │
├────────┼──────────────────────────────────────────────┤
│ 0x0005 │ Raspberry Pi Pico MicroPython firmware (CDC) │
├────────┼──────────────────────────────────────────────┤
│ 0x000A │ Raspberry Pi Pico SDK CDC UART               │
├────────┼──────────────────────────────────────────────┤
│ 0x000B │ Raspberry Pi Pico CircuitPython firmware     │
├────────┼──────────────────────────────────────────────┤
│ 0x1000 │ Cytron Maker Pi RP2040                       │
└────────┴──────────────────────────────────────────────┘

Prop_DeviceName Pico 112
2e8a:000a  USB Serial Device (COM6), Reset
2e8a:0003  USB Mass Storage Device, RP2 Boot
#>

# requires -RunAsAdministrator
#requires -version 5.1

using namespace System.Diagnostics.Eventing
using namespace System.Management.Automation

# import usbipd-win PowerShell module
Import-Module $env:ProgramW6432'\usbipd-win\PowerShell\Usbipd.Powershell.dll'

# Used to represent all WSL distribution status values
enum WSLStatus {
    Stopped
    Running
    Installing
    Uninstalling
    Converting
}

<#
Used as a switch value relating to actions for binding and attaching COM
devices' data bus to active WSL distributions. Enum values are based on
the addition of IsBound & IsAttached Boolean properties of the objects
returned by Get-UsbipdDevice (usbipd-win).

A device can be:

1. Not shared or attached
2. Shared & not attached
3. Shared & attached
#>
enum COMStatus {
    NotShared
    Shared
    Attached
}


# Pico PID enumerations
enum PicoPID {
    RP2040Boot = 0x0003
    MicroPython = 0x0005
    PicoSDK = 0x000A
    CircuitPython = 0x000B
}


function global:Get-WSLDistributionInformation {
    <#
    .SYNOPSIS
        Get all WSL distribution details.

    .DESCRIPTION
        Parses the string output from wsl --list --verbose command
        into a hashtable with the following structure:

        [string]Name: distribution name
        [bool]Default: default distribution flag
        [string]Status: distribution status
        [int]Version: WSL version

    .INPUTS
        None

    .OUTPUTS
        System.hashtable
    #>
    [CmdletBinding()]
    [OutputType([hashtable])]
    param ()

    process {
        # Available WSL distributions & status
        $WSLOutput = wsl --list --verbose |
            Select-Object -Skip 1 -Unique |
            Where-Object -Property Length -gt 1

        $WSLOutput | ForEach-Object {
            <#
            An '*' before a distribution name would denote that
            particular WSL distribution as the WSL default.

            $WSLOutput:
                * Alpine    Stopped      2

            [Regex]::new("\b[^\w]{2,}").Replace($WSLOutput, "-"):
                * Alpine-Stopped-2

            ConvertFrom-String -Delimiter "-":
                P1: "* Alpine", P2: "Stopped", P3: "2"

            Select-Object:
                Name: Alpine, Default: True, Status: Stopped, Version: 2
            #>

            # Calculated properties for each WSL distribution object
            $Name = @{label="Name"; expression={$_.P1 -replace "[\*\s]", ""}}
            $Default = @{label="Default"; expression={$_.P1.StartsWith("*")}}
            $Status = @{label="Status"; expression={$_.P2}}
            $Version = @{label="Version"; expression={$_.P3}}

            # WSL distribution objects converted from distribution details string
            [Regex]::new("\b[^\w]{2,}").Replace($_.Trim(), "-") |
                ConvertFrom-String -Delimiter "-" |
                Select-Object -Property $Name, $Default, $Status, $Version
        }
    }
}


function global:Approve-COMDeviceMountToWSL {
    <#
    .SYNOPSIS
        Share connected device to allow WSL attachment.

    .DESCRIPTION
        Invoke the command usbipd bind --busid $BusId to share
        a connected device, allowing it to be attached to WSL.

    .INPUTS
        None

    .OUTPUTS
        System.Void

    .NOTES
    #>
    [CmdletBinding()]
    [OutputType([Void])]
    param (
        [Parameter(Mandatory)]
        [ValidatePattern("^[1-9]-([1-9]$|1[0-5])")]
        [string]$BusId
    )

    process {
        usbipd bind --busid $BusId
    }
}


function global:Mount-COMDeviceToWSL {
    <#
    .SYNOPSIS
        Attach connected device to WSL distributions.

    .DESCRIPTION
        Invoke the command usbipd --wsl --busid $busId to attach
        a connected device (if shared) to all WSL distributions.

    .INPUTS
        None

    .OUTPUTS
        System.Void

    .NOTES
        A connected device must be shared before it can be attached
        to WSL. The Approve-COMDeviceMountToWSL function can be used
        to share the device if administrator rights are active.
    #>
    [CmdletBinding()]
    [OutputType([Void])]
    param (
        [Parameter(Mandatory)]
        [ValidatePattern("^[1-9]-([1-9]$|1[0-5])")]
        [string]$BusId
    )

    process {
        $sharedDevice = Get-usbipdDevice | Where-Object { $_.BusId -eq $BusId -and $_.IsBound }
        if ($sharedDevice) {
            usbipd attach --wsl --busid $busId | Out-Null
        }

        Write-Error "Device on bus $BusId must be shared before attaching to WSL"
    }

}


function global:Connect-COMDeviceToWSL {
    <#
    .SYNOPSIS
        Facilitate device connection to active WSL distributions.

    .DESCRIPTION
        Invokes usbipd-win commands for binding and attaching a
        COM device's data bus to active WSL distributions based
        on the connected device status:

        1. Not shared (or attached)
        2. Shared (but not attached)
        3. Shared & attached

        NOTE: If there are active WSL distributions, then
        a device will be shared & attached.

    .INPUTS
        None.

    .OUTPUTS
        hashtable.
    #>
    [CmdletBinding()]
    [OutputType([System.Void])]
    param (
        [Parameter(Mandatory = $false)]
        [PicoPID[]]$PIDValues = @("RP2040Boot", "MicroPython", "PicoSDK", "CircuitPython")
    )

    process {
        $PIDRegex = ($PIDValues | ForEach-Object { "{0:x4}" -f[int]$_ }) -Join "|"
        $connectedDevices = Get-UsbipdDevice |
            Where-Object { $_.IsConnected -and $_.HardwareId -match "2e8a:($PIDRegex)" }

        $informationParams = @{
            MessageData = "Connected devices: $($connectedDevices.Count)`n"
            InformationAction = [ActionPreference]::Continue
        }
        Write-Information @informationParams

        if (-not $connectedDevices) {
            return $null
        }

        $isActiveWSL = (Get-WSLDistributionInformation |
            Where-Object { $_.Status -eq [WSLStatus]::Running }).Count -gt 0

        $connectedDevices | ForEach-Object {
            $PIDName = [PicoPID].GetEnumName([int]$_.HardwareId.Pid)
            $device = "$($_.Description) [$PIDName] on bus $($_.BusId)"
            $busId = $_.BusId

            $deviceStatus = [COMStatus].GetEnumName($_.IsBound + $_.IsAttached)
            switch ($deviceStatus) {
                NotShared {
                    $informationParams.MessageData = "[Unshared device] binding $device"
                    Write-Information @informationParams

                    Approve-COMDeviceMountToWSL -Busid $busId
                    if ($isActiveWSL) {
                        $informationParams.MessageData = "[WSL active] attaching $device"
                        Write-Information @informationParams
                        Mount-COMDeviceToWSL -Busid $busId
                        continue
                    }
                    $informationParams.MessageData = "[WSL inactive] not attaching $device"
                    Write-Information @informationParams
                    continue
                }
                Shared {
                    $informationParams.MessageData = "[Shared device] not binding $device"
                    Write-Information @informationParams

                    if ($isActiveWSL) {
                        $informationParams.MessageData = "[WSL active] attaching $device"
                        Write-Information @informationParams
                        Mount-COMDeviceToWSL -Busid $busId
                        continue
                    }
                    $informationParams.MessageData = "[WSL inactive] not attaching $device"
                    Write-Information @informationParams
                    continue
                }
                Attached {
                    $informationParams.MessageData = "[WSL active] - $device already attached"
                    Write-Information @informationParams
                    continue
                }
                Default {
                    $informationParams.MessageData = "COM device status ($deviceStatus) not recognised"
                    Write-Information @informationParams
                    continue
                }
            }
        }
        Out-Null
    }
}

# # Get WSL distribution details
# $DistributionDetails = Get-WSLHashtable

# Write-Output "Fetching ($($DistributionDetails.Count)) installed WSL distribution details"

# $DistributionDetails | Format-List | Write-Output

# # Bind the data bus for any Raspberry Pi devices & attach to WSL (if active)
# Connect-COMDevice

<#
Start a WSL distribution
-----------------------------------
Log Name: System
Source: Hyper-V-VmSwitch
Event ID: 232
-----------------------------------
NIC B944A522-3D6A-4950-A844-16DC623D891A--6B4A85CC-55A1-4C9F-8C85-44C2A33AAE4A (Friendly Name: )
successfully connected to port 6B4A85CC-55A1-4C9F-8C85-44C2A33AAE4A (Friendly Name: 6B4A85CC-55A1-4C9F-8C85-44C2A33AAE4A)
on switch 790E58B4-7939-4434-9358-89AE7DDBE87E(Friendly Name: WSL (Hyper-V firewall)).
-----------------------------------
EventData
  NicNameLen 74
  NicName B944A522-3D6A-4950-A844-16DC623D891A--6B4A85CC-55A1-4C9F-8C85-44C2A33AAE4A
  NicFNameLen 0
  NicFName
  PortNameLen 36
  PortName 6B4A85CC-55A1-4C9F-8C85-44C2A33AAE4A
  PortFNameLen 36
  PortFName 6B4A85CC-55A1-4C9F-8C85-44C2A33AAE4A
  SwitchNameLen 36
  SwitchName 790E58B4-7939-4434-9358-89AE7DDBE87E
  SwitchFNameLen 22
  SwitchFName WSL (Hyper-V firewall)
#>

# Query options & filter
$SystemLog = "System"
$HyperVFilter = "*[System[(EventID=232)] and EventData[(Data='WSL (Hyper-V firewall)')]]"
$HyperVQuery = [Reader.EventLogQuery]::new($SystemLog, [Reader.PathType]::LogName, $HyperVFilter)

# Overload used: EventLogWatcher(EventLogQuery, EventBookmark, Boolean)
# Boolean determines inclusion of pre-existing events that match the EventLogQuery
$HyperVWatcher = [Reader.EventLogWatcher]::new($HyperVQuery, $null, $false)
$HyperVWatcher.Enabled = $true

<#
Attach Raspberry Pi Pico in FS Mode
-----------------------------------
Log Name: Microsoft-Windows-DeviceSetupManager/Admin
Log path: %SystemRoot%\System32\Winevt\Logs\Microsoft-Windows-DeviceSetupManager%4Admin.evtx
Event ID: 112
-----------------------------------
Device 'Board in FS mode' ({8d7d90ed-ac33-50a0-b7d8-4b3d1f750bd7}) has been serviced, processed 4 tasks,
wrote 0 properties, active worktime was 145 milliseconds.
-----------------------------------
EventData
  Prop_DeviceName 'Board in FS mode', 'Pico', 'RP2 Boot'
  Prop_ContainerId {8d7d90ed-ac33-50a0-b7d8-4b3d1f750bd7}
  Prop_TaskCount 4
  Prop_PropertyCount 0
  Prop_WorkTime_MilliSeconds 145
#>

# Query options & filter
$DeviceSetupManagerLog = "Microsoft-Windows-DeviceSetupManager/Admin"
$FSModeFilter = "*[System[(EventID=112)] and EventData[Data='Board in FS mode' or Data='Pico' or Data='RP2 Boot']]"
$DeviceQuery = [Reader.EventLogQuery]::new($DeviceSetupManagerLog, [Reader.PathType]::LogName, $FSModeFilter)

# Overload used: EventLogWatcher(EventLogQuery, EventBookmark, Boolean)
# Boolean determines inclusion of pre-existing events that match the EventLogQuery
$DeviceWatcher = [Reader.EventLogWatcher]::new($DeviceQuery, $null, $false)
$DeviceWatcher.Enabled = $true


$Action = {
    <#
    NOTE: The commands in the Action run when an event is raised, instead of sending the
    event to the event queue, making the 'Wait-Event' (Ln 285) act as an indefinite wait.

    This script has access to automatic variables;

    $Event, $EventSubscriber, $Sender, $EventArgs & $Args.
    #>

    # $EventArgs.EventRecord.Properties | Get-Member -MemberType Property | Out-Host
    # $EventArgs.EventRecord.Properties | Select-Object | Out-Host

    $sourceIdentifier = $Event.SourceIdentifier
    $identifierMap = @{
        DeviceConnection = $EventArgs.EventRecord.Properties | Select-Object -First 1 -Property Value -ExpandProperty Value
        HyperVConnection = $EventArgs.EventRecord.Properties | Select-Object -Last 1 -Property Value --ExpandProperty Value
    }

    $informationParams = @{
        MessageData = $"[Event detected - ($identifierMap[$SourceIdentifier])]"
        Tags = $sourceIdentifier
        InformationAction = [ActionPreference]::Continue
    }
    Write-Information @informationParams

    Connect-COMDeviceToWSL
}

$HyperVEventParams = @{
    InputObject = $HyperVWatcher
    EventName = "EventRecordWritten"
    SourceIdentifier = "HyperVConnection"
    Action =  $Action
}

$DeviceEventParams = @{
    InputObject = $DeviceWatcher
    EventName = "EventRecordWritten"
    SourceIdentifier = "DeviceConnection"
    Action =  $Action
}

$JobDevice = Register-ObjectEvent @DeviceEventParams
$JobHyperV = Register-ObjectEvent @HyperVEventParams

try {
    $informationParams = @{
        MessageData = "`nWatching for device connections`n"
        InformationAction = [ActionPreference]::Continue
    }
    Write-Information @informationParams

    <#
    Indefinite wait for 'DeviceConnection' & 'HyperVConnection' events, as they are handled by
    the $Action script. Wait-Event call prevents the script from exiting, without blocking.
    NOTE: Use Ctrl-C to exit the script
    #>
    Wait-Event
}
finally {
    Write-Warning "Script terminated - WSL & RPi device connection monitoring stopped."

    # Unregister events for each EventSubscriber
    Get-EventSubscriber -SourceIdentifier "DeviceConnection" | Unregister-Event
    Get-EventSubscriber -SourceIdentifier "HyperVConnection" | Unregister-Event

    # Delete the background JobDevice & JobHyperV background jobs
    $JobDevice | Remove-Job -Force
    $JobHyperV | Remove-Job -Force
}
```

### Analysis & Research

![Jupyter Notebook](https://img.shields.io/badge/jupyter-%23FA0F00.svg?style=for-the-badge&logo=jupyter&logoColor=white) &nbsp;
![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) &nbsp;
![NumPy](https://img.shields.io/badge/numpy-%23013243.svg?style=for-the-badge&logo=numpy&logoColor=white) &nbsp;
![Pandas](https://img.shields.io/badge/pandas-%23150458.svg?style=for-the-badge&logo=pandas&logoColor=white) &nbsp;
![scikit-learn](https://img.shields.io/badge/scikit--learn-%23F7931E.svg?style=for-the-badge&logo=scikit-learn&logoColor=white) &nbsp;
![SciPy](https://img.shields.io/badge/SciPy-%230C55A5.svg?style=for-the-badge&logo=scipy&logoColor=%white) &nbsp;
![Matplotlib](https://img.shields.io/badge/Matplotlib-%23ffffff.svg?style=for-the-badge&logo=Matplotlib&logoColor=black) &nbsp;




### Embedded Projects

![Raspberry Pi](https://img.shields.io/badge/-RaspberryPi-C51A4A?style=for-the-badge&logo=Raspberry-Pi) &nbsp;
![CMake](https://img.shields.io/badge/CMake-%23008FBA.svg?style=for-the-badge&logo=cmake&logoColor=white) &nbsp;
![C](https://img.shields.io/badge/c-%2300599C.svg?style=for-the-badge&logo=c&logoColor=white) &nbsp;
![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) &nbsp;



