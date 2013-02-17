RSVA11001 Adapter
=================

The [RSVA11001](http://www.newegg.com/Product/Product.aspx?Item=N82E16881147011) is a home surveillance DVR Kit sold by Newegg. Rosewill
is the 'house brand' of Newegg. The DVR box is identical to the one
sold with the RSVA12001, the difference is supposedly in the cameras.
The DVR is most likely not made explicitly for Newegg. I have not had
a reason to open it up, but if someone could contribute some information
about who the OEM is I would gladly include it.

I purchased this kit because it was reasonably priced and seemed to 
be packed full of features. I assumed that since it advertised a web
interface I could just use some open source software like 
[motion](http://www.lavrsen.dk/foswiki/bin/view/Motion/WebHome) to scrape
the web interface and get most of what I needed from that. 

Initial Impressions
------
The HTTP server on the box is extremely glitchy. While Newegg claims it is linux
powered, my guess is that all the code is actually running kernel mode
or something equally bizarre.
Sending it **well-formed** HTTP requests can crash it if the request
is not in the tiny grammar that the DVR accepts. It apprently does have
some sort of hardware-watchdog, because the box will thankfully reboot
itself about 20 seconds afterwards. Just browsing to a random URL
from my browser was enough to kill it in my tests.

Once I got it up and running I realized its ActiveX primarily,
which puts it firmly out of my reach since I am a linux user only. It 
appears to try and do some User-Agent detection. If you browse to it from
Firefox under Ubuntu it will give you a page that somehow gets the browser
to open the RTSP connection on the DVR(the actual images are not delivered
over HTTP).

I did some browsing on the included CD-ROM and concluded the following

* The CD includes almost no documentation, but a variety of clients.
* There are clients for J2ME, Android, iOS and other stuff I didn't 
recognize.
* The Android client is called 'Castillo Player'. Decompiliation with
jad showed me that it uses an apparently proprietary streaming tech. I 
assume the iOS version is the same thing in obj-c. Both versions had
some sort of blob that I think is native code for the target platforms.
* The J2ME version uses a web interface that does long-polling to 
return a MJPEG-ish stream. It returns up to some finite amount of 
bytes then closes the connection.
* All the coding was mostly likely done by people with little to no
best-practices experience, but a 'can do' attitude. It does work, but 
things like function calls are passed over in preference to complicated
while loops.

Problem
----
After some more testing I also figured out that no matter what you do,
no more than one client can be using any of the interfaces at once. A
second client connecting just boots the first or freezes it. This immediately
rules out most open-source motion detection software since it is has 
no concept of access exclusion. Plus, if you live in a shared housing 
arrangement it'd make sense that more than one person might want to check
up on the house at a time. 

Another note is that opening the RTSP stream takes eons. It 
seems to take  at least 30 seconds. Then no matter what it just cuts off after a
few minutes. So I consider it useless.

Solution
----
So, the most obvious solution I could think of is to write a proxy
that translates all requests to the box. I ended up writing some software
to just continously round-robin the JPEG stream from the web interface.
It writes it into files. They can be served with a web server to any
number of clients. Currently the cycle to round-robin is 

1. Open a connection
2. Send the HTTP request
3. Wait for at least one JPEG image
4. Write JPEG image to file
5. Close the connection.
6. Goto the next image stream.

The present resulting FPS is rather slow. Certainly not full motion
video like the cameras can deliver. But it does work!
I plan to improve things but presently I am just happy this works at all.
Right now it is just a proof-of-concept. I plan to finish converting
it into a useful piece of software soon.

Firmware Analysis
---
If you goto the [product page](http://www.rosewill.com/support/Support_Download.aspx?ids=26_133_412_1952)
you can download what appears to be firmware for the device. There is no 
explanation of what it is, or how to upgrade the device. The manual just says
to put it on a USB stick and boot the unit. I'm guessing you'd need to extract it first
and format the USB with a FAT32 filesystem, but I have not tried.

Inspecting the firmware
with a hex editor reveals strings such as  "U-Boot 2008.10-svn (Dec  3 2012 - 15:25:46)".
This led me to conclude that the device is using GPL'd software.  Other symbols in the blob are

    nand_select_chip
    nand_transfer_oob
    nand_fill_oob
    nand_scan_tail

These are definitely linux kernel functions. I also found the following
linux kernel image boot lines

    console=ttyAMA0,115200 root=1f02 rootfstype=jffs2 mtdparts=physmap-flash.0:384K(boot),1280K(kernel),5M(rootfs),9M(app),128K(para),128K(init_logo) busclk=220000000
    bootargs=console=ttyAMA0,115200 root=1f02 rootfstype=jffs2 mtdparts=physmap-flash.0:384K(boot),1280K(kernel),5M(rootfs),9M(app),128K(para),128K(init_logo) busclk=220000000
    bootcmd=showlogo;bootm 0x80060000.bootdelay=1.baudrate=115200.ethaddr=00:16:55:00:00:00.ipaddr=192.168.0.100.serverip=192.168.0.1.gatewayip=192.168.0.1.netmask=255.255.255.0.bootfile="uImage"

Apparently ttyAMA0 refers to a serial port on the SoC. Inside the box
there is a 5 pin port labeled UART0. 

These lines are the password for the root user in /etc/shadow format

    root:$1$$qRPK7m23GJusamGpoGLby/:0:0::/root:/bin/sh
    root:ab8nBoH3mb8.g:0:0::/root:/bin/sh
    
If someone wants to decrypt those, it would be of a great aid. A rainbow
table can probably be used.

It should be possible to force them to release the source code, or at least force
Newegg since it is an American company.

After more browsing I realized I was just looking at a JFFS2 filesystem
the hard way, so I extracted it from the firmware. It is under the 
reverse engineering directory. Based on the boot line 'mtdargs' value it
looks like what I have extracted thus far is just a single one of many
filesystems. The version of JFFS2 on my desktop is not fully compatible
with the images in the firmware. However, JFFS2 is not explicitly versioned.
I am downloading Debian Etch in the hopes that happens to be a similiar
enough version to resolve the problems.

Hardware
---

* Single board construction
* FPGA-based 32-bit ARM SoC, little-endian
* mPower mP2809 - 4 channel video + 5 channel audio
* mPower mP2807 - 4 channel video
* Realtek RTL8201CP
* 16 megabytes of non-volatile flash memory
* Hynix memory chips
* A single relay which seems to be related to the RS-432 port
* A 74HC140 (Hex-inverting schmitt trigger) that appears to be used for the ALARM port
* A port labelled UART0
* Two SATA ports
* Some miscellaneous power filtering and regulation analog components.

The mP2809 has a single DAC, I think that is how the device does 
composite out. I have no idea what does the VGA. It must be in the SoC 
as well.

Interestingly, there does not appear to a dedicated SATA controller. It
is most likely integrated into the SoC.

Software Stack
---
The software stack is 

* U-Boot Universal Loader
* Linux 2.6.24 with a modular kernel
* A number of proprietary kernel modules for the special purpose hardware, for example one is labelled 74hc1605 which is a discrete flip-flop.
* Proprietary kernel modules for hardware encoding of video, etc. There are some files referencing an FPGA, so this is a very general purpose core.
* Busybox userspace
* More stuff I have not analyzed yet

Credits
----
The HTTP Parser included in the software is http-parser available
on GitHub. Many thanks
to presenting a simple interface and a library free from a million
dependencies. Also, many thanks to the whole CMake team for writing
a build manager that doesn't suck.
