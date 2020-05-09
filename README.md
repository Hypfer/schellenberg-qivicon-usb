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


## Communication
It appears that the device might be a general-purpose solution with a vendor-specific customizing that contains all the relevant business logic.

The device can enter at least three different modes where different command subsets are available:

* `B:0` - **Bootloader Mode** denoted by the on-board LED pulsing with a low duty-cycle. Probably a bootloader mode
* `B:1` - **Initial Mode** denoted by the on-board LED pulsing with a higher duty-cycle than `B:0`
* `B:2` - **Listening Mode** denoted by the on-board LED being lit all the time

When first connected to USB, the device will be in **Initial Mode**.

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

Entering anything else (like lowercase `hello`) in **Initial Mode** will enter **Listening Mode**.

In **Listening Mode**, the dongle will dump every Schellenberg message it receives to its serial output.
Line endings are denoted by `\r`.

### Communicating with Schellenberg Devices
It is possible to both listen to Schellenberg Communication around you as well as send your own commands to paired devices.

Since every Schellenberg Device has its own unique identifier which cannot be changed, there's no security risk here which is good.

#### Receiving Messages
As an example, pushing the middle button of a Schellenberg Remote may result in a message like this `ss118F365400029820CB` to appear on the serial monitor.

Lets take a closer look at the structure of this message:

|          | Meaning                | Notes                                                                                                                             |
|----------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| ss       | Schellenberg Prefix?   | Fixed it seems                                                                                                                    |
| 11       | Device Enumerator      | The ID used by the controlling device to identify the paired device.                                                              | 
| 8F3654   | Device ID              | Unique per remote / Stick / Gateway                                                                                               |
| 00       | Command                |                                                                                                                                   |
| 0298     | 2 Byte Counter         | Incremented on each new cmd. Proably to prevent replay attacks                                                                    |
| 20       | 1 Byte local counter   | Each message is sent 10 times by the remote. [0,2,4,8,12,16,20,24,28,32]                                                          |
| CB       | 1 Byte signal strength |                                                                                                                                   |

Since the communication is unidirectional, there's no way to make sure that the command has been received and executed.
To mitigate that, Schellenberg is simply spamming the same command 10 times in a row which probably works well enough.

The newest generation of Schellenberg devices claims to be bidirectional so that's probably something

#### Data Structures
Time to explain the data structures we saw in the received message. These are also used when sending messages.

##### Device Enumerator
The device enumerator can be anything from `0x01` to `0xFF`.

Schellenberg Remotes are using `0x01` to communicate to _all_ paired devices and - in case of the 5ch remote - `[0x11,0x21,0x31,0x41,0x51]` to address a specific one.
The USB Dongle however can use every possible value which leaves us with 255 controllable devices.

#### Commands
The following commands are currently known:

| Hex  | Binary    | Meaning            | Notes                                               |
|------|-----------|--------------------|-----------------------------------------------------|
| 0x00 | 0b0000000 | Stop               |                                                     |
| 0x01 | 0b0000001 | Up                 |                                                     |
| 0x02 | 0b0000010 | Down               |                                                     |
| 0x40 | 0b1000000 | Pair?              | Make the selected device listen to a new Remotes ID |
| 0x41 | 0b1000001 | Manual Up          | As long as the button is held                       |
| 0x42 | 0b1000010 | Manual Down        | As long as the button is held                       |
| 0x60 | 0b1100000 | Pair?              |                                                     |
| 0x61 | 0b1100001 | Set upper endpoint |                                                     |
| 0x62 | 0b1100010 | Set lower endpoint |                                                     |

#### Sending Schellenberg Messages
With this knowledge, we're now able to communicate with Schellenberg Devices.

##### Command Message Format
This is the structure of a command message:

|      | Meaning              | Notes                                                                |
|------|----------------------|----------------------------------------------------------------------|
| ss   | Schellenberg Prefix? | Fixed                                                                |
| A5   | Device Enumerator    | The ID used by the controlling device to identify the paired device. |
| 9    | ?                    | A lonesome nine with no known purpose. Required                      |
| 01   | Command              |                                                                      |
| 0000 | Padding              | Has to be 0000. Required.                                            |

The purpose of both the `9` as well as the trailing `0000` is yet to be found


Putting everything together, an example command may look like this:
```
ssA59010000
```
Which translates to "Change Device with Enumerater A5 to drive up".

The Dongle will answer with `t0` followed by `t1`. The Software on the SmartFriends-Box titles these ACKs, but they only seem to be an acknowledgement of sending.
After the process the Dongle will fall back into listening mode `B:2`

Any command sent between `t0` and `t1` will be rejected with a `tE`.
If you're implementing this protocol, make sure to queue commands and only send the next after you've received confirmation for the first

#### Pairing a Device with the Dongle

Since every Dongle, Remote and Gateway seems to have a unique identifier which can't be changed, 
the Device we are trying to control needs to know our unique ID and know where our Rolling Code is starting.
 
Apparently, this seems to be only required for signing purposes and not for encryption, because every other device can reed the Rolling Code in clear text.

The pairing process works like this:

1. Select the Device you want to pair on an already paired Remote.
2. Put the already paired Remote into Programming Mode by pressing the Button in the battery compartment (The LED of the selected Device on the Remote should now be blinking)
3. Press the middle `Stop` button on the blinking Remote. (When pairing a Shutter it should beep and rattle a bit)
4. Send a command which consist of a device number on the Remote ( for example `C1`) and the pair command (again when pairing a Shutter it should beep and rattle a bit)
5. Finished. You can now send commands to the device.

##### Example pairing procedure:
Pair Mode on Remote -> Stop Button on Remote -> Send Command `ssC19400000`-> Finished

You can now Control the device with
```
ssC19000000
ssC19010000
ssC19020000
```

### Supported Devices
To pair any device to the Dongle, you require at least one Schellenberg Remote.
This can be both the 1-Channel as well as the 5-Channel version.

These Devices have been successfully be controlled by using the information found here:

| EAN           | Art. Nr. | Name                         | Notes                                                  |
|---------------|----------|------------------------------|--------------------------------------------------------|
| 4003971202646 | 20264    | Funk-Markisenantrieb Premium | Doesn't support fixed positions between Open and Close |


### Misc
The button on the Dongle doesn't seem to do anything?
