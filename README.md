## Raspbian and rtl8812au monitoring mode

This guide was created to assist setting up the find-lf application for wifi trangulation. Steps 1-5 cover building of the rtl8812au driver for monitor mode support.

The kernel used for this was `4.9.41-v7+` however the steps are fairly generic so hopefully it will continue to work as raspbian gets updated.

## 1 - Prerequisites
The Raspbian Lite version works just fine so need to bother with the GUI for any of this. You will need to have the pi connected to the internet somehow to download all the required dependancies so a second working wifi adapter or connected via LAN cable is required.

## 2 - Update the Pi

- `sudo rpi-update`
- `sudo apt-get update`
- `sudo apt-get install libncurses5-dev git`

Now reboot

 ## 3 - Download the Pi Source code

Before we can build the wifi driver we need to download the source code for the pi kernel.
 - `sudo wget https://raw.githubusercontent.com/notro/rpi-source/master/rpi-source -O /usr/bin/rpi-source && sudo chmod +x /usr/bin/rpi-source && /usr/bin/rpi-source -q --tag-update`
 - `rpi-source`

  _Note: If this step errors and you have to re-run it first delete the downloaded data to start fresh `rm -rf ~/linux*`_

## 4 - Grab a copy of the rtl8812 driver

Its a specific copy of the driver you need that supports monitor mode. There are quite a few forks of the driver on github with
various tweaks and versions. I only found one that worked fully with monitor mode and channel switching.

https://github.com/astsam/rtl8812au

`cd ~/ && git clone https://github.com/astsam/rtl8812au.git`

### 4.1 - Ensure your card is on the driver supported list

Since WiFi adapters are being released all the time with the same chipset the list of supported devices can sometimes be out of date. This means even after compiling the driver for an rtl8812 and inserting a card with rtl8812 the os doesn't enable the card as expected.

The card I used was a **TP Link Archer T4U AC1300 Wireless Dual Band Adapter**. My card had no revision numbers on it so I assume its the 'v1' of the card. TP Link seem to switch chipsets at will so it can be a minefield getting the right card with the right chipset, internally v1 and v2 of the same model can be totally different pieces of hardware!

If you are sure you are using RTL8812AU chipset lets continue...

Execute `lsusb` and inspect the output

Look at the output to identify your card. If you are struggling do it twice, once with the card connected and once with it disconnected to see the difference.

Example :

```
Bus 001 Device 004: ID 2357:010d
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

Its the ID were interested in so take a note, if your card is the exact same as mine you will see `2357:010d`. The other devices are things embedded on the pi itself.

To check if the card is supported `cat ~/rtl8812au/os_dep/linux/usb_intf.c | grep 010d` (Replace the grep with the second half of your ID).

If you get a result then skip updating the driver, if not proceed to that step.

### 4.2 - Update the driver if required

We need to edit the following file
`nano ~/rtl8812au/os_dep/linux/usb_intf.c`

and add the line that matches the card into the list, note how the id found in the lsusb output has been split up `2357:010d`

`{USB_DEVICE(0x2357, 0x010d),.driver_info = RTL8812}, /* TP-link T4U AC1300 */`

## 5 -  Compile the driver

- `cd ~/rtl8812au`
- `make ARCH="arm" CROSS_COMPILE=arm-linux-gnueabihf- KSRC=/home/pi/linux/`
- `sudo make install`

Reboot! Now your card should work (green light should be flashing). You can validate its working using `ifconfig`, if you have a pi with built in wifi you will have `wlan0` and `wlan1`, if not just `wlan0`

## 6 - Configure Find

The documentation covers setting up the pi for find-lf so follow the instructions in the readme.

`https://github.com/schollz/find-lf`

There are a few niggles that can trip you up. If you have a PI with 2 wifi cards you will need to establish which one is wlan0 and wlan1 for the configuration. The find app/documentation assumes wlan1 however this can easily be changed if required when you run the initialize function.
The easiest way to check which card is which is to try put them into monitor mode `sudo iwconfig wlan0 mode monitor`.

If you only have 1 wifi adapter (for example because you have the older pi) the find application will attempt to toggle wifi adapter in and out of monitor mode, this is so it can monitor.. switch to normal wifi to obtain a valid internet connection, ship data.. switch back to monitor.. This may be desired behavior but if your pi is connected via the wired LAN interface its a pain, you can bypass it by modifying the client and commenting out the section that check for number of wlan interfaces.

`scan.py`
```

if num_wifi_cards() == 1:
    default_single_wifi = True
    default_wlan = "wlan0"
```

There is also the option to pass the command line argument `--single-wifi` false when executing `scan.py` however there doesnt seem to be a way of passing this when running everything via the `cluster.py` application
