# ACPI WMI Adventure

## Why?
As Linux user, you may have found that many cool/useful switch to select feature of the hardware are not available in Linux.
Notable mention on Legion is the Hybrid/dGPU toggle, without going in BIOS.

## What we can do
Thankfully, most of the time these features are exported trough WMI (Windows Management Instrumentation), even with this fancy name, these are simple ACPI method, wrapped in a device that Windows can enumerate, but being ACPI method can also be called directly for example with acpi_call

## How to do it
The proper way to do this would be creating a kernel module, register to the WMI and export a sysfs interface, but we are lazy so we will call the method directly with acpi_call

## Step 0
Identify if a WMI method, if exist.
To do this boot a Windows install, open an admin PowerShell.

Now we can list all possible WMI Method

`Get-WmiObject -Namespace 'ROOT/WMI' -List  | select Name`

The result will be huge, but there we can find some interesting one, we can also notice that all the Lenovo one, have a LENOVO in the name, so we can filter only for them 

`Get-WmiObject -Namespace 'ROOT/WMI' -List -Class *LENOVO* | select Name`

This on a 2020 AMD will result in 

    LENOVO_GAMEZONE_SMART_FAN_MODE_EVENT
    LENOVO_UTILITY_EVENT
    LENOVO_GAMEZONE_SMART_FAN_SETTING_EVENT
    LENOVO_GAMEZONE_KEYLOCK_STATUS_EVENT
    LENOVO_GAMEZONE_POWER_CHARGE_MODE_EVENT_EVENT
    LENOVO_GAMEZONE_TEMP_EVENT
    LENOVO_GAMEZONE_THERMAL_MODE_EVENT
    LENOVO_SUPERKEY_EVENT
    LENOVO_GAMEZONE_LIGHT_PROFILE_CHANGE_EVENT
    LENOVO_GAMEZONE_OC_EVENT
    LENOVO_GAMEZONE_FAN_COOLING_EVENT
    LENOVO_GAMEZONE_GPU_TEMP_EVENT
    LENOVO_UTILITY_DATA
    LENOVO_GAMEZONE_DATA
    LENOVO_SUPERKEY_DATA
    Lenovo_SystemElement
    Lenovo_BatteryInformation
    LENOVO_GAMEZONE_CPU_OC_DATA
    LENOVO_GAMEZONE_GPU_OC_DATA

We can notice two major type of class, **DATA** classes and **EVENT** Classes.

The EVENT classes, are trigger event that allow the Windows to be notified when something happen, notable mention are the Fn+Q and the Caps-Lock overlay (Maybe one day will do also a guide on these)

What we are interested in are instead the DATA one:

    LENOVO_UTILITY_DATA
    LENOVO_GAMEZONE_DATA
    LENOVO_SUPERKEY_DATA
    LENOVO_GAMEZONE_CPU_OC_DATA
    LENOVO_GAMEZONE_GPU_OC_DATA

Let's ignore the OC one, that don't contain anything useful, so we are left with LENOVO_UTILITY_DATA and LENOVO_GAMEZONE_DATA.

Let's query what Method these class contain:

There are various method to do this, one of these is

`([wmiclass]'\\.\ROOT\WMI:LENOVO_UTILITY_DATA').Methods | select Name`

This will return a boring

    GetIfSupportOrVersion

Let's try on the LENOVO_GAMEZONE_DATA
`([wmiclass]'\\.\ROOT\WMI:LENOVO_GAMEZONE_DATA').Methods | select Name`

    GetIRTemp
    GetThermalTableID
    SetThermalTableID
    IsSupportGpuOC
    GetGpuGpsState
    SetGpuGpsState
    GetFanCount
    GetFan1Speed
    GetFan2Speed
    GetFanMaxSpeed
    GetVersion
    IsSupportFanCooling
    SetFanCooling
    IsSupportCpuOC
    IsBIOSSupportOC
    SetBIOSOC
    GetTriggerTemperatureValue
    GetCPUTemp
    GetGPUTemp
    GetFanCoolingStatus
    IsSupportDisableWinKey
    SetWinKeyStatus
    GetWinKeyStatus
    IsSupportDisableTP
    SetTPStatus
    GetTPStatus
    GetGPUPow
    GetGPUOCPow
    GetGPUOCType
    GetKeyboardfeaturelist
    GetMemoryOCInfo
    IsSupportWaterCooling
    SetWaterCoolingStatus
    GetWaterCoolingStatus
    IsSupportLightingFeature
    SetKeyboardLight
    GetKeyboardLight
    GetMacrokeyScancode
    GetMacrokeyCount
    IsSupportGSync
    GetGSyncStatus
    SetGSyncStatus
    IsSupportSmartFan
    SetSmartFanMode
    GetSmartFanMode
    GetSmartFanSetting
    GetPowerChargeMode
    GetProductInfo
    IsSupportOD
    GetODStatus
    SetODStatus
    SetLightControlOwner
    SetDDSControlOwner
    IsRestoreOCValue
    GetThermalMode

Now we are talking

Some interesting one are:

    GetFan1Speed
    GetFan2Speed
    ...
    GetGSyncStatus
    SetGSyncStatus

This SetGSyncStatus, is the Hybrid/dGPU toggle, even if the name is not that explicit, somehow make sense..

## Step 1
On Linux (Can be also done on Windows, using acpidump for getting the tables)

install `iasl` from your distro package manager

Create a Work folder and cd into it

`mkdir acpi_wmi && cd  acpi_wmi`

and copy the useful acpi table (SSDT and DSDT) onto it

`sudo cp --no-preserve=mode /sys/firmware/acpi/tables/*SDT* .`

now we can decompile the DSDT using the SSDT for external symbol resolution

`iasl -e SSDT* -d DSDT`

Now we will have a nice decompiled DSDT file DSDT.dsl

Open it

Search for WMAA *(TODO Explain Why WMAA, and what are the possible alternative)*
No we can see a bunch of `If ((Arg1 == xx))` this is a "switch-case" for all the possible WMI method..

Thankfully, the order exported from PowerShell, is on ACPI order, so If we take SetGsysnc Status, wee see that is the 42 entry, so in Hex the Ox2A

And if we cross-check there's a mach for   `If ((Arg1 == 0x2A))`
So this is our target

Now get the full path of WMAA, traversing the hierarchy, 
it turn to be `\_SB.GZFD.WMAA`

so the ACPI call will be `\_SB.GZFD.WMAA 0 0x2A 0` to disable dGPU and `\_SB.GZFD.WMAA 0 0x2A 1` to enable it

where  `\_SB.GZFD.WMAA` is the method path, the first zero is the arg0 of the call, looking at the ACPI is ignored, the second one is the method to call, the 3 one is the method argument

Just for fun let's try another one, this time a Get one, the perfect target is GetFan1Speed, and is the 8 Entry so in hexadecimal 0x08

If we check the method on ACPI, we see that if contain referee to FANS, so is the right one...

The acpi_call this time will be `\_SB.GZFD.WMAA 0 0x08`

let's try it

`echo "\_SB.GZFD.WMAA 0 0x08" | sudo tee /proc/acpi/call`
`sudo cat /proc/acpi/call`

the response will by a hex number that converted to decima will be the Fan Speed

# Bonus one

`\_SB_.GZFD.WMAA 0 0x0D 1` enable turbo fan, while `\_SB_.GZFD.WMAA 0 0x0D 0` disable it








# Still Some missing Feature
Some feature like Conservation mode and Fast Charging/ Overdrive are not in that list, for this we need to go to the EC territory
but that's a story for another day
[Get other Feature On Linux/The EC Rabbit Hole](EC_RABBIT_HOLE.md)