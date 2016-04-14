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
	$ sudo yum update -y
	$ sudo yum install wget vim mc ruby firewalld
	```

* Edit the network cards, in this case 3 of them. 1# External, 2# Internal, 3# DMZ (for WiFi guests)
	```
	$ sudo vim /etc/sysconfig/network-scrips/ifconfig-enp3s0
	```
	* Modify or Add the following lines to the config
	```
	BOOTPROTO=dhcp
	NM_CONTROLLED=no
	ONBOOT=yes
	ZONE=external
	```
	```
	$ sudo vim /etc/sysconfig/network-scrips/ifconfig-enp4s1
	```
	* Modify or Add the following lines to the config
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
	* Modify or Add the following lines to the config
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
	* Modify or Add the following lines to the config
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

* Setup the firewall
	```
	$ sudo systemctl start firewalld
	$ sudo firewall-cmd --list-all-zones
	$ sudo firewall-cmd --zone=external --change-interface=enp3s0 (only if it is not there already)
	$ sudo firewall-cmd --zone=internal --change-interface=enp4s1 (only if it is not there already)
	$ sudo firewall-cmd --zone=dmz --change-interface=enp4s2 (only if it is not there already)
	$ sudo firewall-cmd --zone=internal --add-service=dhcp
	$ sudo firewall-cmd --zone=internal --add-service=dns
	$ sudo firewall-cmd --zone=dmz --add-service=dhcp
	$ sudo firewall-cmd --runtime-to-permanent
	```
	
* In case you need PPPOE
	```
	$ sudo yum install rp-pppoe
	$ sudo pppoe-setup
	<FOLLOW STEPS>
	```
	* Add ```ZONE=external``` to yout ppp0 interface
	```
	$ sudo vim /etc/sysconfig/network-scripts/ifcfg-ppp0
	$ sudo reboot
	$ sudo ifup ppp0
	```
	
## Make it easy

* Create an alias for converting the configuration files and restarting dnsmasq
	```
	$ sudo cp ./dnsmasq-convert /usr/local/sbin/dnsmasq-convert
	
	$ sudo vim ~/.bashrc
	```
	* Add the following line to the end of your .bashrc
	```
	alias dnsrestart="/usr/local/sbin/dnsmasq-convert; systemctl restart dnsmasq"
	```
	* Re-Read the .bashrc file
	```
	$ sudo ~/.bashrc
	```
## Q&A

* Why firewalld and not iptables?
	* Because RedHat didn't make firewalld to use iptables. Firewalld will be the future

* When I try to start firewalld I get the following error: FATAL ERROR: No IPv4 and IPv6 firewall.
	* There could be two options. One, you don't have iptables installed. This is unlikely. The other is that there is a problem with your kernel version installed. Run a yum upgrade and reboot the machine. This worked for me tho, I still don't get what is the problem, but it appeared on very old computers for me.

* In the dnsmasq.conf you have 2 dhcp-ranges. One with ```net:known```, the other is without it. Why?
	* In my case we have 2 WiFi networks with 2 routers. One for the internal network, the other one is a guest WiFi. I've set the routers to be only Acces Points because it would be better if the firewall handles the DHCP requests and not them. This way I don't need to forward traffic from the routers network to the internal, etc. But the real reason is simple. Imagine a guest WiFi where you can join only if you have your MAC address in the system already? Sounds stupid right? "Anyone" should be able to connect. The ```net:known``` parameter tells the dhcp-range to let only the known devices to connect. Everyone else will not get any IP address. For the Guest WiFi I've disabled this, so all my guests can access to the internet.

* What is that ZONE=external, internal and dmz in the network interfaces cfg file?
	* Firewalld can configures the network interfaces directly. If you enter the ZONE variable you can tell the interface which zone it belongs to. This has to be done because of a BUG, you can't save the firewall-cmd configuration with --runtime-to-permanent. 

* Why the 3 internet zones?
	* Simple: External handles the internet side. If you have PPPOE connection, you have to add the ppp0 device to this zone as well. Otherwise nobody will have iinternet, only the firewall. Internal is for the office use. DMZ is for the guests so they can access the internet without connecting to my internal network. Firewall handles and masqurades the connections already so you don't have to worry about this. But only if you've used the internal, external and dmz zones. In other cases you have to set the masqurade up yourself but it isn't that hard.

* Is this all I have to do? What about FTP server, git server, etc?
	* OCF not. I also recommend generating an SSH keypair for the user who will log in the computer and deny passwordlogin in the sshd_config. Running anything other on the firewall than the DHCP and DNS services would be very silly, so get another computer for that.
