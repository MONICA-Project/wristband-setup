# Crowd wristband deployment
<!-- Short description of the project. -->
The crowd wristbands need a dedicated infrastructure of base stations that communicate with each other
and with the wristbands. The maximum safe range between a wristband and a base station is 75 m. This
implies that a wristband must always be at maximum 75 m away from a base station in order to have coverage.
This characteristic can be used to design and setup the base station infrastructure for a specific venue.
Since there is a limit to the number of base stations (96) there is a limit to the maximum area that can be
covered. Hence, currently the spatial scalability is limited. The number of wristbands that is currently supported
is limited by the 3 bytes that are used to identify a wristband. There is no inherent limitation to the
number of wristbands in the protocol itself.

The base station radio is controlled by an ARM based PC board running Linux and the base station software.
The base stations themselves are joined together in a software cluster. A unique redundant-communication 
protocol has been developed that enables the use of multiple physical communication layers between the
base stations.

TCP/IP based communication, both Ethernet and WiFi, as well as several low-bandwidth wireless communication
technologies (Plexus) are supported. Altogether this creates a highly fault-tolerant communication
channel between the base stations. If for example the Ethernet or WiFi infrastructure fails, the messages are
still sent using the alternative available wireless infrastructure, making the system independent of the festival’s
infrastructure. A typical communication use case is a message that originates from a wristband, being
received by one or more base stations and further transported to our server node(s).
The server infrastructure is partly deployed locally on the festival premises and partly in the MONICA cloud.
This setup enables the mobile apps that are running on the visitor’s smartphones to interact with the system.
Again, for fault-tolerance reasons, the server infrastructure can be set up redundantly. Failure of a server
node does not result in failure of the entire system. A management console is available for operators to control
the entire system. Furthermore, the software on each base station can be updated simultaneously with a
single mouse click in a matter of seconds. The messages that are received from the wristband are used to
perform real-time triangulation to drive the crowd monitoring system, heat map visualisation and individual
wristband tracking.

## Installing the Cubietruck base stations 
<!-- Instruction to make the project up and running. -->

The project documentation is available on the [Wiki](https://github.com/MONICA-Project/template/wiki).

The hardware of the base station is a Cubietruck NUC. The NUC runs the Linux OS. The software that communicates with the wristband
is written in Java. In this section it is described how to configure the NUC

# Install Lubuntu

Install LiveSuite application: 
http://linux-sunxi.org/LiveSuit

Download lubuntu image: 
http://dl.cubieboard.org/software/a20-cubietruck/lubuntu/ct-lubuntu-nand-v2.0/server/ct-lubuntu-server-nand-v2.0.img.gz

Use FEL button to install

Attach monitor + keyboard

Set root password

```passwd root```

unlock

```passwd -u root```

edit /etc/ssh/sshd_config:
Change:
```
#PermitRootLogin without-password
PermitRootLogin yes
```

Restart ssh daemon

```service ssh restart```

Now use ssh to log in (detach monitor / keyboard )

# Install Dependencies

```apt-get install libjssc-java haveged openvpn easy-rsa usb-modeswitch linux-firmware salt-minion ntp hostapd isc-dhcp-server```

# Set hostname

Use basestation-[number]

```
/etc/hostname
/etc/hosts
```
reboot

# Disable IPv6
add to /etc/sysctl.conf:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

# Increase open file descriptors

add line to /etc/sysctl.conf:
```fs.file-max = 512000```

run

```sysctl -p```

edit /etc/security/limits.conf:

```
* soft     nproc          65535
* hard     nproc          65535
* soft     nofile         65535   
* hard     nofile         65535
root soft     nofile         65535
root hard     nofile         65535
```
reboot

(see http://stackoverflow.com/questions/21515463/how-to-increase-maximum-file-open-limit-ulimit-in-ubuntu)

# Configure network

Please ensure beforehand which eth1/2 interface is connected to which device. Make sure this is correct in the network config and DHCP config. The 3G/4G module should be on DHCP and have a route to the cloud. The cable LAN should have a static address and be configured for (handing out) DHCP

edit /etc/network/interfaces:

```
# Configure Wifi WAN: wlan1
auto wlan1
iface wlan1 inet dhcp
pre-up ip link set wlan1 up
pre-up iwconfig wlan1 essid "SENDRATO_HQ"
wpa-ssid "SENDRATO_HQ"
Wpa-psk "sendrato means wireless"
post-up route del default dev $IFACE

# Configure 3G/4G WAN: eth2
auto eth2
iface eth2 inet dhcp
up route add -host cloud.sendrato.com gw 192.168.8.1
post-up route del default dev $IFACE

# Configure Ethernet LAN: eth1
auto eth1
iface eth1 inet static
address 192.168.102.1
netmask 255.255.255.0

# Configure Wifi LAN (AP): wlan0
auto wlan0
iface wlan0 inet static
hostapd -dd /etc/hostapd/hostapd.conf
address 192.168.100.108
netmask 255.255.255.0

# Configure Ethernet WAN: eth0
auto lo eth0
iface lo inet loopback
iface eth0 inet dhcp
```
# Configure DHCP

edit /etc/dhcp/dhcpd.conf:

```
subnet 192.168.100.0 netmask 255.255.255.0 {
  interface wlan0;
  range 192.168.100.110 192.168.100.200;
  option subnet-mask 255.255.255.0;
  option domain-name-servers 8.8.4.4, 8.8.8.8;
  option routers 192.168.100.108;
}
subnet 192.168.102.0 netmask 255.255.255.0 {
  interface eth1;
  range 192.168.102.110 192.168.102.200;
  option subnet-mask 255.255.255.0;
  option domain-name-servers 8.8.4.4, 8.8.8.8;
  option routers 192.168.102.108;
}
```
# Configure Access Point

edit /etc/hostadp/hostadp.conf:

```
macaddr_acl=0
auth_algs=1
# Set below to 1 to make SSID invisible
ignore_broadcast_ssid=0
driver=nl80211
interface=wlan0
hw_mode=g
channel=11
ssid=Plexus
wpa=2
wpa_passphrase=this is a plexus node
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

## Configure on-board wifi chip to AP mode

edit /etc/modules:

```bcmdhd op_mode=2```

## Mode switch Huawei Dongle

(see: https://github.com/Sendrato/everywear/wiki/Using-the-Huawei-E3372h-153-dongle-with-a-Cubietruck)

edit /etc/usb_modeswitch.conf:

```
DefaultVendor=0x12d1
DefaultProduct=0x1f01
MessageContent="55534243123456780000000000000011062000000100000000000000000000"
```

# OpenVPN client as as service (using Upstart/Ubuntu instead of systemd)
(reference: http://serverfault.com/questions/458591/how-to-auto-start-openvpn-client-on-ubuntu-12-04-cli)

### Generate client certificate on server 

On cloud.sendrato.com:

```
cd /root/easy-rsa
./easyrsa gen-req [hostname] nopass
./easyrsa sign-req client [hostname]
```
### Install certificates on client

Copy following certificate files from server to client in /etc/openvpn

```
[hostname].crt
[hostname].key
ca.crt
```
Create /etc/openvpn/sendrato.conf (make sure to replace [hostname]!):

```
client
dev tun
proto tcp
remote cloud.sendrato.com 5001 
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert [hostname].crt
key [hostname].key
remote-cert-tls server
comp-lzo
verb 3
```

To automatically start Sendrato VPN make sure to edit

/etc/default/openvpn:

```
AUTOSTART="sendrato"
```
# Make sure that correct DNS server is always used

Edit 

/etc/dhcp/dhclient.conf

Add:
```
supersede domain-name-servers 8.8.8.8, 8.8.4.4;
```

# Install Java 8

```
echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main" | tee /etc/apt/sources.list.d/webupd8team-java.list
echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main" | tee -a /etc/apt/sources.list.d/webupd8team-java.list
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys EEA14886
apt-get update
apt-get install oracle-java8-set-default
```

# Salt high state

```
salt-call state.highstate
```

# Github configuration

Generate ssh key:

```
ssh-keygen -t rsa -b 4096 -C "info@sendrato.com"
```

Add ssh key to Github

```
mkdir /root/git
```

Check everywear-sendrato-proxy:

```
cd /root/git/
git clone git@github.com:Sendrato/everywear-navajo-proxy.git
```

# Configure service (this uses upstart)

Edit:  /etc/init/sendrato.conf

```
# sendrato.conf
start on (started networking)
stop on shutdown
chdir /usr/sendrato
script
export FILE_REPOSITORY_PATH=/root/git/everywear-navajo-proxy
export FILE_REPOSITORY_TYPE=multitenant
export FILE_REPOSITORY_DEPLOYMENT=default
export FILE_REPOSITORY_FILEINSTALL=config
exec java -Xmx256m -Djava.net.preferIPv4Stack=true -Dorg.osgi.framework.system.packages.extra=javax.smartcardio -Dorg.osgi.framework.storage=/var/sendrato/felix-cache -Dgosh.args=--nointeractive -Duser.dir=/usr/sendrato -jar /usr/sendrato/bin/org.apache.felix.main-5.4.0.jar > /var/sendrato/stdout.txt
end script
```

# UDev rules for DASH-7 modules

ik heb nu ook een manier gevonden om de UID vd nodes mee in de usb descriptor te krijgen, zodat je hiervan gebruik kan maken in een udev rule. Heeft als voordeel dat je udev een symlink kan laten aanmaken met daarin de UID vd node, zodat de file per node altijd dezelfde blijft (na reboot bv)

```
[13/3/17, 15:53:30] : 15:25 $ cat /etc/udev/rules.d/99-ttyacm.rules 
# silabs CDC
ATTRS{idVendor}=="10c4" ATTRS{idProduct}=="0003", SYMLINK+="d7_$attr{serial}", ENV{ID_MM_DEVICE_IGNORE}="1"
# EFM32 bootloader
ATTRS{idVendor}=="2544" ATTRS{idProduct}=="0003", ENV{ID_MM_DEVICE_IGNORE}="1"
[13/3/17, 15:53:40] : resulteert in:
[13/3/17, 15:53:43] : 15:25 $ ll /dev/d7*
lrwxrwxrwx 1 root root 7 Mar 13 15:18 /dev/d7_b570091418 -> ttyACM1
lrwxrwxrwx 1 root root 7 Mar 13 15:19 /dev/d7_b570091c0a -> ttyACM2
```

# Routing table

```
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         192.168.50.1    0.0.0.0         UG        0 0          0 wlan1
10.8.0.0        10.8.0.45       255.255.255.0   UG        0 0          0 tun0
10.8.0.45       0.0.0.0         255.255.255.255 UH        0 0          0 tun0
82.196.14.46    192.168.8.1     255.255.255.255 UGH       0 0          0 eth2
192.168.8.0     0.0.0.0         255.255.255.0   U         0 0          0 eth2
192.168.50.0    0.0.0.0         255.255.255.0   U         0 0          0 eth0
192.168.50.0    0.0.0.0         255.255.255.0   U         0 0          0 wlan1
192.168.100.0   0.0.0.0         255.255.255.0   U         0 0          0 wlan0
192.168.102.0   0.0.0.0         255.255.255.0   U         0 0          0 eth1
```

# ifconfig

```
eth0      Link encap:Ethernet  HWaddr 02:92:09:83:17:46  
          inet addr:192.168.50.84  Bcast:192.168.50.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3185 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2061 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:315807 (315.8 KB)  TX bytes:352977 (352.9 KB)
          Interrupt:117 

eth1      Link encap:Ethernet  HWaddr a0:ce:c8:1f:65:38  
          inet addr:192.168.102.1  Bcast:192.168.102.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth2      Link encap:Ethernet  HWaddr 0c:5b:8f:27:9a:64  
          inet addr:192.168.8.100  Bcast:192.168.8.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:285 errors:0 dropped:0 overruns:0 frame:0
          TX packets:158 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:91275 (91.2 KB)  TX bytes:20837 (20.8 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:10.8.0.46  P-t-P:10.8.0.45  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:31 errors:0 dropped:0 overruns:0 frame:0
          TX packets:31 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100 
          RX bytes:2604 (2.6 KB)  TX bytes:2604 (2.6 KB)

wlan0     Link encap:Ethernet  HWaddr 02:1a:11:f3:2e:af  
          inet addr:192.168.100.108  Bcast:192.168.100.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

wlan1     Link encap:Ethernet  HWaddr 00:0d:b0:02:cf:86  
          inet addr:192.168.50.85  Bcast:192.168.50.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:505 errors:0 dropped:0 overruns:0 frame:0
          TX packets:280 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:92282 (92.2 KB)  TX bytes:36721 (36.7 KB)
```



## Deployment

## Development
<!-- Developer instructions. -->

### Prerequisite

## Affiliation
![MONICA](https://github.com/MONICA-Project/template/raw/master/monica.png)  
This work is supported by the European Commission through the [MONICA H2020 PROJECT](https://www.monica-project.eu) under grant agreement No 732350.

