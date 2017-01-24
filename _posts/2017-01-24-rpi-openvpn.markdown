---
title: "Setting up Raspberry Pi 3 as an OpenVPN router"
layout: post
description: A guide to setting up Raspberry Pi 3 as an OpenVPN router
author: anirudh
blog: true
tag:
- LEDE
- OpenVPN
- Raspberry Pi
headerImage: false
---

## Introduction

Raspberry Pi 3 makes up a great miniature PC and can be used to come up with some great projects. But there are reasons why would you want to tone down on that versitality and use it exclusively for a particular function. The LEDE Project, based on OpenWRT, lets you set up your Raspberry Pi as a router. Add to this, OpenVPN and you have got yourself a nice box that lets you create an access point which routes your traffic through a VPN, thus securing your browsing. This could be really convenient to have all your personal devices establish a secure wireless connecting in places like hotels where you can simply plug in your RPi to ethernet port.

## Setting up Raspberry Pi

### Fetching the LEDE Image

We would be fetching the latest LEDE snapshot for Rapberry Pi 3 from the official LEDE source. Execute the following commands to download the image into your current directory.
{% highlight shell %}
wget https://downloads.lede-project.org/snapshots/targets/brcm2708/bcm2710/lede-brcm2708-bcm2710-rpi-3-ext4-sdcard.img.gz
{% endhighlight %}

### Flashing the LEDE Image

Uncompress the gzip archive to get the image file and install the image on a Micro SD Card. View instructions on [this link](https://www.raspberrypi.org/documentation/installation/installing-images/) in case you need help.

## Configuring LEDE

### Accessing Raspberry Pi

By default LEDE is configured to have a static IP and in order to access the RPi to change the configuration you would need to connect your computer directly to your RPi using an ethernet cable. Your computer would be assigned an IP and can now connect to the RPi using SSH. Log in to RPI using the following command.

{% highlight shell %}
ssh root@192.168.1.1
{% endhighlight %}

Now that we have access to the RPi filesystem, let's modify the network and firewall configurations.

### Modifying network configuration

The `network` configuration file is located in the `/etc/config/network`. We will modify the lan interface and add two new interfaces to achieve our goal, leaving the other interfaces unchanged.

{% highlight text %}
config interface 'lan'
    option ifname 'eth0'
    option proto 'dhcp'

config interface 'tun0'
    option ifname 'tun0'
    option proto 'none'

config interface 'wireless'
    option proto 'static'
    option ipaddr '10.0.0.1'
    option netmask '255.255.255.0'
    option ip6assign '60'
{% endhighlight %}

To use a static IP instead of using DHCP to obtain an IP from the router use the following configuration for the lan interface. Here is an example of assigning an IP `192.168.1.10` to the RPi for a router that provides IP's in the `192.168.1.x` range and has netmask `255.255.255.0`.

{% highlight text %}
config interface 'lan'
    option ifname 'eth0'
    option proto 'static'
    option ipaddr '192.168.1.10'
    option netmask '255.255.255.0'
{% endhighlight %}

### Modifying wireless configuration

The `wireless` configuration file is located in the `/etc/config/wireless`. The wireless adapter is disabled by default. Remove the line option disabled 1 or change 1 to 0 to enable the wireless adapter. Modify the `default_radio0` wireless interface to the following (replace `your_ssid` and `your_password` with the name and the password you would like to give your wireless access point).

{% highlight text %}
config wifi-iface 'default_radio0'
    option device 'radio0'
    option mode 'ap'
    option encryption 'psk2'
    option key 'your_password'
    option ssid 'your_ssid'
    option network 'wireless'
{% endhighlight %}


### Modifying dhcp configuration

The `wireless` configuration file is located in the `/etc/config/dhcp`. We will add another dhcp configuration for the `wireless` interface we defined in the `networks` file so that client which connect to the AP are assigned IPs using DHCP.

{% highlight text %}
config dhcp 'wireless'                           
    option interface 'wireless'              
    option start '100'                       
    option limit '150'                       
    option leasetime '12h'                   
    option dhcpv6 'server'                   
    option ra 'server'
{% endhighlight %}

### Modifying firewall configuration

The `firewall` configuration file is located in the `/etc/config/firewall`. We will first define a zone with input, output and forward rules for the `wireless` interface we defined in the `networks` file.

{% highlight text %}
config zone                      
    option name             wireless  
    list network            'wireless'
    option input            ACCEPT
    option output           ACCEPT      
    option forward          ACCEPT
{% endhighlight %}

We will add a similar zone for the tunnel interface using by the VPN.

{% highlight text %}
config zone                                
    option name             tun0       
    list network            'tun0'    
    option input            REJECT          
    option output           ACCEPT    
    option forward          REJECT    
    option masq             1
{% endhighlight %}

The last thing we need to add to configuration file is forwarding packets from wireless client to the VPN.

{% highlight text %}
config forwarding                                      
    option src              lan                
    option dest             wan
{% endhighlight %}

### Reflecting changes to the configuration files

Execute the following commands on the RPi.

{% highlight shell %}
/etc/init.d/firewall restart   
wifi up
/etc/init.d/network restart
{% endhighlight %}

### Connecting Raspberry Pi to the internet

We can now unplug the RPi from the computer and connect it to the LAN port of a router. If you chose `dhcp` for the `lan` interface, for your RPi to get an IP, DHCP should be enabled on the router. Check your router web interface to obtain the IP assigned to your RPi. If the RPi is assigned the IP 192.168.1.10 for instance, execute the following command on your computer, connected to the same router, to log back into your router.

{% highlight text %}
ssh root@192.168.1.10
{% endhighlight %}

### Updating packages and installing LuCI

Once connected to the router, your RPi should be able to access the internet. Update the packages and install LuCI for web configuration by executing the following commands. You can now access your RPi's configuration, similar to a conventional router, by keying in its IP.

{% highlight shell %}
opkg update
opkg install luci
{% endhighlight %}

## Setting up OpenVPN

### Installing OpenVPN

{% highlight shell %}
opkg install openvpn-openssl
{% endhighlight %}

### Transferring .ovpn VPN configuration

Transfer the .ovpn configuration for VPN to your RPi using the following command, replacing vpn_config_file.ovpn is your configuration file and 192.168.1.10 by your RPi's IP.

{% highlight shell %}
scp vpn_config_file.ovpn root@192.168.1.10:~
{% endhighlight %}

### Modifying OpenVPN configuration

The OpenVPN configuration is located at `/etc/config/openvpn`. Modify the `custom_config` configuration to the following

{% highlight text %}
config openvpn custom_config

    # Set to 1 to enable this instance:
    option enabled 1

    # Include OpenVPN configuration
    option config vpn_config_file.ovpn
{% endhighlight %}

### Enabling autostart and VPN

Now that all required configuration in place, we can enable autostart at boot. Finally we enable OpenVPN to get it running right away.

{% highlight shell %}
/etc/init.d/openvpn enable
/etc/init.d/openvpn start
{% endhighlight %}

## Wrapping it up

The Raspberry Pi is now running as a router with OpenVPN. If you set it up to use DHCP for the ethernet connection and enabled autostart at boot you now have a nifty plug and play box which you can connect to an ethernet port to gain secure internet access through VPN, on your wireless devices. 
