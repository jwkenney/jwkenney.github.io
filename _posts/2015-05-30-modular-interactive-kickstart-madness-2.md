---
title: Modular Interactive Kickstart Madness pt. 2
date: 2015-05-30
tags:
  - Tech
  - Linux
---

  
This is Part 2. See Part 1 here.  
  
Previously, we covered using a combination of %include files and an interactive pre- script to make Kickstart more flexible. Now we cover post-install customization.  
  

## The %post section

To make good use of the %post section, you should be aware of another quirk in Kickstart: _by default, the %post scripts run in a different environment than the %pre scripts_.
  
The Kickstart installer runs out of a temporary environment that exists in memory. During installation, Kickstart pushes content into a [chrooted](https://en.wikipedia.org/wiki/Chroot) environment that is mounted under `/mnt/sysimage/` . The `%pre` and main sections of Kickstart run from outside of the chroot, but the `%post` section runs from _inside_ the chroot by default. This saves you from having to prepend '/mnt/sysimage/' to every file path when you run your post content. If we need any of our Kickstart files to be available for the `%post` section, we will need to copy them into the chrooted environment first.
  
We do this by declaring two `%post` sections. The first `%post` runs outside of the chroot with the \--nochroot flag, and we use it to push our content from Kickstart's `/tmp` directory into `/mnt/sysimage/tmp/` . The second `%post` section runs from inside the chroot, and we can work as if we are sitting in the installed filesystem. You only need one `%end` tag to cover both `%post` sections.

{% raw %}
```shell
# Start a post section outside of chroot, so we can push kickstart files into chroot
%post --nochroot

# In the %pre script eariler, we mounted a NFS share to /tmp/kickmedia to pull down kickstart scripts
# Now we copy them into our chroot to access them within the installed environment
cp -r /tmp/kickmedia /mnt/sysimage/tmp/newkickmedia

# Start another post script (inside the chroot), and run your post-install routines.
# These are really running out of /mnt/sysimage/tmp, but the chroot lets us use normal file paths. 
%post --log=/root/ks.post_install.op --interpreter /bin/bash

/tmp/newkickmedia/postinstall.sh

%end #End post scripts
```
{% endraw %}

Like we did with the `%include` files, you can also use the `%pre` scripts to customize your `%post` scripts before they run.