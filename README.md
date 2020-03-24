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

# Install Java 8

```
echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main" | tee /etc/apt/sources.list.d/webupd8team-java.list
echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main" | tee -a /etc/apt/sources.list.d/webupd8team-java.list
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys EEA14886
apt-get update
apt-get install oracle-java8-set-default
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

## Deployment

## Development
<!-- Developer instructions. -->

### Prerequisite

## Affiliation
![MONICA](https://github.com/MONICA-Project/template/raw/master/monica.png)  
This work is supported by the European Commission through the [MONICA H2020 PROJECT](https://www.monica-project.eu) under grant agreement No 732350.

