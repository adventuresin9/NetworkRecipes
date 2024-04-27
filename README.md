# NetworkRecipes

## Introduction

Many of these examples are assuming a dedicated gateway machine.  This could be your grid's file server, or it could be a dedicated gateway box that has extra Ethernet adapters installed and booting as a diskless cpu server.

Some of these examples make use of the /cfg directory.  This deirectory is checked by the default configuration script in /rc/bin/cpurc for machine specific configuration.  In many of these examples, *gate* is used as the name of the gateway machine.  To add additional configuration files for your gateway machine, add a directory with the same name as your gateway machine.  Into that directory, create files and be sure to make them executable.  These files are not found in /cfg by default, and must be added.  Other options include;

+ cpurc

	This is ran fairly early from /rc/bin/cpurc and is useful to set up things that will be called from later parts of the default configuration.

+ cpustart

	This is called at the end of the default cpurc, so this is used to do more configuration after all the defaults have been done.

+ namespace

	This is used to add additional things to the default namespace on this machine.  It is called at the end of the default namespace recipe found in /lib/namespace.  

+ service/

	This is a directory that functions like /rc/bin/service/.  The files must be named in a specific way for the system to run listen(8) at the specific port and protocol.  Keep in mind that this service directory will be ran in place of, and _not_ in addition to, the default /rc/bin/service/ scripts.

### Warning

Another thing to be cafeful of, which depends on the font being used to render this text, is the distinction between the kernel devices '#l' and '#I'.  The first is a lower case L, and those are for the Ethernet interface, see; ether(3).  The second is a upper case i, and that is for the IP stack interface, see; ip(3).  Some fonts make it hard to tell the two letter apart.  Additionally, be aware of the TLS device '#a', which may need to be included in any custom /net directory for TLS connections to function properly; see tls(3).  

There are also Connection Service and DNS, which are usually posted to /srv and mounted from there, see; ndb(8).  In some of these examples, it will show running the 'nbd/cs' and 'nbd/dns' commands for configuration.  If CS an DNS have already been setup for a given side of a gateway, you may be able to just run 'mount -a /srv/cs /net.alt' and 'mount -a /srv/dns /net.alt'.  If CS and DNS aren't configured properly or mounted properly, 'ip/ping' by sysname or domain name will fail.  In cases where the -x flag is used, like 'ndb/cd -x net.alt -f /lib/outside', the /srv post produced will be '/srv/dns_net.alt'.

## Plan9 style imported interface:

This is the classic idea of importing an outside facing networking interface into the namespace of a process that needs access to the outside world.  In this scenario you would have a closed network for the 9Front machines.  One of the machines, *gate*, has 2 Ethernet ports and is running as a cpu server.  

+ ether0 = inside network
+ ether1 = outside network

This assumes ether0 is the default network interace configured at boot.  The second Ethernet port (#l1) and a second IP stack (#I1) will be added to /net.alt.  It will then be configured via DHCP by what ever assigns addresses on the outside network.  Next, add Connection Service (ndb/cs) and DNS (ndb/dns), using the -x flag to specify that this is a network stack in /net.alt, rather than the default /net (see; ndb(8)).  Finally, srvfs is used to package all this up and place it in /srv so that it can be mounted into a namespace.

The following demonstrates this as a script placed in /cfg/gate, and will be ran from /cfg/gate/cpustart.  To make it so that this outside network stack always appears in /net.alt on *gate*, the a mount is included in /cfg/gate/namespace.

### /cfg/gate/mknet1

	#!/bin/rc
	rfork
	bind '#l1' /net.alt
	bind -a '#I1' /net.alt
	ip/ipconfig -x /net.alt ether /net.alt/ether1
	ndb/cs -x /net.alt
	ndb/dns -x /net.alt
	srvfs -p 666 outside.net /net.alt

### /cfg/gate/cpustart

	#!/bin/rc
	/cfg/gate/mknet4

### /cfg/gate/naemspace

	mount -ac /srv/net1 /net.alt


## Moody's NAT:

This is taken from instructions found here:  https://posixcafe.org/blogs/2024/01/04/0/

In this case, ether0 (#l0) is assumed to be configured at boot to be the standard cpu server interface, where the machine can be called with rcpu(1).

This will setup ether1 (#l1) as the inside port to be used as the gateway.  The inside network is using 198.168.2.0, and the gateway port will be set to 192.168.2.1.  The outside interface will be ether2 (#l2), and will use DHCP to be configured by the outside network.  Each interface will use it's own IP stack (#I1 and #I2).

In this setup, the configuration is done in a script called *mknat* and will be called by /cfg/gate/cpustart.

### /cfg/gate/mknat

	#!/bin/rc
	rfork
	# /net set up for inside connection
	bind '#I1' /net
	bind -a '#l1' /net

	# /net.alt setup for outside connection
	bind '#I2' /net.alt
	bind -a '#l2' /net.alt

	x=/net.alt
	o=/net
	<$x/ipifc/clone {
		# Read the new interface number
		xi=$x/ipifc/`{read}

		# Write in to the ctl file of the newly created interface to configure it
		>$xi/ctl {
			# This is a packet interface
			echo bind pkt

			# Our ip is 192.168.69.3/24 and we only allow remote connections from 192.168.69.2
			echo add 192.168.69.3 255.255.255.255 192.168.69.2

			# Route packets to others
			echo iprouting 1

			# Now create a new interface on the inside IP stack
			<$o/ipifc/clone {
				oi=$o/ipifc/`{read}
				>$oi/ctl {
					# Hook up this device to the outside IP stack device
					echo bind netdev $xi/data

					# Our ip is 192.168.69.2/24 and we only allow remote connections from 192.168.69.3
					echo add 192.168.69.2 255.255.255.0 192.168.69.3
					echo iprouting 1
				}
			}
		}
	}

	# Configure our route table for both the inside and outside IP stacks
	# Arguments are: target mask nexthop interface(addressed by IP)
	echo add 192.168.2.0 255.255.255.0 192.168.69.2 192.168.69.3 > $x/iproute
	echo add 0.0.0.0 /96 192.168.69.3 192.168.69.2 > $o/iproute

	# Do DHCP on the external interface. -x tells us which
	# IP stack to use. -t tells us that we are doing NAT
	# and to configure the route table as such. NAT is implemented
 	# as just a route table flag.
	ip/ipconfig -x /net.alt -t ether /net.alt/ether2

	# Configure a static IP on our internal interface, which will
	# act as a gateway for our internal network.
	ip/ipconfig ether /net/ether1 192.168.2.1 255.255.255.0

	# Package up these namespaces 
	srvfs -p 644 nat-out /net.alt
	srvfs -p 644 nat-in /net

This script can then be set to run automaticaly at boot.

### /cfg/gate/cpustart

	#!/bin/rc
	/cfg/gate/mknat

An entry for the gateway as ipgw will need to be added to /lib/ndb/local so that the other systems inside the grid know where the gateway is and to automatically forward outbound traffic there.

### /lib/ndb/local

	...
	ipgw=192.168.2.1
	...



## Basic Tunnel:

This is taken from:  https://9lab.org/plan9/tunnel/

This could be used to create a connection between your home 9Front system and a VPS running 9Front.  This has the upside of being both very simple to do, and is initiated by your local system, rather than something running on the remote system that could be exploited.

From the home system, run the command;

    inside% rexport -s inside.9lab.home / outside.9lab.org &

This will send a /srv post to outside.9lab.org, which will appear there as /srv/inside.9lab.home .  In this case it is sending everything from the root directory on down.  So the entire namespace that rexport was ran in.  

On the remote system, run the command;

    outside% mount /srv/inside.9lab.home /n/inside.9lab.home

Running this on the remote system would mean that everything in / on the local system can now be read from /n/inside.9lab.home/ .  

The local system's /net can then be mounted and accessed in the remote system, allowing for use of rcpu.

    outside% bind /n/inside.9lab.home/net /net.alt
    outside% rcpu -h /net.alt/tcp!inside.9lab.home



## Fancy Tunnel:

In this case a virtual Ethernet interface (#l5) will be created and bridged to an unused physical Ethernet interface (#l3).  That virtual interface can then be sent via rexport to a remote system and mounted there.  The remote machine will then be directly accessable through 10.0.0.99 on the local grid.  On this machine, there is 1 on board Ethernet port for ether0, and a 4 port Ethernet PCIe card for ether1-4.  So ether5 would be the first '#l' that wouldn't be associated whith a physical Ethernet device.

First, set up the unused ether3 (#l3) as the trunk end of of a bridge (#B1).

### /cfg/gate/mknet3

	#!/bin/rc
	rfork
	bind '#B1' /net
	bind -a '#l3' /net
	echo 'bind ether trunk 0 /net/ether3' >/net/bridge1/ctl
	srvfs -p 666 net4 /net

Next, configure ether5 (#l5) as a virtual interface (sink) and give it a MAC address (cafe42000005).  Then hook is at port1 on the same bridge (#B1).  Add an IP stack (#I5), run ip/ipconfig to give it address 10.0.0.99.  CS and DNS can be set up by just using the existing inside grid ones mounted from /srv/cs and /srv/dns.  Finally, post this setup to /srv.

### /cfg/gate/mknet5

	#!/bin/rc
	rfork
	bind -a '#B1' /net
	bind '#l5:sink ea=cafe42000005' /net.alt
	echo 'bind ether port1 0 /net.alt/ether5' >/net/bridge1/ctl
	bind -a '#I5' /net.alt
	ip/ipconfig -x /net.alt -g 10.0.0.1 ether /net.alt/ether5 10.0.0.99 255.255.255.0
	mount -a /srv/cs /net.alt
	mount -a /srv/dns /net.alt
	srvfs -p 666 net5 /net.alt

On the inside machine, run;

	inside% mount /srv/net5 /n/net5
	inside% rexport -s net5 /n/net5 outside.com

On the outside machine, run;

	outside% mount /srv/net5 /n/net5
	outside% bind -b /n/net5 /net


