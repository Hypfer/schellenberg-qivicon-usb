# Schellenberg USB protocol reverse-engineered

![The Device](https://user-images.githubusercontent.com/974410/80019349-a9cf6d80-84d7-11ea-8ea5-4d4418ef69bf.png)


## Overview
`The Schellenberg 21009 Magenta SmartHome Funk-Stick` is a USB dongle to connect Schellenberg Products to the QIVICON Smarthome System.

As of now, this is the only way to interface with Schellenberg devices via something else then the official remote controls.
Sadly though, the QIVICON Smarthome system is a closed-source source solution requiring monthly payments.

This repository documents how to interface with said USB Dongle so that Schellenberg devices can be used via proper FOSS Smarthome software.


Currently, this is WIP. Help is greatly appreciated.

## The Device

### General
As it is the case with other vendors, this device reports itself as a USB to serial converter:

`dmesg` output:
```
usb 1-4: new full-speed USB device number 6 using xhci_hcd
usb 1-4: New USB device found, idVendor=16c0, idProduct=05e1
usb 1-4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
usb 1-4: Product: ENAS
usb 1-4: Manufacturer: schellenberg
usb 1-4: SerialNumber: 000000000000
cdc_acm 1-4:1.0: ttyACM0: USB ACM device
usbcore: registered new interface driver cdc_acm
cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters 
```

`lsusb` output:

```Bus 001 Device 006: ID 16c0:05e1 Van Ooijen Technische Informatica Free shared USB VID/PID pair for CDC devices ```

The SerialNumber seems to always be `000000000000`

Other than that, it seems to respond to a lot of different baudrates. Apparently it's clever enough to auto-negotiate or something like that?


### Communication
It appears that the device might be a general-purpose solution with a vendor-specific customizing that contains all the relevant business logic.

The device can enter at least three different modes where different command subsets are available:

* `B:0` denoted by the on-board LED pulsing with a low duty-cycle. Probably a bootloader mode
* `B:1` denoted by the on-board LED pulsing with a higher duty-cycle than `B:0`
* `B:2` denoted by the on-board LED being lit all the time

When first connected to USB, the device will be in `B:1`.

All known commands are prefixed with a `!` and seem to be uppercase-only.

`!?` for example returns `RFTU_V20 F:20180510_DFBD B:1` which is both the version as well as the current mode.

Here's a list of all currently known commands:

| CMD | Example output               | Meaning                | Notes                   |
|-----|------------------------------|------------------------|-------------------------|
| !?  | RFTU_V20 F:20180510_DFBD B:1 | Version + current Mode |                         |
| !B  | OK                           | Enter B:0              |                         |
| !G  | OK                           | Enter B:1              |                         |
| !F  | Si446x                       |                        | The transceiver module? |
| !R  | OK                           | Reboots the device     | Only available in B:0   |
| !L  | L:2                          | ??                     | Only available in B:0   |
| !*  | EE                           | Error                  | * = Wildcard            |

Entering anything else (like lowercase `hello`) in `B:1` will enter `B:2`.

In `B:2`, the dongle will dump every Schellenberg message it receives to its serial output.
Line endings are denoted by `\r`.

#### B:2 Schellenberg Message Format
Pushing the middle button of a Schellenberg Remote causes `ss118F365400029820CB` to appear on the serial monitor.

This is the structure of the message:

|          | Meaning                | Notes                                                                                                                             |
|----------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| ss       | Schellenberg Prefix?   | Fixed it seems                                                                                                                    |
| 11       | Device Enumerater      | The Selected Device on the Remote (01 for all, 11-51 for Device 1 to 5). The Gateway uses A1-F1 (Maybe?! Have only seen A and B) | 
| 8F3654   | Device ID              | Unique per remote / Stick / Gateway                                                                                               |
| 00       | Command                |                                                                                                                                   |
| 0298     | 2 Byte Counter         | Incremented on each new cmd. Proably to prevent replay attacks                                                                    |
| 20       | 1 Byte local counter   | Each message is sent 10 times by the remote. [0,2,4,8,12,16,20,24,28,32]                                                          |
| CB       | 1 Byte signal strength |                                                                                                                                   |

The following commands are currently known:

| Hex  | Binary    | Meaning            | Notes                         |
|------|-----------|--------------------|-------------------------------|
| 0x00 | 0b0000000 | Stop               |                               |
| 0x01 | 0b0000001 | Up                 |                               |
| 0x02 | 0b0000010 | Down               |                               |
| 0x60 | 0b1100000 | Pair?              |                               |
| 0x61 | 0b1100001 | Set upper endpoint |                               |
| 0x62 | 0b1100010 | Set lower endpoint |                               |
| 0x40 | 0b1000000 | Manual Stop        | lol                           |
| 0x41 | 0b1000001 | Manual Up          | As long as the button is held |
| 0x42 | 0b1000010 | Manual Down        | As long as the button is held |

#### B:1 Sending Schellenberg Messages

To send a message to a device you have to go into `B:1` mode using `!G` and tell it what you want.
You start of with the `ss` Prefix. After that comes the Device Enumerater (see table above) for example `A5` which is followed by the command that one is sending.
At the End there need to be four zeros `0000` which at the moment are of unrecognized purposes.

So sending a command for exmaple would look something like:
```
!G
ssA59010000
```

So we are sending "Change Device with Enumerater A5 to drive up".
The Dongle will answer with `t0` followed by `t1`. The Software on the SmartFriends-Box titles these ACKs, but they only seem to be an acknowledgement of sending.
After the process the Dongle will fall back into listening mode `B:2`

### Misc
The button on the Dongle doesn't seem to do anything?
