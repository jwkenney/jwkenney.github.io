---
title: Medeawiz Sprite Deep-Dive for Holiday Decorations
date: 2021-08-07
tags:
  - DIY
  - Holidays
---

When preparing for the holidays, I'm a big fan of digital decorations- which use a projector and [special videos](https://atmosfx.com/) to "decorate" a surface or create various special effects.

One requirement is that you need something portable to play your videos with. Some projectors can play files directly from a memory stick, but the performance is often bad. Amazon is flooded with generic media player boxes, but the quality is all over the place. DIY solutions like the Raspberry Pi can work well, but they often require a certain amount of fussing to configure and use in the field.

When looking for something more streamlined, I came across the [Medeawiz Sprite](http://medeawiz.com/)- a no-frills media player/repeater that is designed for digital signage in a commercial setting. Even better, it supports motion sensors and remote switches- so your decorations can react as people approach them! I bought mine through the AtmosFX store, but they can also be acquired through [Team Kingsley](https://www.teamkingsley.com/). Team Kingsley offers other Sprite accessories for more advanced uses.

![sprite-photo.jpg](/assets/images/sprite-photo.jpg)

*Pictured with motion sensor (included) and a nubby USB stick (not included). Tiny, but fierce.*

## Features and Interface

The Sprite kit comes with the following:

* The Sprite DV-S1, an IR remote, and power adapter
* Old-school Composite cables for analog video
* The Sprite has a 3.5mm I/O port, a 3.5mm Analog AV port, and an HDMI port.
* An I/O adapter plug is included that converts to 4 screw terminals, for other serial/controller accessories
* Some kits include a motion sensor, while others may include a push-button sensor.
* The manual is available online in the [Downloads page](http://medeawiz.com/Downloads.html), and includes a lot of useful information.
* **Not included:** HDMI cable or memory card

The UI is sparse, but easy to navigate. The "Play Mode" and "Control Mode" sections allow you to choose how to play your videos, and whether you want them to be triggered by a remote sensor. The Video Output Mode should be set to something your projector or screen supports. For advanced uses, the Sprite also supports Serial control via the I/O port. The manual provides information on what instructions can be passed to the device in Serial mode.

If playing videos in Control Mode (motion sensor or remote switch), you must ensure that your filenames begin with a three-digit number. The video starting with '000' will be the default footage, and those starting with '001' and above will be your trigger videos. When running with the latest firmware\*, the Sprite supports up to 200 trigger videos!

I created some test videos named [000.quiet.mp4](/assets/files/000.quiet.mp4) and [001.trigger.mp4](/assets/files/001.trigger.mp4), as a way to test the motion sensor and figure out its range. Here it is in action below:

<iframe width="800" height="480" src="/assets/files/sprite_trigger_720p.mp4" frameborder="0" allowfullscreen></iframe>

\**Note: A [firmware update](http://medeawiz.com/Downloads.html) is available for the Sprite, which adds support for multiple trigger videos. Upgrading is done via drag-and-drop; see the online manual.*

## Usage

This is the shortest section for a reason. Once your playback and output settings are dialed in, the device is truly plug-and-play. You copy your media to the root of a USB stick or memory card, insert the card into the Sprite, and the videos will begin playing automatically. To control the order the files are played in, just name them in alphabetical or numeric order.

The device was designed so that non-technical people could keep it running in the field. It auto-plays your content when powered on. Forget the remote? Just remove and re-insert the power cord; the device will power on. I could have used this simplicity last Halloween, when a kid tripped on the power cord leading to my projection setup. I was in tech support purgatory for the next 10 minutes, while I rebooted my DIY media player and fumbled through menus to bring my singing pumpkins back to life. With this device, that situation would be a non-issue.

## Performance Summary and Deep Dive

I tested the Sprite with a variety of digital decorations from AtmosFX, using a Sandisk "Ultra Fit" USB stick for storage, and a 1080p/60Hz monitor as the display. Overall performance was very good, with a few caveats.

| File | Bitrate | Sprite Video Output | Storage | Results |
|---|---|---|---|---|
| Silly Skeletons 1080p | 5000-13000kb/s | <span style='color:darkorange;'>HDMI_1080p_60Hz</span> | USB | <span style='color:darkorange;'>visual glitches on video transition</span> |
| Silly Skeletons 1080p | 5000-13000kb/s | <span style='color:green;'>HDMI_1080p_50Hz</span> | USB | <span style='color:green;'>good!</span> |
| Let it Snow 1080p | 5000-9000kb/s | HDMI_1080p_50Hz | USB | <span style='color:green;'>good!</span> |
| Fireworks 4th of July 1080p | 5000-13000kb/s | HDMI_1080p_50Hz | USB | <span style='color:green;'>good!</span> |
| Fireworks New Year 1080p | 5000-13000kb/s | HDMI_1080p_50Hz | USB | <span style='color:green;'>good!</span> |
| Turkey Tomfoolery 1080p | 5000-12000kb/s | HDMI_1080p_50Hz | USB | <span style='color:green;'>good!</span> |
| St. Patrick's Fireworks 1080p | 5000-15000kb/s | HDMI_1080p_50Hz | USB | <span style='color:green;'>good!</span> |
| St. Patrick's Clovers 1080p | 5000-13000kb/s | HDMI_1080p_50Hz | USB | <span style='color:green;'>good!</span> |
| Jack-o-lantern Jamboree 2 1080p<br>(native) | 7000-10000kb/s | HDMI_1080p_50Hz | SDHC+USB | <span style='color:darkorange;'>audio lag on some files</span> |
| Jack-o-lantern Jamboree 2 1080p<br>(compressed with HandBrake) | 1000-5000kb/s | HDMI_1080p_50Hz | SDHC+USB | <span style='color:green;'>good!</span> |

Some things I learned during testing:

* The [online manual](http://medeawiz.com/Downloads.html) is very informative and helpful with troubleshooting. You should read through it once, to get familiar with the device and common troublehsooting steps.
* Don't use bargain-bin USB sticks or memory cards. They *might* work, but they may also introduce lag or stuttering during playback.
* The Sprite has plenty of horsepower to play AtmosFX decorations. AtmosFX videos run at 1080p resolution and 30 frames per second, with a variable bitrate of 5-15Mbit/sec. The Sprite supports a max bitrate of 40-50Mbit/s for most file types.
* If your projector does not support the default output mode of the Sprite (1080p/60Hz), you may need to hit the "HDMI" button on the remote to cycle though output modes until you find one that is usable.
* If playing in trigger mode with a motion sensor, you can pick either "Trigger Low" or "Trigger High"- it doesn't matter. The distinction matters more for push-button or pressure-switch sensors, which distinguish between a button *press* (low) and a button *release* (high).
* If you encounter problems during video playback, you may need to fine-tune the Video Output Mode on the Sprite for your display. I had to change my output mode to **HDMI_1080p_50Hz** to avoid a common issue mentioned in the manual (visual glitches on video transition).
* The Sprite plays through files in *ASCII order*, which is basically case-sensitive alphabetical (i.e. 'Boy' comes before 'boy'). It goes something like: spaces > symbols > numbers > uppercase > underscores > lowercase. You can avoid thinking about this by putting a set of numbers at the beginning of each filename, to determine the play order (001, 002, 003...).
* The only decoration I had difficulty with, was the "Jack-o-Lantern Jamboree 2" series- the audio kept going out of sync with the video. When I compressed the videos with [HandBrake](https://handbrake.fr/) using the "Very fast 1080p30" preset, they played properly. I suspect that there was some kind of corruption in my video files, and that HandBrake fixed or bypassed it during transcoding.

## Tools for Editing and Troubleshooting

There are some free/open-source tools I would recommend for working with digital decorations:

* [VLC Media Player](https://www.videolan.org/vlc/): For playing videos on PC and getting media/codec information
* [Handbrake](https://handbrake.fr/): An excellent video transcoder (convert or compress videos)
* [OpenShot](https://www.openshot.org/): A user-friendly video editor

The online manual indicates what formats and bitrates are supported. If you have any doubt about the format of your video, VLC media player lets you view this information under the 'Tools' menu while the video plays:

![vlc-get-bitrate.gif](/assets/images/vlc-get-bitrate.gif)

HandBrake is very good at compressing, converting, and/or downscaling videos. It has a number of one-click conversion presets that simplify the process.

![handbrake-transcode.gif](/assets/images/handbrake-transcode.gif)

Lastly, if you are looking to make actual edits to your videos, add/remove audio, or splice videos together, OpenShot covers the basics very well.

![openshot.gif](/assets/images/openshot.gif)

## Final Verdict

**Pros:**

* 1080p decorations should play and transition seamlessly, as long as you find the right Video Output Mode.
* The device is commercial-grade, and made to run constantly. It will probably hold up better than the white-label stuff from Amazon.
* The UI is dirt-simple, and the Sprite is pure plug-and-play once configured. Load your videos onto a memory stick, power the device on, and away it goes!
* Motion sensor / pressure switch support is a killer feature that sets the unit apart from the cheaper stuff.
* For hardcore decorators, it supports serial control. [Team Kingsley](https://www.teamkingsley.com/) sells other accessories for the Sprite.

**Cons:**

* The Sprite does not try to auto-guess the resolution that your projector or screen supports. You may need to fumble with the 'HDMI' or 'AV' button on the remote, to find an output mode that works for your projector.
* Some fine-tuning may be required in the settings for smooth playback, depending on the videos you load and the display you are using.
* The kit does not include an HDMI cable; be sure to get your own.

**Neutral:**

* The device is tuned for auto-playing content in a loop. While you COULD use it as a general-purpose movie player, it would be an awkward experience.
* For auto-play, the Sprite is particular about how files are named and arranged (they must be at the root of the memory card, files play in ASCII order, etc). It's a tradeoff you make to have a simple, plug-and-play interface.

**Overall:**

As someone who [isn't afraid to tinker]({% link _posts/2021-08-07-sprite-media-player-holiday-decorations.md %}) with DIY solutions, I really appreciate how simple the Sprite is to use. The motion sensor is a killer feature that opens up many possibilities for interactive displays, and a recent update expanded those capabilities further. It isn't the cheapest option on the market, but it is probably the easiest to use for the features it provides!
