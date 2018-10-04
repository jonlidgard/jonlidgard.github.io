---
layout: default
title: "Cross Compiling wiringPi on Mac OSX using Docker"
date: 2018-10-03T09:00:00Z
tags: ["wiringpi","arm","osx"]
---

# Cross Compiling wiringPi on Mac OSX using Docker
For a gentle introduction to cross compiling for the pi with Docker try [here](https://desertbot.io/blog/how-to-cross-compile-for-raspberry-pi)

First we need a Docker image with the cross compilation tools installed, so we'll grab the one from above:

`docker pull mitchallen/pi-cross-compile`

Create the build directory & cd into it

`git clone git://git.drogon.net/wiringPi`

Create a lib structure

`mkdir -p usr/local/bin usr/local/lib usr/local/include usr/lib`

Even though we use the pitools version of ldconfig it still looks in /etc/ld.so.conf.d for the
libraries to cache. These are the Ubuntu x86 libs of the host os & not the Arm ones. 
I'm no expert on this so if someone has a nicer way of doing it please let me know, but here's a quick fudge (as the container dies anyway after the build exits):

Create a file libcross.conf with contents:
/build/usr/local/lib

In the ./build script add near the beginning:

```bash
rm /etc/ld.so.conf.d/*.conf 
cp ./libcross.conf /etc/ld.so.conf.d/
```

This temporarily removes all the x86 libraries from the cache & points ldconfig to look in our build dir for lwriringPi & lwiringPiDev

Patch the build file to setup ld.so.conf & remove references to sudo as not needed here: (To patch save the diff to a file & use patch ./build <diff file>

```bash
46c46,47
< sudo=${WIRINGPI_SUDO-sudo}
---
> rm /etc/ld.so.conf.d/*.conf 
> cp ./libcross.conf /etc/ld.so.conf.d/
75c76
<   echo -n "wiringPi: " ; $sudo make uninstall
---
>   echo -n "wiringPi: " ; make uninstall
77c78
<   echo -n "DevLib:   " ; $sudo make uninstall
---
>   echo -n "DevLib:   " ; make uninstall
79c80
<   echo -n "gpio:     " ; $sudo make uninstall
---
>   echo -n "gpio:     " ; make uninstall
131c132
<   $sudo make uninstall
---
>   make uninstall
135c136
<     $sudo make install-static
---
>     make install-static
139c140
<     $sudo make install
---
>     make install
146c147
<   $sudo make uninstall
---
>   make uninstall
150c151
<     $sudo make install-static
---
>     make install-static
154c155
<     $sudo make install
---
>     make install
163c164
<   $sudo make install
---
>   make install
171c172
< # $sudo make install
---
> #  make install
```

Patch the following MakeFiles:

wiringPi MakeFile diff:

```bash
25c25
< DESTDIR?=/usr
---
> DESTDIR?=/build/usr
28c28,29
< LDCONFIG?=ldconfig
---
> LDCONFIG?=/pitools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/arm-linux-gnueabihf/libc/sbin/ldconfig
> 
39c40
< CC	= gcc
---
> CC	= /pitools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-gcc
```

GPIO MakeFile:

```bash
26c26
< DESTDIR?=/usr
---
> DESTDIR?=/build/usr
35c35
< CC	= gcc
---
> CC = /pitools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-gcc
```

devLib MakeFile:

```bash
25c25
< DESTDIR?=/usr
---
> DESTDIR?=/build/usr
28c28,29
< LDCONFIG?=ldconfig
---
> LDCONFIG?=/pitools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/arm-linux-gnueabihf/libc/sbin/ldconfig
> 
39,40c40,41
< CC	= gcc
< INCLUDE	= -I.
---
> CC = /pitools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-gcc
> INCLUDE	= -I. -I$(DESTDIR)$(PREFIX)/include
```

Start the compilation:

```bash
docker run -it -v <insert path to your build directory here>:/build mitchallen/pi-cross-compile ./build
```

use 'build clean' to cleanup if required
That will map the wiring dir to /build in the container & run the build script

Check the usr dir structure for the bins libs & includes


## Compiling the Examples

So now we've built the wiringPi libs & gpio it's time to try & build one of the examples.

Again, we edit the examples/MakeFile to point to the cross compiler & give it access to the headers in our build dir.

```bash
CC = /pitools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf-gcc
INCLUDE	= -I. -I../usr/local/include
```
Now add a quick & dirty build script (call it build-examples) to your build directory (the same one with 'build' in it)
```bash
#!/bin/sh -e

rm /etc/ld.so.conf.d/*.conf 
cp ./libcross.conf /etc/ld.so.conf.d/

cd examples
make clean
make really-all
```

Remember to chmod +x ./build-script then:
```bash
docker run -it -v `pwd`:/build mitchallen/pi-cross-compile ./build-examples
```
