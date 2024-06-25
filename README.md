# Travel Router DIY

Imagine yourself in a situation where you in need to use Public WiFi insecure open-network meaning, to the HACKERS as well. But, you cant risk your data, what would you do ?

## Use your own Private Router (you need a raspberry pi 4 though!!)

easy let me explain, See this design first.

![Network Diagram](./images/NetworkDiagram.jpg)

*Note: Usual project involve taking Wireless network and Broadcasting another one private Network using external wifi adapter, but I tried using the existing ethernet port for providing the same.*

now you know.

## Requirements/Installation

- Raspberry Pi 4 Model B
- Raspberry Pi OS Lite (bookworm, 64-bit)
- Raspberry Pi Imager
- Your precious Time

## Installation Procedure.

#### No VPN method.

> Follow This

1. Image your Raspberry Pi , [I used this one.](https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-15/2024-03-15-raspios-bookworm-arm64-lite.img.xz), you can always experiment with other OS, if you're into that.
   **Don't forgot to setup the Raspberry Pi OS while imaging, meaning setup the username, password amd very important WiFi (the network with which you are gonna ssh into. [*I used headless mode thats why, if you're using DISPLAY output you probably may not need to do this.*]) [refer official documentaion](https://www.raspberrypi.com/documentation/computers/configuration.html#connect-to-a-wireless-network).**
2. Let the Pi reboot and give it some time to get connected to WiFi (the one which you did setup earlier)
   1. you need to find the IP of the Raspberry Pi,
      `ifconfig` then
      `nmap -sn <your-local-ip>/24`
      `ssh <username>@<rasp-pi-local-ip>`
   2. or usually
      `ssh pi@raspberrypi.local` may or maynot work
3. once after you connected sucessfully to the raspberry pi, do the steps.

> `/etc/dhcpcd.conf`

```
interface eth0
static ip_address=192.168.34.1/24
```

**note: Comment out everything else in the file [refer the official write-up [1] for more info].**

> `sudo apt-get install isc-dhcp-server`

> `/etc/dhcp/dhcpd.conf`

Add this to the bottom of the file:

```
authoritative;
subnet 192.168.34.0 netmask 255.255.255.0 {
 range 192.168.34.10 192.168.34.250;
 option broadcast-address 192.168.34.255;
 option routers 192.168.34.1;
 default-lease-time 600;
 max-lease-time 7200;
 option domain-name "local-network";
 option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

> `/etc/default/isc-dhcp-server`
>
> add this to end of the file
>
> `INTERFACESv4="eth0"`

> `service isc-dhcp-server start`

> `/etc/sysctl.conf`

Add this line:

`net.ipv4.ip_forward=1`

> vim `isc-server`

contents of the file:

```
echo Starting DHCP server
service isc-dhcp-server start

echo Setting NAT routing
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT

DEFAULT_IFACE=`route -n | grep -E "^0.0.0.0 .+UG" | awk '{print $8}'`
if [ "$DEFAULT_IFACE" != "wlan0" ]
then
  GW=`route -n | grep -E "^0.0.0.0 .+UG .+wlan0$" | awk '{print $2}'`
  echo Setting default route to wlan0 via $GW
  route del default $DEFAULT_IFACE
  route add default gw $GW wlan0
fi
```

> sudo bash `isc-server`

#### Yes VPN Method

> `sudo apt install openvpn`

Download the .ovpn access file and call the VPN connection:

> `sudo openvpn {file}.ovpn`

Enter your username and password, wait for the connecton. The VPN
will be passing all network traffic through a new route: tun0 (as
opposed to eth0 or wlan0). To account for this, I will create a new file
based on the previous one and replace all instances of wlan0 with tun0:

```
echo Starting DHCP server
service isc-dhcp-server start

echo Setting NAT routing
iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
iptables -A FORWARD -i tun0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o tun0 -j ACCEPT

DEFAULT_IFACE=`route -n | grep -E "^0.0.0.0 .+UG" | awk '{print $8}'`
if [ "$DEFAULT_IFACE" != "tun0" ]
then
  GW=`route -n | grep -E "^0.0.0.0 .+UG .+tun0$" | awk '{print $2}'`
  echo Setting default route to tun0 via $GW
  route del default $DEFAULT_IFACE
  route add default gw $GW tun0
fi
```

OG author call this file as `isc-server-vpn`

```
sudo bash isc-server-vpn
```

Now my system will pass the VPN data into the ethernet port, and if the VPN connection fails, all network traffic will stop.

## Reference

[1]. In detail [write up](https://switchedtolinux.com/tutorials/wireless-internet-passed-to-ethernet-with-raspberry-pi)

Keywords: Raspberry Pi Travel Router, Private Router with VPN, Headless mode Raspberry Pi.
