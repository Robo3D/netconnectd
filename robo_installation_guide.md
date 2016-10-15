I used the netconnectd application to turn the robopi into a WAP, because it can be used to power an octoprint plugin.
https://github.com/foosel/netconnectd

This document is made up of 2 sections: installation steps and known installation issues.

# Installation steps
---

---


### Prepare the system

Install the hostapd, dnsmasq, logrotate and rfkill packages:

    sudo apt-get install hostapd dnsmasq logrotate rfkill

We don't want neither `hostapd` nor `dnsmasq` to automatically startup, so make sure their automatic start on boot is 
disabled:

    sudo update-rc.d -f hostapd remove
    sudo update-rc.d -f dnsmasq remove

You can verify that this worked by checking that there are no files left in `/etc/rc*.d` referencing those two services,
so the following to commands should return `0`:

    ls /etc/rc*.d | grep hostapd | wc -l
    ls /etc/rc*.d | grep dnsmasq | wc -l

If you are running NetworkManager (default for Ubuntu or other desktop linux distributions, usually not the case for 
Raspbian), make sure to disable its own `dnsmasq` by editing `/etc/NetworkManager/NetworkManager.conf` and commenting
out the line that says `dns=dnsmasq`, it should look something like this afterwards (note the `#` in front of the
`dns` line):

    [main]
    plugins=ifupdown,keyfile,ofono
    #dns=dnsmasq
    
    no-auto-default=00:22:68:1F:83:AF,
    
    [ifupdown]
    managed=false

You'll also need to modify `/etc/dhcp/dhclient.conf` to include a timeout setting, e.g.

    timeout 60;

Otherwise -- due to a limitation of how Debian/Ubuntu currently parses Wifi configurations in `/etc/network/interfaces` 
-- netconnectd won't be able to detect when it couldn't connect to your configured local wifi and will never start the 
access point mode. The value above will mean that it will take a maximum of 60sec before netconnectd will be notified 
by the system that the connection was unsuccessful -- you might want to lower that value even more but keep in mind that 
your wifi's DHCP server has to respond within that timeout for the connection to be considered successful.

### Check that your wifi card supports AP mode

Before you continue **make absolutely sure** that hostapd works with your wifi card/dongle! To test, create a file 
`/tmp/hostapd.conf` with the following contents:

    interface=wlan0
    driver=nl80211
    ssid=TestAP
    channel=3
    wpa=3
    wpa_passphrase=foofoofoo
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP CCMP
    rsn_pairwise=CCMP

Then run 

    sudo hostapd -dd /tmp/hostapd.conf

This should not show any errors but start up a new access point named "TestAP" and with passphrase 
"MySuperSecretPassphrase", verify that with a different wifi enabled device (e.g. mobile phone).

If you run into errors in this step, solve them first, e.g. by googling your wifi dongle plus "hostapd". You might need 
a custom version of hostapd (e.g. for the [Edimax EW-7811Un or other RTL8188 based cards](http://jenssegers.be/blog/43/Realtek-RTL8188-based-access-point-on-Raspberry-Pi)) 
or a custom driver. If you change anything related to `hostapd` during getting this to work, verify again afterwards
that the automatic startup of `hostapd` is still disabled and if not, disable it again (see above for infos on how
to do that).

### Install netconnectd

It's finally time to install `netconnectd`:

    cd
    git clone https://github.com/foosel/netconnectd
    cd netconnectd
    sudo python setup.py install
    sudo python setup.py install_extras

Modify `/etc/netconnectd.yaml` as necessary:
 
  * Change the passphrase/psk for your access point
  * If necessary change the interface names of your wifi and wired network interfaces
  * If your machine is **not** running NetworkManager, set `wifi > free` to `false`
  * if you **don't** want to reset the wifi interface in case of any detected errors on the driver level, set
    `wifi > kill` to `false`
 
Last, start netconnectd:

    sudo service netconnectd start

Verify that the logfile looks ok-ish:

    less /var/log/netconnectd.log

and that it's indeed running (error handling of the start up script still needs to be improved):

    netconnectcli status

Congratulations, `netconnectd` is now running and should detect when you don't have any connection available, starting the AP mode to change that.

You can control the daemon via `netconnectcli`:

  * `netconnectcli status` displays the current status (which interfaces are connected, is the AP running, etc)
  * `netconnectcli start_ap` manually starts the AP
  * `netconnectcli stop_ap` manually stops the AP
  * `netconnectcli list_wifi` shows the wifi cells currently in range
  * `netconnectcli configure_wifi <ssid> <psk>` configures the wifi connection (`<ssid>` = the wifi's SSID, `<psk>` = the wifi's passphrase)
  * `netconnectcli select_wifi` manually brings up the wifi configuration

You can always get help with `netconnectcli --help` or `netconnectcli <command> --help` for specific commands.

If everything looks alright, configure the service so that it starts at boot up:

    sudo update-rc.d netconnectd defaults 98

# Known Issues
---

---

##dnsmasq automatic startup

### Summary: 

--- 
**Update 10/14/16**
New findings while trying to make a build script. Looks like wpa_supplicant was running which prevented hostapd from using the device. remove any wpa_supplicant in /etc/network/interfaces and it should work fine. 
---

Dnsmasq still automatically starts at boot.This conflicts with netconnectd's ability to start up an access point because the IP address that netconnectd would have used to bind its run of dnsmasq is already taken up by a previously booted dnsmasq run. **(resolved)**

### Leads for solving:

Verify whether dnsmasq is starting at boot or there's another program thats calling dnsmasq.

http://askubuntu.com/questions/218/command-to-list-services-that-start-on-startup https://help.ubuntu.com/community/UbuntuBootupHowto
http://manpages.ubuntu.com/manpages/precise/man8/update-rc.d.8.html
http://ubuntuforums.org/showthread.php?t=1417870

`netstat -anlp | grep -w LISTEN`: you'll be what IP dnsmasq is bound to

### Solution:

I made sure to uninstall dhcpcd `sudo apt-get remove dhcpcd`

I made sure uninstall dnsmasq and dnsmasq-base `sudo apt-get remove dnsmasq`

Reboot Pi

Reinstall dnsmasq `sudo apt-get install dnsmasq`

Edit `/etc/default/dnsmasq` and change ENABLED=1 to ENABLED=0 which will tell it not to run in daemon mode. My file looks like this: 

~~~
# This file has five functions:
# 1) to completely disable starting dnsmasq,
# 2) to set DOMAIN_SUFFIX by running `dnsdomainname`
# 3) to select an alternative config file
#    by setting DNSMASQ_OPTS to --conf-file=<file>
# 4) to tell dnsmasq to read the files in /etc/dnsmasq.d for
#    more configuration variables.
# 5) to stop the resolvconf package from controlling dnsmasq's
#    idea of which upstream nameservers to use.
# For upgraders from very old versions, all the shell variables set
# here in previous versions are still honored by the init script
# so if you just keep your old version of this file nothing will break.

#DOMAIN_SUFFIX=`dnsdomainname`
#DNSMASQ_OPTS="--conf-file=/etc/dnsmasq.alt"

# Whether or not to run the dnsmasq daemon; set to 0 to disable.
ENABLED=0

# By default search this drop directory for configuration options.
# Libvirt leaves a file here to make the system dnsmasq play nice.
# Comment out this line if you don't want this. The dpkg-* are file
# endings which cause dnsmasq to skip that file. This avoids pulling
# in backups made by dpkg.
CONFIG_DIR=/etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new

# If the resolvconf package is installed, dnsmasq will use its output
# rather than the contents of /etc/resolv.conf to find upstream
# nameservers. Uncommenting this line inhibits this behaviour.
# Note that including a "resolv-file=<filename>" line in
# /etc/dnsmasq.conf is not enough to override resolvconf if it is
# installed: the line below must be uncommented.
#IGNORE_RESOLVCONF=yes
~~~

Make sure that `/etc/init.d/` does not include a `dnsmasq` executable



---

##No Automatic AP startup

### Summary:

The AP that you've defined in `/etc/netconnectd.yaml` does not appear even when you know the pi is not connected to a wifi network. (SOLVED)

### Solution:

https://github.com/foosel/netconnectd/issues/11 

`
Ahh I think I found my issue, it looks like there's a race or other issue with netconnectd and whatever is managing wifi on Raspbian these days. If I explicitly comment out the wlan0 config in /etc/network/interfaces and then boot up the pi everything works great--hostapd fires up the AP and I can connect and access it just fine from another device. I'm betting during startup netconnectd does its thing and configures wlan0 as a AP, then later Raspbian's wifi network config goes in and breaks what was just done by netconnectd. Perhaps there's a way to reorder startup dependencies, but for now blowing away wlan0 config seems to work. Will close since this doesn't really seem like a bug in netconnectd.
`

Make sure that no other wifi configuration is set in `/etc/network/interfaces`. Netconnectd does not like "competiton".
