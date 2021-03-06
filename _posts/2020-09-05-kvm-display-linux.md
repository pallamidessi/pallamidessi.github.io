---
layout: post
title: "Display KVM switch on Linux with udev"
date: 2020-09-05 10:00:00
categories: software
featured_image: /images/cover.jpg
---

Following up on [Haim Gelfenbeyn's excellent article](https://haim.dev/posts/2020-07-28-dual-monitor-kvm/
) on defining a software KVM display switch
by using DCC/CI instructions, I made a Linux "version" with udev.

It essentially the same idea, listening for USB device events to see if the USB KVM
switch is added or removed and issue a DCC/CI command to switch the monitor as well.

I made it for my specific setup: I have a main, always-on, Linux box and
additional laptops, so all the logic is driving by the Linux machine, which
simplifies installation a bit. I used this [USB KVM
switch](https://www.amazon.co.uk/UGREEN-Sharing-Keyboard-Scanner-Printer-Black/dp/B01N6GD9JO?ref_=ast_sto_dp).


## DCC/CI commands with `dccutils`

I didn't know this was a thing, but it is possible to control monitors from a
computer by issuing [DCC/CI](https://en.wikipedia.org/wiki/Display_Data_Channel#DDC/CI
) commands!  
`dccutils` is a tool to inspect the available capabilities and options of a monitor and emit commands such as "changing
inputs", changing brightness, etc...  
I followed [this post](https://friendo.monster/posts/switching-monitor-inputs-with-ddcutil.html
) to query my monitor's interface and tried first in a shell to
change the input between DisplayPort (my Linux box) and HDMI-2 (my work
MacBook), which in my case resulted in those commands:
~~~ bash 
# Set input to DisplayPort
sudo ddcutil setvcp 0x60 0x0f
# Set input to HDMI 2
sudo ddcutil setvcp 0x60 0x12
~~~

Need to be run as sudo since DCC/CI is implemented with the I2C protocol, which
is fairly low-level and bad instructions *could* brick/crash the monitor.

## Triggering on `udev rules`

Cool, we know how to set the monitor input from a machine! Now we just need to
listen to USB event to see when the USB switch goes in or out and issue to
corresponding `ddcutils` command. Enter `udev` !

`udev` is a Linux subsystem for managing device events, especially to respond to
"hotplug" events, and run arbitrary code based on matching rules. The events
come from the kernel but the resulting actions are executed in userspace. Useful for
things like automatically backing up the content of a specific USB key to disk, etc...

Rules available in `/etc/udev/rules.d` on a Debian machine and look like this:
~~~ bash
ACTION=="add", SUBSYSTEM=="USB", ATTR{idVendor}=="XXX", ATTR{idProduct}=="YYY", RUN+="/my/handler/script.sh
~~~
Which in this case instructs: if an USB event of type "add" is detected with the
device's vendor id of `XXX` and product of `YYY` then run the handler script. A good
udev guide can be found [here](http://www.reactivated.net/writing_udev_rules.html)

Reload after adding/changing rules:
~~~ bash
sudo udevadm control --reload
sudo udevadm trigger
~~~

## Identifying the USB KVM device
Almost there! We just need to identify a few unique attributes for our rules.
I used `lsusb` and turned on and off the device a few times to identify it:

~~~ bash
[...]
Bus 007 Device 020: ID 05e3:0610 Genesys Logic, Inc. 4-port hub
[...]
~~~

Where `vendorId` is `05e3` and productId is `0610`. We will use those attributes to
match against "add" events and defined as `ATTR{vendorId}` and `ATTR{productId}` in the udev rule.  
For the "remove" events, the story is a bit different: the remove event don't have
the same attributes/fields due to the device being removed and thus some data are
not accessed/accessible anymore.

To find out, I ran `udevadm monitor --environment` add looked at the resulting
"remove" event:

~~~ bash
[...]

KERNEL[1635240.722923] remove   /devices/pci0000:00/0000:00:08.1/0000:2f:00.3/usb7/7-4/7-4:1.0 (usb)
ACTION=remove
DEVPATH=/devices/pci0000:00/0000:00:08.1/0000:2f:00.3/usb7/7-4/7-4:1.0
SUBSYSTEM=usb
DEVTYPE=usb_interface
PRODUCT=5e3/610/9226
TYPE=9/0/2
INTERFACE=9/0/2
MODALIAS=usb:v05E3p0610d9226dc09dsc00dp02ic09isc00ip02in00
SEQNUM=25200

[...]
~~~

I will match the remove event on the `PRODUCT`, which will be defined as
`ENV{PRODUCT}` in the rule

## Tidying up

We got all the moving parts for our switch !

The simplest implementation, for me, would be:

/etc/udev/rules.d/95-local.rules:
~~~ bash
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="05e3", ATTR{idProduct}=="0610", RUN+="/usr/local/bin/kvm_added.sh"
ACTION=="remove", SUBSYSTEM=="usb", ENV{PRODUCT}=="5e3/610/9226", RUN+="/usr/local/bin/kvm_removed.sh"
~~~

kvm_added.sh:
~~~ bash
#!/usr/bin/env bash

sudo ddcutil setvcp 0x60 0x0f
~~~

kvm_removed.sh:
~~~ bash
#!/usr/bin/env bash

sudo ddcutil setvcp 0x60 0x12
~~~

## Conclusion

I'm very happy with my 23£ KVM and it only took a few hours to research and learn
about udev and dcc/ci and have a working prototype. Will tidy up the implementation a bit as there is
zero error handling and logging. Another great missed opportunity to write code,
instead of making some bash cruft!  

----
