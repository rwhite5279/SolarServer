# SolarServer
Describes a Raspberry Pi Zero 2 W-based digital "little free library" running off a solar panel. CC-BY-4.0

by Richard White
2024-04-23

# How I Did It: Building the Solar-Server Library

# Intro

This project began with the idea of running a Raspberry Pi 4 server out of my home, powered solely by a solar panel. I already knew a little about running a Raspberry Pi, and knew how to set up a server on there. I'd never played with solar power, however, and knew that things could get complicated. Some online research led to some helpful references, and a quick shopping trip to Amazon got me just about everything I needed.

In the end, I had a small Raspberry Pi unit running on a 12-volt battery with a solar power panel to keep the battery charged. The Pi unit itself, rather than serving webpages over the Internet, acts as a local Access Point, serving up just one small website to anyone who logs into its wifi signal. The intention is that it acts as a Little Free Library, where people can come along and download free books or music.

This short guide is not going to cover every detail that you need to get the server up and running, but it will cover most of the important points. If you need more details, considering looking at some of the references listed at the end of this document.

This is a great little project--you should try it!

# Hardware

For this project you'll need the following items:

* [Raspberry Pi Zero 2 W](https://www.adafruit.com/product/5291) ($15)
* [microSD card](https://www.amazon.com/gp/product/B06XWN9Q99/?th=1") (and card reader for computer)
* Various microUSB dongles, a 5-Volt, 2.5-Amp power supply, etc.
* [Solar Panel Kit & Charge Controller](https://www.amazon.com/Topsolar-Monocrystalline-Maintainer-Controller-Adjustable/dp/B0CSCQKJ32?th=1) (30W, 12V, $50)
* [12-Volt battery](https://www.amazon.com/Yuasa-NP7-12-Sealed-Battery-Terminal/dp/B00FA61Q1G)
* [Battery Box](https://www.amazon.com/Camco-55363-Standard-Battery-Box/dp/B00EOX2QJC?th=1)

You'll also need a computer (Mac or PC) to help you initialize the microSD card and to get everything up and running.

Buying the hardware is the easy part. Getting it all set up and running is where the *real* fun begins!

# Software

The following is a somewhat high-level overview of the steps involved in getting the hardware up and running. There may be details in here that it is assumed you are already familiar with. If you get stuck, reach out and I can help!

* Assemble all materials in some place where you can work on them. Set aside a couple of hours if you've done this before, a couple of days if you haven't! :)
* With your laptop, download the [Raspberry Pi Imager app](https://www.raspberrypi.com/software/) and use it to install 32bit Raspbian Pi OS (Legacy, 32-bit) port of Debian Bullseye for installation on the microSD card. As part of that installation, be sure to specify a userid (I used `zero`), a password (I used `i@mR3.14Zero2W`), the Wifi router name and accompanying wifi password that can be used for wifi access (although you can also use an Ethernet cable if you have the adapter), as well as the `raspberrypi.local` name for access on a local network.
* Once the microSD card has been initialized, remove it from the laptop and plug it into the Raspberry Pi Zero 2 W. Then plug in the power supply to the the RPi Zero 2 W so that it can boot up.
* Use `ssh` to log into the RPi using credentials you provided during setup. If you are running a terminal on your laptop, and connected (via wifi or Ethernet) to the same network the RPi is on, you can use

    ```
    $ ssh zero@raspberrypi.local
    ```

    to log in.
* Get the latest updates installed on the RPi.

    ```
    $ sudo apt update && sudo apt upgrade -y
    ```
* Install Apache2 so that you'll be able to serve up a webpage from the RPi.

    ```
    $ sudo apt install apache2 -y
    ```

    Now change owner of files in directory so we can access them.
    
    ```
    $ cd /var/www/
    $ sudo chown -R zero: html
    ```

    Test to see if Apache installed by opening a browser and going to `http://raspberrypi.local` You should get the Apache test page.
* Install Required Packages:  
You'll need to install `hostapd` for the access point, `dnsmasq` for DHCP and DNS services, and `netfilter-persistent` and `iptables-persistent` for managing network rules.  

    ```
    $ sudo apt install hostapd dnsmasq netfilter-persistent iptables-persistent
    ```
* Configure a Static IP for the Wireless Interface  
    Edit the `dhcpcd` configuration file to set a static IP for the wireless interface (`wlan0`).

    ```
    $ sudo nano /etc/dhcpcd.conf
    ```  
    Add the following lines to the end of the file:

    ```
    interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
    ```
    This sets the Raspberry Pi’s WiFi to use the static IP `192.168.4.1`.
* Configure `hostapd`  
    Create and edit the `hostapd` configuration file:

    ```
    $ sudo nano /etc/hostapd/hostapd.conf
    ```
    Add the following configuration:

    ```
    interface=wlan0
    driver=nl80211
    ssid=YourNetworkSSID
    hw_mode=g
    channel=7
    wmm_enabled=0
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=YourPassword
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
    ```

    Replace `YourNetworkSSID` and `YourPassword` with the Wifi name and password that you want users to use when they log on to your Raspberry Pi.
* Link this file to the main `hostapd` configuration:  

    ```
    $ sudo bash -c "echo 'DAEMON_CONF=\"/etc/hostapd/hostapd.conf\"' >> /etc/default/hostapd"
    ```

* Configure `dnsmasq`  
    Backup the original `dnsmasq` configuration:

    ```
    $ sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    ```

    Create a new configuration file:  
    ```
    $ sudo nano /etc/dnsmasq.conf
    ```
    Add the following:

    ```
    interface=wlan0
    dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
    ```

    This sets up DHCP for the network, assigning IPs from `192.168.4.2` to `192.168.4.20`.
* Enable IP Forwarding and Set NAT  
Edit `sysctl.conf`:  

    ```
    $ sudo nano /etc/sysctl.conf
    ```

    Uncomment the following line by removing the `#`
    
    ```
    net.ipv4.ip_forward=1
    ```


* Add a NAT rule to allow outbound traffic:  
    
    ```
    $ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    $ sudo netfilter-persistent save
    ```

* Start Services  
    Enable and start the `hostapd` and `dnsmasq` services:
    
    ```
    $ sudo systemctl unmask hostapd
    $ sudo systemctl enable hostapd
    $ sudo systemctl start hostapd
    $ sudo systemctl restart dnsmasq
    ```

* Test the Configuration  
    At this point, your Raspberry Pi should be broadcasting a WiFi network as specified. Connect a device to this network using the SSID and password you set, and try accessing the web pages served by Apache via the Raspberry Pi's IP address (e.g., `http://192.168.4.1`).

# Assembling the Components

Once the Raspberry Pi has been set up with its software and tested, it only remains to connect that device to battery and solar power. The following order of connection is recommended based on documentation for the solar panel / controller.

1. Connect the battery terminals to the solar controller.
2. Connect the solar cell terminals to the solar controller.  
3. Connect the Raspberry Pi, either to a USB out from the solar controller or the 12-Volt Load terminals connected to a step-down "buck" converter.  
   The RPi will boot up and be available for access, both via SSH (zero@raspberrypi.local) and wifi (using the credentials configured above).
4. To disconnect:
   1. Connect to the RPi using SSH and issue a `sudo shutdown now` command. Wait a few moments until the green LED on the RPi has turned off.
   2. Disconnect the solar cell from the controller.
   3. Disconnect the battery from the controller.

It's possible to install switches for managing the solar and battery connections. It's also possible to program the RPi so that a switch connected to the General Purpose Input Output pins (GPIO) will run a script to initiate the shutdown process.

# References

* [https://learn.adafruit.com/digital-free-library/](https://learn.adafruit.com/digital-free-library/)
* [https://solar.lowtechmagazine.com/2023/12/how-to-build-a-small-solar-power-system/](https://solar.lowtechmagazine.com/2023/12/how-to-build-a-small-solar-power-system/)

