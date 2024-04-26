# NetworkRecipes


## Plan9 style imported interface:

+ outside grid 192.168.1.0
+ inside grid = 192.168.2.0

+ sys=gate
+ ether0 is inside
+ ether1 is outside


/cfg/gate/mknet1

	#!/bin/rc
	rfork
	bind '#l1' /net.alt
	bind -a '#I1' /net.alt
	ip/ipconfig -x /net.alt ether /net.alt/ether4
	ndb/cs -x /net.alt
	ndb/dns -x /net.alt
	srvfs -p 666 net1 /net.alt

/cfg/gate/cpustart

	#!/bin/rc
	/cfg/gate/mknet4

/cfg/gate/naemspace

	mount -ac /srv/net1 /net.alt


## Moody's NAT:

outside grid = 192.168.1.0
inside grid = 192.168.2.0

sys=gate
ether0 = cpu access from inside grid
ether1 = ipgw for inside grid
ether2 = outside world

/lib/ndb/local
	...
	ipgw=192.168.2.1
	...


/cfg/gate/mknat

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




## Basic Tunnel:

inside% rexport -s inside.9lab.home / outside.9lab.org &

outside% mount /srv/inside.9lab.home /n/inside.9lab.home

outside% bind /n/inside.9lab.home/net /net.alt
outside% rcpu -h /net.alt/tcp!inside.9lab.home



## Fancy Tunnel:

/cfg/gate/mknet3

	#!/bin/rc
	rfork
	bind '#B1' /net
	bind -a '#l3' /net
	echo 'bind ether trunk 0 /net/ether3' >/net/bridge1/ctl
	srvfs -p 666 net4 /net


/cfg/gate/mknet5

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


inside% mount /srv/net5 /n/net5
inside% rexport -s net5 /n/net5 outside.com
inside% 

outside% mount /srv/net5 /n/net5
outside% bind -b /n/net5 /net


