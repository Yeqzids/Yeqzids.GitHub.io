---
layout: post
title: Installing Debian 7 + Xfce 4.8 on ThinkPad X60
excerpt: "In this tutorial we will go through the installation of Debian 7 (Wheezy) + Xfce 4.8 environment on a ThinkPad X60 1709APC laptop."
modified: 2014-04-20
categories: resource
comments: true
share: true
---

##Introduction##

In this tutorial we will go through the installation of Debian 7 (Wheezy) + Xfce 4.8 environment on a ThinkPad X60 1709APC laptop. ThinkPad X60 was released in January 2006 and it is among the last ThinkPad series that beared the "IBM ThinkPad" mark on its lower right corner. I have used the original Windows XP Home (which came with my ThinkPad X60), Ubuntu 12.04 LTS + Unity, 12.10 + Unity, Debian 6 + GNOME 2, Debian 7 + GNOME 3 (Default and Classic) and Xfce, and eventually decide to settle down at the (much) faster, newer and more stable Debian + Xfce. If you are interested in a comparison of each desktop environment, you may find [here](http://www.renewablepcs.com/about-linux/kde-gnome-or-xfce) as a good start.

The CPU of 1709APC (Intel Core Duo T2300) does not support x64 system so we must use the x86 version. The firmware of 1709APC also does not support physical memory over 3GB (even with PAE enabled).

The creation of this document is inspired by ["Debian Wheezy on Lenovo ThinkPad L420"](http://www.theiling.de/l420-wheezy.html). I got hints and helps from numerious websites and posts over years and I hope this document can be among one of them that are of some help to others.

##Revision History##

* 2014 April 20: update the solution to Fn+F7.
* 2014 April 17: creation.

##Installing Debian 7 + Xfce Environment##

The installation of the Debian 7 + Xfce 4.8 system is simple. You can download the most recent i386 intallation image from [here](http://cdimage.debian.org/debian-cd/7.4.0/i386/iso-cd/). For me, the ~630MB `debian-7.x.x-i386-xfce-CD-1.iso` is sufficient to use. You can burn it into your USB disk (make sure it is unmounted) with the following command. **Note**: make sure to change `/dev/sdc` to the actual address of your USB device!

```
$ cp debian-7.x.x-i386-xfce-CD-1.iso /dev/sdc
```

After this, you can directly boot from the USB disk and start the installation.

For me, these items work out of the box:

* Keyboard and trackpoint
* Display and VGA out (manual switch)
* Fn Keys: F3, F5, F7
* Volume control (but see fix)
* Fn+Home/End for brightness control (but see "remaining issues")
* USB ports
* SD card reader
* Ethernet

Most of the Fn keys do not work naturally even with `tpb` package installed, but this can be fixed by tweaks. There is an [excellent page](http://www.thinkwiki.org/wiki/How_to_get_special_keys_to_work) on thinkwiki.org on how to get Fn keys to work, but it looks to be outdated for Debian 7 (thinkpad_acpi comes with the installation anyway) and I have no luck with it, although it does offer some hints on diagnosing the issues.

##Fixing Issues##
###Beeps in Moving Cursor###
The laptop beeps when the cursor is moving out of range (e.g. backspace at the beginning of a text), and this cannot be turned off at the setup of the mixer. [Solution](http://forums.debian.net/viewtopic.php?f=10&amp;t=50584): create `/etc/modprobe.d/blacklist.conf`, insert the following lines and reboot:

```
blacklist pcspkr
blacklist snd_pcsp
```

###Fn+F2 Lock Screen###
The Fn+F2 does not work out of the box, but there is a fast and painless workaround with `xflock4` command which is included in `xscreensaver` package. Navigate to Application Menu > Settings > Keyboard > Application Shortcuts, and associate `xflock4` to Fn+F2 and we are done.

I personally would not recommend modifying the ACPI as it seems more complicated than the shortcut method, there could well be a clever way to do this.

###Fn+F4/F12 Suspend/Hibernation Not Working###
I am amazed that Fn+F4/F12 does not work out of the box, it seems not many people have this issue. The keys were working without any additional work for my last intallation (Debian 7.0 beta + GNOME 3.6). The reason is in fact simple but not obvious to me at the first place: in `xfce4-power-manager` you can control the behavior after sleep/suspend and hibernation keys have been pressed, and by default they are set to "do nothing"!

If you have tried this trick without luck and you think your issue may be hardware-related, you can try `acpi_listen` and `xev` to see whether your keys are really responsive, if nothing shows on your screen when you press the keys with these commands, then you may have a bigger trouble! Otherwise you can check if `pm-utils` is properly installed by, for example, manually executing `sudo pm-suspend` to see if the laptop can enter suspend mode properly.

As an ending note, to avoid reloading issues after hibernation, be sure to create the file `/etc/pm/config.d/thinkpad_acpi` and add a single line to it:

```SUSPEND_MODULES="thinkpad_acpi iwlwifi psmouse"```

###Fn+F4/F12 Suspend/Hibernation Not Locking the Screen###
After getting Fn+F4/F12 working, I realize that the screen is not locked with these two keys, while if you enter the suspend/hibernation mode using the Applications Menu, screen is locked properly. This seems directly opposite than [what Theiling reported](http://www.theiling.de/l420-wheezy.html). Theiling's tweak is working, but I have one occasion that when you want to enter suspend/hibernation mode via Fn+F4/F12 you have to lock and unlock your screen first. This issue magically goes away when I tweak other things so I am not sure what makes it disappeared.  I also tried [the method of 00screensaver-lock posted on archlinux](https://wiki.archlinux.org/index.php/pm-utils#Locking_the_screen_saver_on_hibernate_or_suspend) but it does not seems to be working.

Theiling's tweak is simply create `/etc/pm/sleep.d/40_xfce4-lock-screen` with the following content and make it executable:

```
#! /bin/sh -x
# Hack for xfce4-session v4.8 to support screen lock on sleeping.
case "$1" in
    suspend|hibernate)
        . /usr/share/acpi-support/power-funcs
        d=/tmp/.X11-unix
        for x in $d/X*; do
            displaynum=${x#$d/X}
            getXuser
            if test "x$XAUTHORITY" != x
            then
                export DISPLAY=":$displaynum"
                if xprop -root _DT_SAVE_MODE | grep -q xfce4
                then
                    KEY="/xfce4-power-manager/lock-screen-suspend-hibernate"
                    CMD="xfconf-query -c xfce4-power-manager -p $KEY || echo true"
                    DOLOCK=`(su "$XUSER" -c "$CMD") 2>/dev/null`
                    if test "x$DOLOCK" = "xtrue"
                    then
                        xflock4
                    fi
                fi
            fi
        done
    ;;
esac
```

Theiling also reports that the issue is fixed in Xfce 4.9.

###Fn+F7 Switch between Display/VGA###
The Fn+F7 key will invoke `xfce4-display-settings`, but this can be nasty when one disconnect an external monitor from the laptop: the display goes no where and you will have to manually restart `lightdm` to get the display back (I think there must be a cleverer way to deal with this). ~~After some search and experiment, I found an easy way to work around this, although not fully what I am hoping for: navigate to Application Menu > Settings > Keyboard > Application Shortcuts, and associate `xrandr --auto` to Fn+F7.~~ There is an excellent workaround as suggested [here](http://hoyhabloyo.wordpress.com/2012/01/18/xfce-display-switching-dual-single-monitor/): just create the following sh file, place it somewhere (e.g. `/etc/acpi/video-switch.sh`) and make it as executable (with `chmod +x`):

```
#!/bin/sh
# Based on the script from
# <span class="skimlinks-unlinked">http://quepagina.es/ubuntarium/vamos-a-personalizar-la-tecla-switch-display-de-portatil.html</span>

# Will do a cycling output from state 0 to 2 and use a sound as feedback
# State VGA LVDS Beeps
#   0    0   1      1
#   1    1   0      2
#   2    1   1      3
#   (back to state 0)  

#For identifying our monitors use xrandr tool and view output
LVDS=LVDS1      # could be another one like: LVDS, LVDS-1, etc
VGA=VGA1        # could be another one like: VGA, VGA-1, etc
EXTRA="--right-of $LVDS" # addtional info while dual display

# Lets check both LVDS and VGA state from the string "$display connected ("
xrandr | grep -q "$LVDS connected (" && LVDS_IS_ON=0 || LVDS_IS_ON=1
xrandr | grep -q "$VGA connected (" && VGA_IS_ON=0 || VGA_IS_ON=1

# Output switch cycle
if [ $LVDS_IS_ON -eq 1 ] && [ $VGA_IS_ON -eq 1 ]; then
    #Go to state 0 -&gt; just LVDS
    xrandr --output $LVDS --auto
    xrandr --output $VGA --off
    beep
elif [ $LVDS_IS_ON -eq 1 ]; then
    #Go to state 1 -&gt; just VGA
    xrandr --output $LVDS --off
    xrandr --output $VGA --auto
    beep && beep
elif [ $VGA_IS_ON -eq 1 ]; then
    #Go to state 2 -&gt; both outputs
    xrandr --output $LVDS --auto
    xrandr --output $VGA --auto $EXTRA
    beep && beep && beep
else
    #This should never be reached but just in case..
    xrandr --output $LVDS --auto
    beep && beep && beep && beep
fi
```

You can associate the script to Fn+F7, so it will automatically loop around the setup (laptop only > dual monitor same display > dual monitor extended display > external only) when you press Fn+F7. You can also automate the things when you plug/unplug the cable. Create `/etc/screen-autoswitch.hotplug` with the following content:

```
#! /bin/sh
#
# This is run from an udev rule.  It needs to find out which user is the X user and run switch-auto as
# that user in order to connect to the X server.
#

. /usr/share/acpi-support/power-funcs

d=/tmp/.X11-unix
for x in $d/X*; do
    displaynum=${x#$d/X}
    getXuser
    if test "x$XAUTHORITY" != x
    then
        sleep 2   # wait for monitor to settle
        for screen in 0 1
        do
            export DISPLAY=":$displaynum.$screen"
            /etc/acpi/video-switch.sh
        done
    fi
done
```

Make this file executable. Note that if you place `video-switch.sh` elsewhere you need to change to location accordingly. Lastly, create `/etc/udev/rules.d/40-display.rules` (does not need to be executable) with:

```
ACTION=="change", SUBSYSTEM=="drm", ENV{HOTPLUG}=="1", RUN+="/etc/screen-autoswitch.hotplug"
```

And here we done!

###Maximizing Window with Double Clicks on the Title Bar###
Initially, maximazing the window with double clicks on the title bar seems buggy to me, as the window may or may not be maximazed. This is because the default interval threshold (250 ms) is too short for me. Navigate thru Applications Menu > Settings > Mouse > Behavior, and increase the "Double Click Time" should do the trick. For me 400 ms is enough.

###ThinkPad Utilities###
Be sure to install `thinkfan` and `hdapsd` for automate fan control and IBM Hard Drive Active Protection System.

###Wireless Firmware###
The wireless firmware is in the non-free repository, so it does not come with the installation disk. First, we modify `/etc/apt/sources.list` and add the following lines:

```
deb http://ftp.us.debian.org/debian/ wheezy main non-free contrib
deb-src http://ftp.us.debian.org/debian/ wheezy main non-free contrib
deb http://security.debian.org/ wheezy/updates main non-free contrib
deb-src http://security.debian.org/ wheezy/updates main non-free contrib
```

Then we execute the following commands from the console:

```
$ su
$ aptitude update && aptitude install firmware-iwlwifi wireless-tools
$ modprobe -r iwlwifi
$ modprobe iwlwifi
$ iwconfig
```

###Workrave Applet Corruption###
The Workrave applet in the notification area is corrupted with the original Workrave 1.9.909 in Xfce 4.8. I remove the original one and compile the most recent version from `git` (1.10.2 at the time of writing) and it works like a charm.

##Remaining Issues##
* The drag-n-drop feature seems buggy with Iceweasel 24.4.0.
* Xfce/Thunar mounting SD card: after clicking the "eject" icon in the sidebar, the icon is still there despite the device has been ejected.
* Fn+Home/End moves in multiple steps.
* The volume control keys can only control the Alsa mixer called ThinkPad Console Audio Control, this leads to confliction if you use an external keyboard that can control volume and another mixer (like PulseAudio). Things are working, but just inconvenient.

##Untested##
* Fn keys player control.
* Fn+Space (screen zoom).
* Some old-fashion features like the modem, infrared port and 1394 port.
