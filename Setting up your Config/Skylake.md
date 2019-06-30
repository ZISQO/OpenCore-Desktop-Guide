# Starting Point

You'll want to start with either the stock config.plist that OpenCore gives you, or with just a blank canvas. In the next examples, I'll show you how I set things up from scratch; if you start from somewhere else, you may have more things checked/set than I do - but you'll want to follow along with what I do.


# ACPI

![ACPI](https://i.imgur.com/sjlX3aT.png)

**Add:** This is where you'll add SSDT patches for your system, these are most useful for laptops and OEM desktops but also common for [USB maps](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271) and [disabling unsupported GPUs](https://github.com/khronokernel/How-to-disable-your-unsupported-GPU-for-MacOS)

**Block**: 

**Patch**: 

This section allows us to dynamically rename parts of the DSDT via OpenCore. Since we're not running a real mac, and macOS is pretty particular with how things are named, we can make non-destructive changes to keep things mac-friendly. We have three entries here:

* *change XHCI to XHC* - helps avoid a conflict with built-in USB injectors
* *change XHC1 to XHC* - helps avoid a conflict with built-in USB injectors
* *change SAT0 to SATA* - for potential SATA compatibility

**Quirk**: Settings for ACPI.

* FadtEnableReset: Enable reboot and shutdown on legacy hardware, not recommended unless needed
* NormalizeHeaders: Cleanup ACPI header fields, irrelevant in 10.14
* RebaseRegions: Attempt to heuristically relocate ACPI memory regions
* ResetHwSig: Needed for hardware that fail fail to maintain hardware signature across the reboots and cause issues with
waking from hibernation
* ResetLogoStatus: Workaround for systems running BGRT tables, most don't have to worry about this

&#x200B;

# DeviceProperties

![DeviceProperties](https://i.imgur.com/8gujqhJ.png)

**Add**: Sets device properties from a map.

This section is setup via Headkaze's *[Intel Framebuffer Patching Guide](https://www.insanelymac.com/forum/topic/334899-intel-framebuffer-patching-using-whatevergreen/?tab=comments#comment-2626271)* and applies only one actual property to begin, which is the *ig-platform-id*. The way we get the proper value for this is to look at the ig-platform-id we intend to use, then swap the pairs of hex bytes.

If we think of our ig-plat as `0xAABBCCDD`, our swapped version would look like `0xDDCCBBAA`

The two ig-platform-id's we use are as follows:

* 0x19120000 - this is used when the iGPU is used to drive a display
   * 00001219 when hex-swapped
* 0x19120001 - this is used when the iGPU is only used for compute tasks, and doesn't drive a display
   * 01001219 when hex-swapped

We also add 2 more properties, framebuffer-patch-enable and framebuffer-stolenmem. The first enables patching via WhateverGreen.kext, and the second sets the min stolen memory to 19MB.

`PciRoot(0x0)/Pci(0x1b,0x0)` -> `Layout-id`

* Applies AppleALC audio injection, you'll need to do your own research on which codec your motherboard has and match it with AppleALC's layout. [AppleALC Supported Codecs](https://github.com/acidanthera/AppleALC/wiki/Supported-codecs).

Layout=1 would be interprected as 01000000

**Block**: Removes device properties from map, for us we can ignore this



# Kernel

**Add**: Here's where you specify which kexts to load, order matters here so make sure Lilu.kext is always first! Other higher priority kexts come after Lilu such as, VirtualSMC, AppleALC, WhateverGreen, etc.

**Emulate**: Needed for spoofing unsupported CPUs like Pentiums and Celerons

* CpuidMask: When set to Zero, original CPU bit will be used
* CpuidData: The value for the CPU spoofing, don't forget to swap hex

**Block**: Blocks kexts from loading. Sometimes needed for disabling Apple's trackpad driver for some laptops.

**Patch**: Patches kexts (this is where you would add USB port limit patches and AMD CPU patches).

**Quirks**:

* AppleCpuPmCfgLock: NO (Only needed when CFG-Lock can't be disabled in BIOS)
* AppleXcpmCfgLock: NO (Only needed when CFG-Lock can't be disabled in BIOS)
* AppleXcpmExtraMsrs: NO (Disables multiple MSR access needed for unsupported CPUs like Pentiums and certain Xeons)
* CustomSMBIOSGuid: NO (Performs GUID patching for UpdateSMBIOSMode Custom mode. Usually relevant for Dell laptops)
* DisbaleIOMapper: NO (Needed to get around VT-D if unable to disable in BIOS, can interfere with Firmware so avoid when possible)
* ExternalDiskIcons: YES (External Icons Patch, for when internal drives are treated as external drives)
* LapicKernelPanic: NO (Disables kernel panic on AP core lapic interrupt)
* PanicNoKextDump: YES (Allows for reading kernel panics logs when kernel panics occurs)
* ThirdPartyTrim: NO (Enables TRIM, not needed for AHCI or NVMe SSDs)
* XhciPortLimit: YES (This is actually the 15 port limit patch, don't rely on it as it's not a guaranteed solution for fixing USB. Please create a [USB map](https://usb-map.gitbook.io/project/) when possible. Its a temporary solution for those who have yet to create a USB map)

![Kernel](https://i.imgur.com/DcafUhE.png)

# Misc

**Boot**: Settings for boot screen (leave as-is unless you know what you're doing).
* Timeout: 5 (This sets how long OpenCore will wait until it automatically boots from the default selection).
* ShowPicker: YES (
* UsePicker: YES (Uses OpenCore's default GUI, set to NO if you wish to use a different GUI)

**Debug**: Debug has special use cases, leave as-is unless you know what you're doing.
* DisableWatchDog: NO (May need to be set for yes if macOS is stalling on something while booting)

**Security**: Security is pretty self-explanatory.

* RequireSignature: NO (We won't be dealing vault.plist so we can ignore)
* RequireVault: NO (We won't be dealing vault.plist so we can ignore as well)
* ScanPolicy: 0 (This allow you to see all drives available, please refer to OpenCore's DOC for furthur info on setting up ScanPolicy)

**Tools** Used for running OC debugging tools like clearing NVRAM, we'll be ignoring this

![Misc](https://i.imgur.com/6NPXq0A.png)

# NVRAM

**Add**: 7C436110-AB2A-4BBB-A880-FE41995C9F82 (System Integrity Protection bitmask)

* boot-args:
   * -v - this enables verbose mode, which shows all the behind-the-scenes text that scrolls by as you're booting instead of the Apple logo and progress bar.  It's invaluable to any Hackintosher, as it gives you an inside look at the boot process, and can help you identify issues, problem kexts, etc.
   * dart=0 - this is just an extra layer of protection against Vt-d issues.
debug=0x100 - this prevents a reboot on a kernel panic.  That way you can (hopefully) glean some useful info and follow the breadcrumbs to get past the issues.
   * keepsyms=1 - this is a companion setting to debug=0x100 that tells the OS to also print the symbols on a kernel panic.   That can give some more helpful insight as to what's causing the panic itself.
   * shikigva=40 - this flag is specific to the iGPU.  It enables a few Shiki settings that do the following (found [here](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_shiki.hpp#L35-L74)):
      * 8 - AddExecutableWhitelist - ensures that processes in the whitelist are patched.
      * 32 - ReplaceBoardID - replaces board-id used by AppleGVA by a different board-id.

* csr-active-config: <00000000> 

Settings for SIP, recommeded to manully change this within Recovery partition with csrutil

csr-active-config is set to e7030000 which effectively disables SIP. You can choose a number of other options to enable/disable sections of SIP. Some common ones are as follows:

* `00000000` - SIP completely enabled
* `30000000` - Allow unsigned kexts and writing to protected fs locations
* `E7030000` - SIP completely disabled

* nvda_drv:  <> (For enabling Nvidia WebDrivers, set to 31 if running a [Maxwell or Pascal GPU](https://github.com/khronokernel/Catalina-GPU-Buyers-Guide/blob/master/README.md#Unsupported-nVidia-GPUs). This is the same as setting nvda_drv=1 but instead we translate it from [text to hex](https://www.browserling.com/tools/hex-to-text))

* prev-lang:kbd: <> (Needed for non-latin keyboards)

**Block**: Forcibly rewrites NVRAM variables, not needed for us as `sudo nvram` is prefered but useful for those edge cases

**LegacyEnable** Allows for NVRAM to be stored on nvram.plist 

**LegacySchema** Used for assigning nvram variable

![NVRAM](https://i.imgur.com/MPFj3TS.png)

# Platforminfo

![PlatformInfo](https://i.imgur.com/dIKAlhj.png)

For setting up the SMBIOS info, I use acidanthera's *[macserial](https://github.com/acidanthera/macserial)* application. I wrote a *[python script](https://github.com/corpnewt/GenSMBIOS)* that can leverage it as well (and auto-saves to the config.plist when selected). There's plenty of info that's left blank to allow OpenCore to fill in the blanks; this means that updating OpenCore will update the info passed, and not require you to also update your config.plist.

For this Skylake example, I chose the *iMac17,1* SMBIOS.

To get the SMBIOS info generated with macserial, you can run it with the `-a` argument (which generates serials and board serials for all supported platforms). You can also parse it with grep to limit your search to one SMBIOS type.

With our iMac17,1 example, we would run macserial like so via the terminal:

`macserial -a | grep -i iMac17,1`

Which would give us output similar to the following:

      iMac17,1 | C02S8DY7GG7L | C02634902QXGPF7FB
      iMac17,1 | C02T4WZSGG7L | C02703104GUGPF71M
      iMac17,1 | C02QQAYPGG7L | C025474014NGPF7FB
      iMac17,1 | C02SNLZ3GG7L | C02645501CDGPF7AD
      iMac17,1 | C02QQRY8GG7L | C025474054NGPF71F
      iMac17,1 | C02QK1ZXGG7L | C02542200GUGPF7JC
      iMac17,1 | C02SL0YXGG7L | C026436004NGPF7JA
      iMac17,1 | C02QW0J5GG7L | C02552130QXGPF7JA
      iMac17,1 | C02RXDZYGG7L | C02626100GUGPF71H
      iMac17,1 | C02R4MYRGG7L | C02603200GUGPF7JA
      
The order is `Product | Serial | Board Serial (MLB)`

The `iMac17,1` part gets copied to Generic -> SystemProductName.

The `Serial` part gets copied to Generic -> SystemSerialNumber.

The `Board Serial` part gets copied to SMBIOS -> Board Serial Number as well as Generic -> MLB.

We can create an SmUUID by running `uuidgen` in the terminal (or it's auto-generated via my GenSMBIOS script) - and that gets copied to Generic -> SystemUUID.

We set Generic -> ROM to either an Apple ROM (dumped from a real Mac), your NIC MAC address, or any random MAC address (could be just 6 random bytes)

**Automatic**: YES (Generates PlatformInfo based on Generic section instead of DataHub, NVRAM, and SMBIOS sections)

**UpdateDataHub**: YES (Update Data Hub fields)

**UpdateNVRAM**: YES (Update NVRAM fields)

**UpdateSMBIOS**: YES (Update SMBIOS fields)

**UpdateSMBIOSMode**: Create (Replace the tables with newly allocated EfiReservedMemoryType)



# UEFI

![UEFI](https://i.imgur.com/acZ1PUA.png)

**ConnectDrivers**: YES (Forces .efi drivers, change to NO for faster boot times but cerain file system drivers may not load)

**Drivers**: Add your .efi drivers here.

**Protocols**:

* AppleBootPolicy: NO (Ensures APFS compatibility on VMs or legacy Macs)
* ConsoleControl: NO (Replaces Console Control protocol with a builtin version, needed for when firmware doesn’t support text output mode)
* DataHub: NO (Reinstalls Data Hub)
* DeviceProperties: NO (Ensures full compatibility on VMs or legacy Macs)

**Quirks**:

* ExitBootServicesDelay: 0 (Switch to 5 if running ASUS Z87-Pro with FileVault2)
* IgnoreInvalidFlexRatio: NO (Fix for when MSR_FLEX_RATIO (0x194) can't be disabled in the BIOS, required for all pre-skylake based systems)
* IgnoreTextInGraphics: NO (Fix for UI corruption when both text and graphics outputs happen)
* ProvideConsoleGop: YES (Enables GOP, AptioMemoryFix currently offers this but will soon be removed)
* ReleaseUsbOwnership: NO (Releases USB controller from firmware driver)
* RequestBootVarRouting: NO (Redirects AptioMemeoryFix from EFI_GLOBAL_VARIABLE_G to OC_VENDOR_VARIABLE_GUID. Needed for when firmware tries to delete boot entries)
* SanitiseClearScreen: NO (Fixes High resolutions displays that display OpenCore in 1024x768)