---
title: Modular Interactive Kickstart Madness pt. 1
date: 2015-02-23
categories:
  - Tech
  - Linux
---

Kickstart is a great time-saver for making standard Linux builds, but the traditional Kickstart process grows brittle as you try to scale it out. You may find yourself tweaking your configs manually for every new build, or else making a million copies to cover every build scenario. When only 10% of the code is different between them, it creates a recipe for frustration later.
  
There are a few neat tricks you can use to make your Kickstart more adaptable. My favorites include...

## Keep it modular with %include files

You can break up almost all of the kickstart file into separate %include files:

{% raw %}
```shell
# misc.kick contents: root password, text mode, keyboard/language, timezone, etc.

%include /tmp/modules/misc.kick

# network.kick starts blank, and is filled out by the pre-install script.

%include /tmp/modules/network.kick

# disk.kick starts blank, and is filled out by pre-install script. Partition and disk info.

%include /tmp/modules/disk.kick

# List of packages and package groups. Research --nobase option before using it!

%packages --nobase --ignoremissing

%include /tmp/modules/packages.kick

%end #End for packages section
```
{% endraw %}

**Note**: Despite being found later in the kickstart file, the %pre script runs before the rest of the kickstart file is loaded. This means we can use the %pre section to tweak these module files before they take effect (see below).  

## Support kickstarting over multiple networks

You may need to kickstart a server over one of multiple networks, with content hosted on a network share. Instead of having a separate config file with network settings for each network, you can run some code in the %pre section to ping around and figure out which server to use for installation media.

{% raw %}
```
%pre

iotty=`tty`
exec 1> $iotty 2> $iotty


service netfs start

mkdir -p /tmp/kickmedia && chmod 755 /tmp/kickmedia

media_svrs="10.0.0.100 192.168.1.100 172.48.10.100"

for ip in $media_svrs ; do
  if ping -q -c 1 $ip ; then
    mount -o ro,vers=3 $ip:/path/to/install/media/ /tmp/kickmedia
    /tmp/modules/preinstall.sh 
  fi
done

%end
```
{% endraw %}
  
More info below on what you can do with the pre-install scripts.  

## Take the guesswork out of network config

One common annoyance when kickstarting over the network, is figuring out the name of the network device you are booting from- the installer may see it as "eth0", until you reboot and find that the OS renamed it to "em1."  
  
Instead of blindly configuring eth0 and hoping it's the right one, try the parameter \--device=bootif. It's Kickstart shorthand for: "use whatever network interface we booted from." You have to set the option "IAPPEND 2" and "ksdevice=bootif" in your PXEboot server's pxelinux.cfg file for this to work.  

{% raw %}
```shell
network --bootproto=static --device=bootif --gateway=$gateway --ip=$hostip --nameserver=10.0.0.10 --netmask=255.255.255.0 --onboot=on --hostname=$name --noipv6
```
{% endraw %}

## Use an interactive %pre script to tweak your install on-the-fly

Notice how the above network line had $variables for IP and hostname? Wouldn't it be nice if your pre-install script could ask you for that info, and just plug it into your %include file for networking?
  
By default, Kickstart runs in a console that don't accept user input... but we can use a redirection trick from RedHat to make the console interactive for our pre-install script. It takes two lines of code in the %pre section, as seen in the previous example:
{% raw %}
```
iotty=`tty`
exec 1> $iotty 2> $iotty
```
{% endraw %}

With an interactive pre-install script, you can really go wild customizing your installation. Some of the things you can do:

*   Ask what function a server has, and append different packages to your package.kick include file.
*   Let the user change partition sizes for /root, /var, swap
*   Prompt for what network settings to use (no more "default kickstart IP" for a server)

It may sound complicated, but a pre-install script can be nothing more than a bunch of case- and read- statements to get user input, followed by echoing your new settings into the various %include files. You can build out your include files line-by-line this way, or you can use the pre-script to copy other include files over your "default" ones for lego-like customization.
  
Continued in Part 2.