# dnsmasq-centos7

This repository is for those who are trying to setup a CentOS 7 machine as their corporate (or home) firewall.
In this tutorial I'm using the built-in Firewall instead of iptables because it is more convenient and it'll do for us now.
I also use dnsmasq for DHCP services because a lot of other features can be implemented if required.
In the beginning I'll just tell you the methods and the way of configuration. Some explaination will be at the end of the document.

---

## Requirenemtns

* CentOS 7
	* A TextEditor (nano, vim, mcedit, whatever...)
	* Ruby (<1.8)
	* FirewallD (32bit editions don't have it by default, no clue why)

## The setup

* Update server
	```
	$ sudo yum install epel-release
	$ sudo yum update -y
	$ sudo yum install wget vim mc ruby
	```

* Edit the network cards, in this case 3 of them. 1# External, 2# Internal, 3# DMZ (for WiFi guests)
	```
	$ sudo vim /etc/sysconfig/network-scrips/ifconfig-enp3s0
	```
	* Modify / Add the following lines to the config
	```
	BOOTPROTO=dhcp
	NM_CONTROLLED=no
	ONBOOT=yes
	ZONE=external
	```
	```
        $ sudo vim /etc/sysconfig/network-scrips/ifconfig-enp4s1
        ```
        * Modify / Add the following lines to the config
        ```
        BOOTPROTO=static
        IPADDR=10.0.0.1
	NETMASK=255.255.255.0
	NM_CONTROLLED=no
        ONBOOT=yes
        ZONE=internal
	
        ```
	```
        $ sudo vim /etc/sysconfig/network-scrips/ifconfig-enp4s1
        ```
        * Modify / Add the following lines to the config
        ```
        BOOTPROTO=static
        IPADDR=192.168.0.1
        NETMASK=255.255.255.0
        NM_CONTROLLED=no
        ONBOOT=yes
        ZONE=dmz

        ```

* Edit the dnsmasq configuration file
	```
	$ sudo vim /etc/dnsmasq.conf
	```
	* Modify / Add the following lines to the config
	```
	bogus-priv
	server=8.8.8.8
	local=/bap/
	interface=enp4s1,enp4s2
	no-hosts
	addn-hosts=/etc/dnsmasq.hosts.conf
	dhcp-hostsfile=/etc/dnsmasq.dhcp.conf
	log-facility=/var/log/dnsmasq.log
	expand-hosts
	dhcp-range=net:known,enp4s1,10.0.0.2,10.0.0.254,24h
	dhcp-range=enp4s2,192.168.1.150,192.168.100.200,1h
	log-dhcp
	conf-dir=/etc/dnsmasq.d
	```
* Create and alias for the converting the configuration files and restarting dnsmasq service
	```
	$ sudo chmod +x ./dnsmasq-convert (optionally if it is not already executeable)
	$ sudo cp ./dnsmasq-convert /usr/local/sbin/dnsmasq-converter
	$ sudo vim ~/.bashrc
	```
	Add the following line to the end of your file:
	```
	alias dnsrestart="dnsmasq-converter; systemctl restart dnsmasq"
	```
	Re-read your bashrc file
	```
	$ sudo source ~/.bashrc
	```
