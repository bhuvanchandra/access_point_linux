##<B>Setting up Embedded linux device as WiFi Hotspot<B>


###<B>Test Setup:<B>
* Colibri VF61 V1.1A
* Colibri Evaluation Board V3.1a/ Iris Rev 1.1
* Ambicom WL250N-USB Wireless 802.11b/g/n (V1.0a)

![alt tag](https://www.dropbox.com/s/dg6hcitfnbhhbjz/eval_3.1a_test_setup.jpg)
![alt tag](https://www.dropbox.com/s/ynrh8iofkscbdbc/iris_test_setup.jpg)
####During boot or upon insertion the following kernel messages identify Ambicom WL250N-USB stick:
````
[   10.153746] usb 1-1.1: new high speed USB device number 3 using fsl-ehci
[   10.299135] usb 1-1.1: New USB device found, idVendor=148f, idProduct=3070
[   10.309882] usb 1-1.1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[   10.324597] usb 1-1.1: Product: 802.11 n WLAN
[   10.332875] usb 1-1.1: Manufacturer: Ralink
[   10.340887] usb 1-1.1: SerialNumber: 1.0
[   10.384713] Registered led device: rt2800usb-phy0::radio
[   10.384904] Registered led device: rt2800usb-phy0::assoc
[   10.385062] Registered led device: rt2800usb-phy0::quality
````	

####In user space it can be identified by `lsusb` as follows:
````
root@colibri-vf:~# lsusb
…..
Bus 001 Device 003: ID 148f:3070 Ralink Technology, Corp. RT2870/RT3070 Wireless Adapter
…..
````	

####Ethernet/Network:
````
[   20.491401] eth0: Freescale FEC PHY driver [Micrel KS8041] (mii_bus:phy_addr=2:01, irq=-1)
[   20.517857] ADDRCONF(NETDEV_UP): eth0: link is not ready
[   22.500404] ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
[    7.721280] eth0: no IPv6 routers present
````	
````
root@colibri-vf:~# ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 00:14:2D:49:3D:8D  
          inet addr:10.18.0.116  Bcast:10.18.3.255  Mask:255.255.252.0
          inet6 addr: fe80::214:2dff:fe49:3d8d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:31427 errors:0 dropped:24 overruns:0 frame:0
          TX packets:2186 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:3677223 (3.5 MiB)  TX bytes:580461 (566.8 KiB)
````	
````
root@colibri-vf:~# connmanctl services
*AO Wired                { ethernet_00142d493d8d_cable }
root@colibri-vf:~# connmanctl services ethernet_00142d493d8d_cable                                                                                                                                           
[ ethernet_00142d493d8d_cable ]


  Type = ethernet
  Security = [ ] 
  State = online
  Favorite = True
  Immutable = False
  AutoConnect = True
  Name = Wired
  Ethernet = { Method=auto Interface=eth0 Address=00:14:2D:49:3D:8D MTU=1500 }
  IPv4 = { Method=dhcp Address=10.18.0.116 Netmask=255.255.252.0 Gateway=10.18.0.1 }
  IPv4.Configuration = { Method=dhcp }
  IPv6 = { }
  IPv6.Configuration = { Method=auto Privacy=disabled }
  Nameservers = [ 10.18.0.2 10.0.0.2 ] 
  Nameservers.Configuration = [ ] 
  Timeservers = [ 10.18.0.1 ] 
  Timeservers.Configuration = [ ] 
  Domains = [ toradex.int ] 
  Domains.Configuration = [ ] 
  Proxy = { Method=direct }
  Proxy.Configuration = { }
  Provider = { }
````	

###<B>To configure a hotspot requires several steps:<B>
* Configure the wireless adapter with IP address
* Configure UDHCP server
* Install and configure hostap-daemon  the user space daemon for IEEE 802.11 management
* Configure IP routing between wlan and ethernet


###<B>Configure the wireless adapter with IP address:<B>
Setting static IP for wlan0 manually :
````
root@colibri-vf:~#ifconfig wlan0 192.168.0.1 up
root@colibri-vf:~# ifconfig wlan0
wlan0     Link encap:Ethernet  HWaddr 00:10:7A:4F:73:B4  
          inet addr:192.168.0.1  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::210:7aff:fe4f:73b4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:578 errors:0 dropped:3 overruns:0 frame:0
          TX packets:441 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:97112 (94.8 KiB)  TX bytes:138516 (135.2 KiB)
````
###<B>Configure UDHCP server:<B>
Create a configuration file for udhcp server
Sample UDHCP server configuration file:
/etc/udhcpd.conf
````
# The start and end of the DHCP lease block
start           192.168.0.20
end             192.168.0.254


# The wireless interface used by udhcpd
interface       wlan0


# If remaining is true (default), udhcpd will store the time
# remaining for each lease in the udhcpd leases file. This is
# for embedded systems that cannot keep time between reboots.
remaining       yes


# The location of DHCP lease file
lease_file      /var/lib/misc/udhcpd.leases


# The location of the pid file
pidfile         /var/run/udhcpd.pid


# DNS servers that connected devices will use.  Use Google DNS.
opt    dns      8.8.8.8 8.8.4.4
# or Use openDNS
#opt dns 208.67.222.222 208.67.220.220
#or Use unblocks DNS
#opt dns 208.122.23.22 208.122.23.23


# The IP address of the access point
opt    router   192.168.0.1
opt    subnet   255.255.255.0
opt    domain   local


# 10 days of lease period
opt     lease   864000


# Optionally specify static lease(s)
#static_lease   00:51:AF:05:B0:05 192.168.0.100
#static_lease   00:51:AF:00:E1:02 192.168.0.110
````	
####<B>Install and configure hostapd  the user space daemon:<B>
hostapd recipe was available in open embedded, one can easily build it using OE build.

http://layers.openembedded.org/layerindex/recipe/7367/

Please refer the below knowledge base article for setting up oe-core:
http://developer.toradex.com/software-resources/arm-family/linux/board-support-package/openembedded-(core)

Once hostap-daemon was build, copy the respective .ipk package and dependencies to target (Colibri VF61) and install the ipk packages.

Installing ipk packages:
````
root@colibri-vf:~#opkg install hostap-daemon_1.0-r0_armv7ahf-vfp-neon.ipk
````	

After installing hostapd, a sample configuration file hostapd.conf will be available at /etc/hostapd.conf and a script file hostapd was populated in /etc/init.d/  for auto starting at boot time.

One can use/edit the configuration available by default or use a custom configuration.

####<B>Sample custom hostapd configuration file:<B>
/etc/hostapd.conf
````
# Interface used by Access Point
interface=wlan0


# Dump file for state information (on SIGUSR1)
dump_file=/tmp/hostapd.dump


# Firmware Driver
driver=nl80211


# Access Point SSID
ssid=ToradexColibriVF61-AP


# Country code (ISO/IEC 3166-1). Used to set regulatory domain.
# Set as needed to indicate country in which device is operating.
# This can limit available channels and transmit power.
country_code=IN-KA


# Operation Mode (a = IEEE 802.11a, b = IEEE 802.11b, g = IEEE 802.11g)
hw_mode=g


# Channel number (IEEE 802.11)
# (default: 0, i.e., not set)
channel=6


macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0


# Key management algorithm
wpa_key_mgmt=WPA-PSK
wpa_passphrase=toradexColibriVF61
wpa=2


# Set ciphers
wpa_pairwise=TKIP
rsn_pairwise=CCMP
````	

Configure IP routing between the wlan and ethernet:
For the first time we need to do the following steps for enabling ip forwarding.

####<B>Enabling IP forwarding:<B>
````
Uncomment the “#net.ipv4.ip_forward=1” line in /sys/sysctl.conf
````
````	
root@colibri-vf:~#sysctl -w net.ipv4.ip_forward=1
	 or
root@colibri-vf:~#echo 1 > /proc/sys/net/ipv4/ip_forward
````	

####<B>Create new iptables Rule:<B>
````
root@colibri-vf:~#iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
root@colibri-vf:~#iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
````	

####<B>Saving iptables:<B>
Save the iptables for future reuse.
````
root@colibri-vf:~#iptables-save > /etc/iptables.ipv4.nat
````
For configuring the wlan0 at start one can use systemd service files to do the initial configuration for setting the static ip for wlan0 and starting hostapd, udhcp daemons and restoring the iptable for IP forwarding.

####<B>Systemd service file for initializing wlan0 and starting hostap-daemon and udhcp-daemon:<B>
Create a new systemd service file:
/etc/systemd/system/access-point.service
````
[Unit]
Description=Access Point
Wants=network.target
Before=network.target
BindsTo=sys-subsystem-net-devices-wlan0.device
After=sys-subsystem-net-devices-wlan0.device


[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/ip link set dev wlan0 up
ExecStart=/bin/ip addr add 192.168.0.1/24 broadcast 192.168.0.255 dev wlan0
ExecStart=/usr/sbin/wpa_supplicant -B -iwlan0 -c /etc/wpa_supplicant.conf
ExecStart=/usr/sbin/hostapd -B /etc/hostapd.conf
ExecStart=/usr/sbin/udhcpd -fS /etc/udhcpd.conf


ExecStop=/sbin/ip addr flush dev wlan0
ExecStop=/sbin/ip link set dev wlan0 down


[Install]
WantedBy=multi-user.target
````	
####<B>Enabling access-point.service:<B>
````
root@colibri-vf:~#systemctl enable access-point.service
````	

######NOTE: There is no need of the script file hostapd in /etc/init.d as we are taking care of loading it at boot time using systemd service file one can remove the file. ( root@colibri-vf:~#rm /etc/init.d/hostapd )

####<B>Systemd service file for restoring iptables:<B>
Create a new systemd service file:
/etc/systemd/system/iptables-restore.service
````
[Unit]
Description=iptables
DefaultDependencies=false


[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables-restore /etc/iptables.ipv4.nat


[Install]
WantedBy=access-point.service
````
####<B>Enabling iptables-restore.service:<B>
````
root@colibri-vf:~#systemctl enable iptables-restore.service
````
####<B>Sample iptables.ip4.nat with ip forwarding:<B>
/etc/iptables.ipv4.nat
````
# Generated by iptables-save v1.4.17 on  Thu Jun 05 12:10:23 2014
*nat
:PREROUTING ACCEPT [19:2729]
:INPUT ACCEPT [19:2729]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o eth0 -j MASQUERADE
COMMIT
# Completed on  Thu Jun 05 12:10:23 2014
# Generated by iptables-save v1.4.17 on  Thu Jun 05 12:10:23 2014
*mangle
:PREROUTING ACCEPT [185:24908]
:INPUT ACCEPT [178:23356]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [93:14375]
:POSTROUTING ACCEPT [143:26355]
COMMIT
# Completed on  Thu Jun 05 12:10:23 2014
# Generated by iptables-save v1.4.17 on   Thu Jun 05 12:10:23 2014
*filter
:INPUT ACCEPT [31:3442]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A FORWARD -i wlan0 -o eth0 -j ACCEPT
COMMIT
# Completed on  Thu Jun 05 12:10:23 2014
````
Once everything was configured restart the module, and check whether all the services are up and running successfully.
####<B>Checking the systemd services:<B>
````
root@colibri-vf:~# systemctl status access-point.service  
access-point.service - Access Point
          Loaded: loaded (/etc/systemd/system/ap.service; enabled)
          Active: activating (start) since Tue 2014-03-18 22:01:00 UTC; 13min ago
         Process: 319 ExecStart=/usr/sbin/hostapd -B /etc/hostapd.conf (code=exited, status=0/SUCCESS)
         Process: 312 ExecStart=/usr/sbin/wpa_supplicant -B -iwlan0 -c /etc/wpa_supplicant.conf (code=exited, status=0/SUCCESS)
         Process: 310 ExecStart=/bin/ip addr add 192.168.0.1/24 broadcast 192.168.0.255 dev wlan0 (code=exited, status=0/SUCCESS)
         Process: 216 ExecStart=/bin/ip link set dev wlan0 up (code=exited, status=0/SUCCESS)
        Main PID: 329 (udhcpd)
          CGroup: name=systemd:/system/ap.service
                  |-318 /usr/sbin/wpa_supplicant -B -iwlan0 -c /etc/wpa_suppl...
                  |-328 /usr/sbin/hostapd -B /etc/hostapd.conf
                  `-329 /usr/sbin/udhcpd -fS /etc/udhcpd.conf
````
````	
root@colibri-vf:~# systemctl status iptables-restore.service 
iptables-restore.service - iptables
          Loaded: loaded (/etc/systemd/system/iptables-restore.service; enabled)
          Active: active (exited) since Mon 2001-01-01 09:11:12 UTC; 13 years 2 months ago
         Process: 67 ExecStart=/usr/sbin/iptables-restore /etc/iptables.ipv4.nat (code=exited, status=0/SUCCESS)
````
````	
root@colibri-vf:~# ps aux | grep -E 'hostapd|udhcpd'
root       328  0.1  0.4   4752  1056 ?        Ss   22:01   0:01 /usr/sbin/hostapd -B /etc/hostapd.conf
root       329  0.0  0.2   2152   684 ?        Ss   22:01   0:00 /usr/sbin/udhcpd -fS /etc/udhcpd.conf
````
####<B>Check the ethernet interface:<B>
````
root@colibri-vf:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 00:14:2D:49:3D:8D  
          inet addr:10.18.0.116  Bcast:10.18.3.255  Mask:255.255.252.0
          inet6 addr: fe80::214:2dff:fe49:3d8d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:6588 errors:0 dropped:2 overruns:0 frame:0
          TX packets:1123 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:922746 (901.1 KiB)  TX bytes:459961 (449.1 KiB)


lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:52 errors:0 dropped:0 overruns:0 frame:0
          TX packets:52 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2600 (2.5 KiB)  TX bytes:2600 (2.5 KiB)


mon.wlan0 Link encap:UNSPEC  HWaddr 00-10-7A-4F-73-B4-00-00-00-00-00-00-00-00-00-00  
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:10590 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:3197137 (3.0 MiB)  TX bytes:0 (0.0 B)


wlan0     Link encap:Ethernet  HWaddr 00:10:7A:4F:73:B4  
          inet addr:192.168.0.1  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::210:7aff:fe4f:73b4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1022 errors:0 dropped:3 overruns:0 frame:0
          TX packets:1011 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:444925 (434.4 KiB)  TX bytes:304509 (297.3 KiB)
````	
Now we can able to see the access point SSID on other wifi devices. One can connect to the access point with the appropriate pass key and able to browse the internet.
![alt tag](https://www.dropbox.com/s/c1qxhinfp0kc3q9/ap1.png)
![alt tag](https://www.dropbox.com/s/u3tb65di9146e4y/ap2.png)
![alt tag](https://www.dropbox.com/s/78mcdssb9ewz7ld/ap3.png)
