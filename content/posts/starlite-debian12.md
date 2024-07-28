---
title: 'Setup Debian 12 on the Starlabs Starlite mk V'
date: 2024-07-28T17:50:41+02:00
draft: false
tags: ["starlite", "debian", "doc"]
author: "Alexandre"
---

Hello everyone, 

A little time ago, I received my [Starlite mv V](https://fr.starlabs.systems/pages/starlite) from Starlabs after a quiet long time.

Here is a technical documentation on how to do the full setup of the **Debian 12** and **Gnome** (wayland) install to have a fully working (in my opinion) hybrid pc.

## OS Install 

The Debian 12 installation is quite simple and works out of the box, I just used my [ventoy](https://www.ventoy.net/en/index.html) key (with a USB C to USB adapter) with the [Debian 12 iso](https://www.debian.org/download) on it. I followed the installation process as I would on any machine.

> **Disclaimer:** On my computer, I do not set the root password, so the installer installs sudo and adds my user to the sudo group. So some commands in the flowing article will use sudo, don't be surprised.

## OSK-SDL and Plymouth Install

> **Disclaimer:** Doing this can mess with your initramfs/grub. Proceed with caution, or you can brick your computer. You will then be forced to rescue your system by mounting your luks partition on a live CD or by reinstalling Debian 12.

### OSK-SDL

If, like me, you want to use full disk encryption, you can install OSK-SDL (from the Debian apt repository) to allow you to unlock your Starlite without the physical keyboard. It will also give you an OSK (on screen keyboard) during the Luks unlocking.

To install OSK-SDK run:
```bash
sudo apt install osk-sdl
```
Then you can edit your /etc/crypttab and add: `initramfs,keyscript=/usr/share/initramfs-tools/scripts/osk-sdl-keyscript`

Be careful to keep the discard at the end. Your file should look like this:
```
nvme0n1p3_crypt UUID=XXXX-XXX-XXX none luks,initramfs,keyscript=/usr/share/initramfs-tools/scripts/osk-sdl-keyscript,discard
```

You can then run:
```bash
sudo update-initramfs -u
sudo update-grub
```

And reboot your computer. If you get a black screen instead of a prompt for the unlock passphrase, touch the screen or press a key, and it should appear. You will also see the option to open the OSK in the bottom right corner of your screen.

### Plymouth 

Optional, but what's the point of having a fancy device if you do not have a fancy boot screen?

To install Plymouth, run:
```bash
sudo apt install plymouth plymouth-themes
```

Then I recommend to use the bgrt theme:
```bash
sudo plymouth-set-default-theme -R bgrt
```

You can then edit your grub config file (/etc/default/grub) and update the GRUB_CMDLINE_LINUX_DEFAULT like this: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`

Then run:
```bash
sudo grub-update
```

And reboot your computer.

## Fix the Bluetooth 

Out of the box, the Bluetooth isn't working. To fix it, you need to download the firmware manually. You can do it by running:

```bash
cd /usr/lib/firmware/intel/
sudo wget https://anduin.linuxfromscratch.org/sources/linux-firmware/intel/ibt-0040-2120.ddc 
sudo wget https://anduin.linuxfromscratch.org/sources/linux-firmware/intel/ibt-0040-2120.sfi
```

Then reboot the computer. If your Bluetooth still does not work, install rfkill and unblock the Bluetooth:
```bash
sudo apt install rfkill
sudo rfkill unblock bluetooth
```

## Update the firmware 

At the time I received my Starlite, I didn't have the latest firmware installed. You can install the latest firmware by running the following commands :
```bash
sudo apt install flashrom
wget https://github.com/StarLabsLtd/firmware/raw/master/StarLite/MkV/coreboot/24.07/24.07.rom
sudo flashrom -p internal -w 24.07.rom -i bios --ifd -n -N
sudo shutdown now
```

Once that has finished, please shutdown (not reboot), **disconnect the charger**, and wait for 12 seconds until you see the LEDs flicker. Once that happens, you can reconnect the charger and continue.

You can also check out this:
- [Github thread about the installation](https://github.com/StarLabsLtd/firmware/issues/184)
- [Release page](https://github.com/StarLabsLtd/firmware/tree/master/StarLite/MkV/coreboot) 

Updating the firmware when newer versions are released may improve many hardware-related things on this device.

## Fix the autorotate on the screen 

The kernel packaged in Debian 12 is not really bleeding edge, so you need to tweak a little the config to fix the screen autortate. I found this on the [Starlab's support page](https://support.starlabs.systems/kb/guides/starlite-fixing-rotation-on-older-kernel):
```bash
printf "sensor:modalias:acpi:KIOX000A*:dmi:*:*\n ACCEL_MOUNT_MATRIX=1, 0, 0; 0, -1, 0; 0, 0, 1;\n ACCEL_LOCATION=display\n" | sudo tee /etc/udev/hwdb.d/21-kiox000a.hwdb
sudo systemd-hwdb update
sudo udevadm control --reload-rules 
sudo udevadm trigger
```
Then, after a reboot, the autorotate should be fixed.

## Install TLP

The battery life on the Starlab does not give you a lot. So I installed TLP to slightly improve it: 
```bash
sudo apt install tlp tlp-rdw
sudo systemctl enable --now tlp.service
```

I use the default config. I do not have statistics about the gain, but If you do, feel free to share them!

## Install improved OSK

The default Gnome OSK is not really great, but this can be fixed with the awesome module: improved osk.

To install it you need to:
- [Install this add-on on Firefox](https://addons.mozilla.org/en-US/firefox/addon/gnome-shell-integration/) 
- [Install this module](https://extensions.gnome.org/extension/4413/improved-osk/) 

That's it!

## Firefox touch scroll and OSK

By default, Firefox does not scroll when you use the touch screen. It also does not open the OSK. To fix that, you can edit the /etc/security/pam_env.conf and add:
```
MOZ_USE_XINPUT2 DEFAULT=1
MOZ_ENABLE_WAYLAND=1
```
Then log-out and login again.

## Configure the microphone

Just a small thing I have noticed. If you let the microphone at 100% it will be horrible (saturation). I lowed it down to like 20/25% and it was much better.

## Conclusion 

That's it. I will keep this article updated if I make another config change, but I'm really happy with this configuration. The Starlite is working as I expected. I hope some information here can be of any help to you. I will do a full review (in French) after a few weeks of daily usage.

Comment this article on Mastodon.
