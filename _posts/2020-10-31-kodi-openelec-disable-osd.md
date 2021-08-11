---
title: Disable the seek bar popup (OSD) in Kodi / OpenELEC / LibreELEC
date: 2020-10-31
tags:
  - Tech
  - DIY
---
Inspired by [this forum post](https://forum.kodi.tv/showthread.php?tid=356140&pid=2971509), and other fragmentary how-to's floating around the internet.  

If using Kodi to run videos in a loop for public display, you will run into an annoying quirk: A seek bar overlay (OSD) will pop up for a few seconds whenever a video starts, pauses, or loops. For public kiosks and [digital decorations](https://vimeo.com/446790471), this can really ruin the mood.

The default Kodi skin "Estuary" does not have a native way to disable this behavior, but there are two workarounds available:

1.  Under **Add-ons > Download > Look and Feel**, search for alternate skins which allow you to disable the OSD
2.  Edit or clone the default skin, and modify it to remove the OSD

We will cover option #2, using **Kodi 18 on LibreELEC** as an example. SSH/Linux commands are shown below, but much of this can be done over a network share via text editors and drag-and-drop.  

###  Step 1: Locate the files on your Kodi installation's filesystem

This varies based on how Kodi is installed, so I will try to catch some common scenarios.

*   Kodi 18+ on LibreELEC:
*   Access files using the [built-in network share](https://libreelec.wiki/support/update#samba-share), or command-line via Putth/SSH.
*   Original addons: **/usr/share/kodi/addons/**
*   Extra addons: **/storage/.kodi/addons/**
*   Kodi 18+ installed on Windows
*   Original addons: **C:\\Program Files\\Kodi\\addons\\**
*   Extra addons: **C:\\Users\\your\_user\\AppData\\Roaming\\Kodi\\addons\\**

### Step 2: Make a copy of the default skin

Using the above locations as a reference, go into the "Original" addons folder, and make a copy of the **skin.estuary** folder into the "Extra" addons folder. Give the copy a different name, i.e. **skin.estuary_no_osd**

`cp -r /usr/share/kodi/addons/skin.estuary /storage/.kodi/addons/skin.estuary_no_osd`

### Step 3: Change your new skin to remove OSD settings  

Go into your new skin folder under the "Extra addons" location, and edit the skin's **addon.xml** file with a text editor. Change the addon id and name to be unique.

{%raw%}`<addon id="skin.estuary_no_osd" version="2.0.27" name="Estuary_no_osd" provider-name="phil65, Ichabod Fletchman">`{%endraw%}

Lastly, from your new skin's folder, go into the **xml\\** subfolder, and edit the file **DialogSeekbar.xml** in a text editor.

`vi /storage/.kodi/addons/skin.estuary_no_osd/xml/DialogSeekBar.xml`

With the file open, locate the section at the top called **<visible>**, with various actions separated by pipes | . Remove any actions where you do NOT want the OSD to appear. For example, to hide the OSD on play or pause, remove the following items:

`Player.DisplayAfterSeek | [Player.Paused + !Player.Caching] |`

When removing, ensure that there is still a pipe separating each action item. Save the file, then restart Kodi or reboot your device.

![2020-10-31-disable_osd_1.PNG](/assets/images/2020-10-31-disable_osd_1.PNG)

### Step 4: Enable the skin in Kodi

After restarting Kodi or rebooting your device, go to **Add-ons > My add-ons > Look and feel > Skin** . You should see your new skin alongside the original. It is currently _disabled_, because Kodi thinks it is an unverified third-party skin. Click on the skin to see properties, then click the "Enable" button followed by the "Use" button.

![2020-10-31-disable_osd_2.PNG](/assets/images/2020-10-31-disable_osd_2.PNG)

Now pick a video and test it out! When the video starts, or when you pause and resume, the OSD should no longer appear.