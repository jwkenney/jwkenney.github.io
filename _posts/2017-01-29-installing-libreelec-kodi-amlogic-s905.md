---
title: Installing OpenELEC/LibreELEC on the Amlogic S905 Chipset
date: 2017-01-29
tags:
  - Tech
---

Amazon is now flooded with media boxes that use the Amlogic S905 or S905X chipsets. Typically they come with a custom build of [Amlogic](http://openlinux.amlogic.com/Android/Mbox) 's reference OS, and they have Kodi pre-installed. At the moment, they are the cheapest way to play 4K and HEVC-encoded content. These builds have a number of downsides though:  

* They use older, shady one-off builds of Android 5.1 and Kodi, and there is very little quality control
* A full Android OS competes with Kodi for system resources
* The bundled Kodi is often "fully loaded..." with crapware and dozens of 3rd-party streaming plugins. Mine had 200(!) plugins installed, and it took almost an hour to remove them all.
* My biggest gripe: despite being advertised as "4K-ready", **some OS and Kodi builds are software-locked at 1080p resolution**- even if the hardware supports it. The vendor didn't mention that on their Amazon page.

If you just want to run Kodi, you can get a big performance boost by running [LibreELEC](https://libreelec.tv/) instead. It is a newer fork of OpenELEC , that is designed to run Kodi and nothing else. Best of all, you can run it directly from a SD / USB stick, and boot to your original OS if needed.  
  
My hardware model was the "MX Pro 4k", sometimes called the "MXQ Pro 4k". I used kszaq's unofficial Amlogic image for S905/S905X chipsets.  

* As of 08/2017, the latest build uses [LibreELEC 8.2 with Kodi 17](https://forum.libreelec.tv/thread/9319-8-1-3-libreelec-8-2-for-s905-s905x/)
* Newer builds may be available in the [parent forum](https://forum.libreelec.tv/forum-38.html).
* [General build instructions](https://forum.libreelec.tv/thread/5556-howto-faq-install-community-builds-on-s905-s905x-s912-device/) available for S905/S905x/S912 devices

  
Read the above forum instructions carefully, and modify as necessary for your chipset. Do not flash the image to internal storage, unless you are confident in your Linux skills.  
  
Some notes from trying version 7.2 of the build:  

* Wireless, 4K, remote control, and HEVC hardware decoding all worked out of the box.
* Some features like Bluetooth are experimental or unsupported.
* Make sure you download and swap in the correct device tree file (.dtb file) from the download page, according to your hardware.  Rename the file to **dtb.img** and overwrite the existing one on your burned LibreELEC image.
* After burning the image to SD/USB, you need to modify it to work with your hardware. At the download page should be a section for device trees. Download a device tree file (.dtb) file for your given hardware build; they are named according to what RAM and Network chip you have. For help figuring this out, go to [Amlogic](http://openlinux.amlogic.com/Android/Mbox)'s page and see what hardware your model contains. My model had 1GB RAM and 100MB ethernet, so I used file `gxbb_p200_1G_100M.dtb`. Rename your downloaded .dtb file to `dtb.img` , and replace the existing `dtb.img` file on your burned SD/USB stick.
* When booting to the Recovery / SD / USB image: The method varies by model, and the instructions give you a bunch of options. My build had an app called UPDATE&BACKUP, that let me trigger a "local update" by picking a random.zip file and hitting Update. Then the OS restarted in recovery mode, and from there the device automatically found and booted LibreELEC from the SD card. If you flashed LibreELEC to a USB stick, you may need to move it around for it to be recognized at boot- recovery mode only looks at the 'primary' USB port and the SD card for new boot images. Depending on the device, you may need to go into the BIOS and change boot order (but in my case it wasn't necessary).
* When running from an external card, LibreELEC behaves much like a "Live" Linux distribution with persistence- any changes are saved to the card, and you can choose to reboot back to the internal or external OS at will.
* When you start up LibreELEC / Kodi for the first time, it will likely boot with a lower resolution and default audio settings. To optimize:
   * Go into  Settings > LibreELEC > System  > Video Output , and edit your resolution according to your TV. Marvel at the wide variety of resolutions now available to you. (NOTE: most "4K" TVs are made for _UHD_ resolution, which is 3840 x 2160). If you get lots of stuttering during playback, try dropping to a lower resolution or framerate.
* If you have audio issues, go to Settings > LibreELEC > System  > Audio Output and try switching the audio output device or settings.
* Download some [Free (Libre) videos](http://www.libde265.org/downloads-videos/) encoded with HEVC/H.265, and test playback at various resolutions Make sure it plays without any stuttering or issues- or tweak your settings until it plays smoothly.
* If Installing to internal memory: If you ran it from SD/USB card for a while, and you are confident that you want to flash LibreELEC to local storage, you can use an installation script bundled in LibreELEC. Enable SSH access (System > LibreELEC > Services), and log into the device using Putty or another ssh client. Then type the command `installtointernal` , and it will copy all system files onto the device. You can optionally copy all of your user data as well, to keep any customizations you made while running on a SD/USB stick. Then reboot by typing the command `rebootfromnand`
* **Note:** Backup images are saved to the /flash/ folder on the SD card, in case you need to re-flash your original OS.
* **Note:** With some S509 hardware sets (like mine), there is a small chance that your device's internal storage is laid out in a way that prevents the image from booting on internal memory. If you are worried by the possibility of re-flashing the stock image to undo your changes, I would recommend running LibreELEC off an SD card, and leaving the original OS intact.

Performance for me was great! The boot times are crazy fast: about 2-3 seconds on a 60MB/s SD card. For comparison, the original OS took ~15-20 seconds to load, not including Kodi. The remote worked out of the box, but you can also [enable the Web interface](http://kodi.wiki/view/Web_interface), and use a phone app like [Yatse](https://play.google.com/store/apps/details?id=org.leetzone.android.yatsewidgetfree) for remote control instead.