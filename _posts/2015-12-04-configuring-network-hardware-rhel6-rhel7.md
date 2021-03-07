---
title: Configuring new network hardware in RHEL6 and RHEL7
date: 2015-04-15
categories:
  - Linux
---

This is one of those things that seems obvious, until you actually need to do it.

In a RHEL 6 or RHEL 7 system, a combination of [udev rules](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/ch-Consistent_Network_Device_Naming.html) and the **biosdevname** utility are used to determine the names of ethernet devices. During installation, the OS will generate config files under **/etc/sysconfig/network-scripts/**,  using predefined udev rules to configure device naming.  

So, what happens when you add a second network card to the server after installation? Or, your distribution does the biosdevname switcheroo on you after install? You won't see any new config files for the new or renamed NICs.

Many forum posts suggest mucking around with the udev rules themselves- such as deleting the file **70-persistent-net.rules** in **/etc/udev/rules.d/** or **/lib/udev/rules.d** and then rebooting the system. At the time of writing though,, the udev configuration seems to have changed over time, and now there is little consensus as to which rules do what. You can sidestep the issue by creating the config files yourself.

First, make sure the new hardware was recognized by the kernel. Various ways to do this, but the below methods will give you the new NIC device names:
{% raw %}
```shell
# Quick and dirty, see the NICs showing in /sys/class/net:

$ ls /sys/class/net
em1  em2  em3  em4  lo  p4p1  p4p2  p4p3  p4p4

# Using the lshw utility, check availability in your repo (maybe in EPEL?)

$ lshw -short -class net
H/W path            Device      Class      Description
======================================================
/0/100/1/0          em1         network    NetXtreme II BCM5709 Gigabit Ethernet
/0/100/1/0.1        em2         network    NetXtreme II BCM5709 Gigabit Ethernet
/0/100/3/0          em3         network    NetXtreme II BCM5709 Gigabit Ethernet
/0/100/3/0.1        em4         network    NetXtreme II BCM5709 Gigabit Ethernet
/0/100/7/0/2/0      p4p1        network    82575GB Gigabit Network Connection
/0/100/7/0/2/0.1    p4p2        network    82575GB Gigabit Network Connection
/0/100/7/0/4/0      p4p3        network    82575GB Gigabit Network Connection
/0/100/7/0/4/0.1    p4p4        network    82575GB Gigabit Network Connection
```
{% endraw %}

In our case, the p4 devices are the new ones.  
  
If you want to make the config files by hand, you will need to get the mac address for each device- found in the file **/sys/class/net/{nicname}/address**  
{% raw %}
```shell
$ ls /sys/class/net
em1  em2  em3  em4  lo  p4p1  p4p2  p4p3  p4p4

$ cat /sys/class/net/p4p1/address
00:11:22:33:44:55
```
{% endraw %}    

Then just copy an existing NIC config file from **/etc/sysconfig/network-scripts/**, and rename it to **ifcfg-{nicname}**. Edit the HWADDR and DEVICE fields to reflect the new device. If using Network Manager, make sure to restart the service so that it can see the new network configs.

Those who want a more "wizard-like" experience can also use the old **system-config-network-tui** utility to generate config files (if using Network Manager, the **nmcli** tool may be a better fit) . It can be installed from standard YUM repositories. You can go to Device Configuration > New Device > Ethernet, enter the name of the interface and other info, and it will put a config file in **/etc/sysconfig/network-scripts/**.