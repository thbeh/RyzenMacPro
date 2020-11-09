# Ryzen Mac Pro - OpenCore EFI for ASRock X570 ITX



This repository provides the basic EFI folder to run macOS Catalina on an ASRock Phantom Gaming ITX/TB3 motherboard. The default provided currently using a Ryzen 9 3900X 12 Core CPU and a Radeon RX 5500 XT. For a short guide to using different CPUs and GPUs see below (all kexts specific to those are named explicitely).
This is intended as a reference and to share improvements for similar build, not as an out of the box EFI to download. It is highly recommended to start with a vanilla OpenCore and following OpenCore Vanilla Guide first.

## Ryzen Mac Pro build

**Prozessor:** AMD Ryzen 9 3900X  
**Cooler:** Custom Water Cooling  
**Motherboard:** AsRock X570 Phantom Gaming ITX/TB3 (BIOS 2.60)  
**WIFI/BT**: Broadcom BCM94360NG  
**Memory:** Kingston HyperX Predator (2x 32GB) DDR4-3600  
**Storage:** Corsair MP600 (1000GB) M.2 NVMe PCIe 4.0  
**Video Card:** XFX Radeon VII 16GB  
**Power Supply:** Corsair SF600 Platinum  
**Case:** Phanteks Enthoo Evolv Shift (Mini-ITX)  

### Notes
I've heavily modded the case to fit the Radeon VII with 3 120mm radiators in it.
Also I've replaced the integrated Intel AX200 module with a BCM94360NG that is natively supported by macOS. 

## Versions
**BIOS:** 2.00 / 2.30  / 2.60 / 2.70 / 2.71
**OpenCore:** 0.6.1  
**macOS:** 10.15 / 11.0 Beta  

## Content

### ACPI

The following SSDT files are for setting up HPET and EC.

- SSDT-EC.aml
- SSDT-HPET.aml
- SSDT-PLUG.aml
- SSDT-SBRG.aml

The following SSDT files are for USB Power and properly name USB controllers and ports. Note: to fix sleep issues the internal HS10 port connected to bluetooth is **not** configured as internal.

- SSDT-USBX.aml
- SSDT-XHC.aml

The following SSDT files fixing SMBUS support. Choose on depending on the BIOS version either 2.00 or 2.30+. Note: Using this SSDT prevents the Machine from entering sleep.

- SSDT-SBUS-MCHC_2.00.aml
- SSDT-SBUS-MCHC_2.30.aml

### Kexts

Besides the default kexts the following are noteworthy:

For enabling the integrated Intel Bluetooth/Wifi you can use the kexts from OpenIntelWireless. Though Bluetooth is working mostly perfect, some things like audio input (Bluetooth Mic from AirPods for example) do not work.

The SMCAMDProcessor.kext is used to provide CPU temperature and frequency information, the AMD Power Gadget can be downloaded from https://github.com/trulyspinach/SMCAMDProcessor/releases. Other monitoring tools can also access and display this information.
AMDRyzenCPUPowerManagement-ES.kext is a custom compiled version with a few minor differences: Upon boot low power state (P2) is enabled and Core Performance Boost is desabled by default. Additionally the adjustment has been tuned down a little to not immediately go to the highest power state and go down again faster. This with the intent to not have the CPU going full power for just minor loads like opening an empty browser tab. Only use this kext if you know what you are doing as it requires P2 to work correctly on your CPU. 

The VoodooTSCSyncAMD kext is used to sync the cores and required the correct number of threads (cores * 2). Either update the Info.plist of the kext or create a new one with the VoodooTSCSync configurator.

The AGPMInjector kext is used to inject proper power management and can be created with Pavo-IM's generator.

The AMD-USB-Map kext is depending on the SMBIOS and can be created with the Hackintool. I've modified it according to Aleksandar's guide to use `IOKitPersonalities_x86_64` instead of `IOKitPersonalities` generated by the Hackintool. 
This seems to improve USB behavior in general (USB thumb drives are more responsive, loading and unmounting is noticably faster).


## Setup

### BIOS settings

Everything is tested with ASRocks latest BIOS v2.00 and v2.30/v2.60:

- CSM: disabled
- Above 4G decoding: disabled (must be enabled for certain older graphics card)
- Thunderbolt: enabled
  - Security Level: No Security
- Fast boot: disabled

Note: 
ASRock did a wide cleanup of BIOS options in v2.30, several options were removed like advanced PCIe settings and possibility to disable USB controllers... 

## USB port mapping
![Back I/O](./back_io.png)
The front USB ports on the internal USB 3 header are SS5/HS5 and SS6/HS6.
The port of the internal USB header is mapped to HS9, the internal Bluetooth module to HS10.
In case the XHC0 controller is disabled, the ports 3/4 on the back I/O are USB 2 only.


| XHC0 -> XHCI | | |
| --- | --- | --- |
| PRT1 | HS4 | USB 2 |
| PRT4 | HS9 | internal USB 2 |
| PRT5 | HS2 | USB 2 |
| PRT6 | HS1 | USB 2 |
| PRT9 | SS1 | USB 2 |
| PRT10 | SS2 | USB 2 |

| XHC1 -> XHC | | |
| --- | --- | --- |
| PRT1 | HS6 | USB 2 |
| PRT2 | HS10 | Bluetooth |
| PRT3 | HS5 | USB 2 |
| PRT4 | SS10 | USB Type C |
| PRT7 | SS6 | USB 3 |
| PRT8 | SS5 | USB 3 |

| XHC0 -> XHC2 | | |
| --- | --- | --- |
| PRT7 | SS3 | USB 3 |
| PRT8 | SS4 | USB 3 |


## Known issues

Thunderbolt controller is not detected by MacOS unless device is already connected during boot. SSDT-TB3.aml file improves TB support (Thanks to [XinJiangCN](https://github.com/XinJiangCN) for that).

The integrated Intel Wifi is supported by [itlwmx](https://github.com/OpenIntelWireless/itlwm) though missing support for AirDrop etc.

Microphone is not yet working through integrated audio codec.

### Sleep

Sleep can be a difficult topic with little things breaking either entering or leaving sleep. Certain USB devices can break transition into sleep (resulting in a kernel panic after 3 minutes).
And make sure boot flag `-hbfx-disable-patch-pci` is set to avoid black screens on wakeup.

The following settings seem to work for consistent sleep:

```
$ pmset -g
System-wide power settings:
Currently in use:
 hibernatemode        0
 autorestart          0
 powernap             0
 disksleep            10
 sleep                10
 Sleep On Power Button 1
 ttyskeepawake        0
 hibernatefile        /var/vm/sleepimage
 tcpkeepalive         0
 gpuswitch            2
 displaysleep         10
```



If it does not entering sleep properly there are some things to be tried:

- Disable "Allow Bluetooth devices to wake this computer" in Advanced Bluetooth settings (may require the use of the power button if keyboard and mouse are connected through standard bluetooth)
- Disconnect specific bluetooth devices (like monitor speakers and the like)
- Turn off monitor before entering sleep


## Notes

Use at your own risk.

Discussions:
* [AMD OS X](https://forum.amd-osx.com/index.php?threads/ryzen-9-3900x-asrock-x570-itx-tb3-sapphire-rx-5500-pulse-catalina.19/)
* [Hackintosh Forum](https://www.hackintosh-forum.de/forum/thread/46160-ryzen-9-3900x-asrock-x570-itx-radeon-rx-5500-xt/)

Cedits and links:
* [OpenCore Desktop Guide](https://github.com/dortania/OpenCore-Desktop-Guide)
* [Aleksandar's USB mapping Guide](https://aplus.rs/2020/usb-mapping-why/)
* [AMD OC Ryzen](https://github.com/iGPU/AMD_OC_Ryzen)
* [OpenIntelWireless](https://github.com/OpenIntelWireless)
* [AGPM Injector](https://github.com/Pavo-IM/AGPMInjector)
* [trulyspinach's SMCAMDProcessor](https://github.com/trulyspinach/SMCAMDProcessor)
* [khronokernel's SmallTree](https://github.com/khronokernel/SmallTree-I211-AT-patch)
* [Hackintool](https://www.hackintosh-forum.de/forum/thread/38316-hackintool-ehemals-intel-fb-patcher/)
* [VoodooTSCSync Configurator](https://www.insanelymac.com/forum/files/file/744-voodootscsync-configurator/)
