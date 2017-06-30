# lede-modhostapd

Modified version of hostapd / wpad that supports restricting AP clients to
only those which have a dBm signal above a settable threshold.
This is based on LEDE 17.01.2 which incorporates hostapd 2016-12-19.

In a public wifi hotspot, you will often find users (STAs) trying to use
the AP from the fringe area of coverage.  
This is not desirable for several reasons:

1 They must be served at a low data rate and excessive retries, reducing 
   the usable bandwidth of the AP.
2 They create a "hidden node" problem since STA's far on one side of the AP 
   cannot detect when those far on the other side are transmitting.
3 They may be outside the intended area of coverage, such as a cafe.  This
   is a disadvantage to the owner who established the hotspot to encourage
   customers to stay inside the business.
   
Drawbacks 2 and 3 can be addressed by reducing the transmitter power of 
the AP.  But that impairs desired user's reception reducing system performance.

This modification causes hostapd to monitor the signal strength from the 
STA's.  If they are above a (settable) threshold, operation of the AP 
proceeds normally.  If a STA's signal is below the threshold, that STA is
not allowed to connect, or dropped if it is already connected.  Probe requests
from weak STAs are also dropped.  This may have some usefulness by limiting
the AP's transmission of Probe Responses in a crowded area.

Installation:
This was tested with LEDE 17.01.2 on ar71xx hardware, specifically a TP-Link
TL-WA701v2 with flash expanded to 16 MB.
Install the LEDE buildroot. (git.lede-project.org/source.git)
Make a default build for your target hardware. (outside the scope of this document)
Install LEDE on your router.  You may use a release binary since this 
package runs only in userspace.
Confirm that wpad-mini is selected.  This is usually the default.  It should work
with other variants of hostapd / wpad but this is not tested.
Place this tree at package/network/services/hostapd, replacing all files
there.  You can also do this with a link.
Rebuild the hostapd package:
make package/hostapd/compile
Find the wpad-mini .ipk file in bin/packages/<target>/base
scp that file to your router's /tmp
ssh to the router
root@LEDE:~# opkg remove wpad-mini
root@LEDE:~# opkg install /tmp/wpad-mini<version>.ipk
root@LEDE:~# killall hostapd
hostapd should now have restarted with the new version.  Confirm you are 
running the mod version by checking if the set_required_signal ubus call
exists:
root@LEDE:~# ubus -v list | grep set_required_signal
     "set_required_signal":{"connect":"Integer","stay":"Integer","strikes":
       "Integer","poll_time":"Integer","reason":"Integer"}

Important:  For routers with only 4 MB flash, it will not be possible to
do as shown above and remove the original wpad-mini to install a new one. 
A complete image must be built and flashed with the new wpad-mini in the squashfs.

Usage:       
Presently, configuration is only possible with a ubus call.  When first
started, the signal levels are set to -128 dBm, which is lower than any
actual signal.  So all connections will be accepted and retained as normal.
Set the log_level in the wifi-device section of /etc/config/wireless to 1 or
lower to see the relevant log messages.  Use logread -f to follow the log.
Connect a STA you should see something like this:
daemon.info hostapd: wlan0: STA f8:e0:79:XX:XX:XX MLME: auth request, signal -31 (Accepted)
In this case -31 is a very stong signal from having the STA device only a few
feet from the router.
Set connect and stay thresholds this way:
root@LEDE:~# ubus call hostapd.wlan0 set_required_signal '{"connect":-50,"stay":-50}'
At this point you should see signal reports: 
daemon.debug hostapd: wlan0: STA f8:e0:79:XX:XX:XX IAPP: -35 -38 (0)
daemon.info hostapd: wlan0: IAPP signal poll: 1 STAs, 0 dropped
The report is instantaneous signal, average signal and (number of strikes)
Experimentation will be needed to determine the optimal settings for the
particular situation.
The ubus call can be placed in /etc/rc.local to set up signal levels after
a reboot.

Settable parameters are:
"connect": signal (dBm) required for a received Probe, Auth Request, or 
           Assoc Request to be acted upon.
"stay"   : signal (dBm) required to stay connected.
"strikes" : Number of consecutive bad signal polls allowed before disconnection.
            Default 3.
"poll_time" : Time in seconds between signal polls.  Default 10.
"reason"  : IEEE 802.11 reason that will be transmitted to the STA upon
            disconnection.  Default 3 (Local Choice).  5 (AP too busy) is a
            reasonable alternative.  This does not affect the operation of
            hostapd at all.  It may (or may not) affect what the STA does
            after it has been disconnected.


Code details:
This project is based mostly on extending the ubus feature which allows dynamically
"banning" a STA based on MAC address.  A hook from the main part of hostapd
calls into the OpenWrt/LEDE-specific ubus.c module whenever a STA sends a
Probe, Authentication Request, or Association Request.  If the STA is not 
on the "banned" list, hostapd_ubus_handle_event will return 0 and hostapd
will respond to the request normally.  If the station is banned, the module
returns -2, which causes hostapd to drop (ignore) the request.

So it is straightforward to extend this routine by examining the signal level
metadata accompaining the packet and either accept or deny based on that.

Once allowed to connect, it is desirable to monitor the signal level of the
STA and drop them if they move away from the AP.  This is done by the 
hostapd_bss_signal_check() routine.  Once a station is fully connected, 
hostapd offloads the handling of data packets to the kernel driver.  Thus
hostapd is not aware of all packets received from the STA.  It is necessary
to call the driver to take a snapshot of the signal level.  These are the
same level numbers reported by for example iw station dump.

Since snapshot signal checks can be momentarilly low, signal_check has several
provisions to prevent unwarranted dropping of a STA.  First if the 
instantaneous signal is more than 5 dB below the "average", the signal report
is ignored completely.  The higher of the instantaneous and average is
compared to the "stay" threshold.  If it is below threshold, a "strike" is 
recorded for that STA.  When the strikeout count is reached, the STA is de-
authenticated (dropped).  If the signal is above threshold, the strike count
is reset to 0.

