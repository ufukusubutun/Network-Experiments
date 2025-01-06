# Building a Multi Queue Discipline 

Load Balancer implemented by forwarding packets to queues by filtering and marking.

WARNING: This one is not tested and is still in development. Commands contain errors and they are not in the correct order.


## Preliminary

Cloudlab line topology 

TODO add image.



at each node

	sudo apt-get update
	sudo apt -y install moreutils


## Instructions

At router: set the bottlencek

	IF=$(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+")
	IF_BACK=$(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+")

	sudo tc qdisc del dev $IF root  
	sudo tc qdisc add dev $IF root handle 1: htb default 3  
	sudo tc class add dev $IF parent 1:2 classid 1:3 htb rate 1Mbit  
	sudo tc qdisc add dev $IF parent 1:3 bfifo limit 0.2mbit 

	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 1Mbit  
	sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit 0.2mbit 

	#or
	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root netem delay 5ms 


At emulator:

	sudo tc qdisc del dev $IF root
	sudo tc qdisc add dev $IF root handle 1: htb default 3 
	sudo tc class add dev $IF parent 1: classid 1:1 htb rate 1Mbit quantum 1514
	sudo tc class add dev $IF parent 1:1 classid 1:2 htb rate 1Mbit quantum 1514 
	sudo tc class add dev $IF parent 1:1 classid 1:3 htb rate 1Mbit quantum 1514

	sudo tc qdisc add dev $IF parent 1:2 handle 2: netem delay 10ms
	sudo tc qdisc add dev $IF parent 1:3 handle 3: netem delay 5ms

	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 1Mbit  
	sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit 0.2mbit


for 10 mbit
At router: set the bottlencek

	sudo tc qdisc del dev $IF root  
	sudo tc qdisc add dev $IF root handle 1: htb default 3  
	sudo tc class add dev $IF parent 1:2 classid 1:3 htb rate 10Mbit  
	sudo tc qdisc add dev $IF parent 1:3 bfifo limit 0.2mbit 

	#sudo tc qdisc del dev $IF_BACK root  
	#sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	#sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 10Mbit  
	#sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit 0.2mbit 

	#or
	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root netem delay 5ms 

	tc -s -d class show dev $IF
	tc -s -d qdisc show dev $IF
	tc -s -d class show dev $IF_BACK
	tc -s -d qdisc show dev $IF_BACK


At emulator:

	sudo tc qdisc del dev $IF root
	sudo tc qdisc add dev $IF root handle 1: htb default 3 
	sudo tc class add dev $IF parent 1: classid 1:1 htb rate 10Mbit quantum 1514
	sudo tc class add dev $IF parent 1:1 classid 1:2 htb rate 10Mbit quantum 1514 
	sudo tc class add dev $IF parent 1:1 classid 1:3 htb rate 10Mbit quantum 1514

	sudo tc qdisc add dev $IF parent 1:2 handle 2: netem delay 10ms
	sudo tc qdisc add dev $IF parent 1:3 handle 3: netem delay 5ms

	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 10Mbit  
	sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit 0.2mbit


	tc -s -d class show dev $IF
	tc -s -d qdisc show dev $IF
	tc -s -d class show dev $IF_BACK
	tc -s -d qdisc show dev $IF_BACK

set the tagging and forwarding rules

	#flush first! 
	sudo iptables -t mangle -F
	#then set
	sudo iptables -A PREROUTING -m statistic --mode nth --every 10 --packet 0 -t mangle --destination 10.10.3.2/24 --source 10.10.1.1/1 -j MARK --set-mark 12
	sudo iptables -A PREROUTING -m mark --mark 12 -t mangle -j RETURN

	sudo tc filter add dev $IF protocol ip parent 1: prio 0 handle 12 fw classid 1:2


to display

	sudo iptables -L -n -t mangle
	tc -s -d filter show dev $IF

to clear

	sudo iptables -t mangle -F


###
####
##

	# for 10 mbit with larger rtt
	# much fewer number of packets reordered
	#At router: set the bottlencek

	sudo tc qdisc del dev $IF root  
	sudo tc qdisc add dev $IF root handle 1: htb default 3  
	sudo tc class add dev $IF parent 1:2 classid 1:3 htb rate 10Mbit  
	sudo tc qdisc add dev $IF parent 1:3 bfifo limit 0.8mbit 

	#sudo tc qdisc del dev $IF_BACK root  
	#sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	#sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 10Mbit  
	#sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit 0.8mbit 

	#or
	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root netem delay 10ms 

	tc -s -d class show dev $IF
	tc -s -d qdisc show dev $IF
	tc -s -d class show dev $IF_BACK
	tc -s -d qdisc show dev $IF_BACK


	#At emulator:

	sudo tc qdisc del dev $IF root
	sudo tc qdisc add dev $IF root handle 1: htb default 3 
	sudo tc class add dev $IF parent 1: classid 1:1 htb rate 10Mbit quantum 1514
	sudo tc class add dev $IF parent 1:1 classid 1:2 htb rate 10Mbit quantum 1514 
	sudo tc class add dev $IF parent 1:1 classid 1:3 htb rate 10Mbit quantum 1514

	sudo tc qdisc add dev $IF parent 1:2 handle 2: netem delay 20ms    #40ms
	sudo tc qdisc add dev $IF parent 1:3 handle 3: netem delay 10ms

	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 10Mbit  
	sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit 0.8mbit


	tc -s -d class show dev $IF
	tc -s -d qdisc show dev $IF
	tc -s -d class show dev $IF_BACK
	tc -s -d qdisc show dev $IF_BACK

	# set the tagging and forwarding rules

	#flush first! 
	sudo iptables -t mangle -F
	#then set
	sudo iptables -A PREROUTING -m statistic --mode nth --every 1000 --packet 0 -t mangle --destination 10.10.3.2/24 --source 10.10.1.1/1 -j MARK --set-mark 12
	sudo iptables -A PREROUTING -m mark --mark 12 -t mangle -j RETURN

	sudo tc filter add dev $IF protocol ip parent 1: prio 0 handle 12 fw classid 1:2


	# to display
	sudo iptables -L -n -t mangle
	tc -s -d filter show dev $IF

	#to clear
	sudo iptables -t mangle -F









10e6*100e-3*2 = 2mbit

	sudo apt-get update; sudo apt-get -y install moreutils


	sudo tc qdisc del dev eth1 root  
	sudo tc qdisc add dev eth1 root netem delay 100ms 


	sudo tc qdisc del dev eth2 root  
	sudo tc qdisc add dev eth2 root handle 1: htb default 3  
	sudo tc class add dev eth2 parent 1:2 classid 1:3 htb rate 10Mbit  
	sudo tc qdisc add dev eth2 parent 1:3 bfifo limit 2mbit 


	sudo tc qdisc add dev eth2 root netem delay 10ms reorder 90% 01%



	$(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+")

	$(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+")


	# At the router
	sudo tc qdisc del dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root netem delay 10ms reorder 90% 01%

	sudo tc qdisc change dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root netem delay 10ms reorder 99.9% 01%

	# At the emulator

	sudo tc qdisc del dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root handle 1: htb default 3  
	sudo tc class add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") parent 1:2 classid 1:3 htb rate 20Mbit  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") parent 1:3 bfifo limit 4mbit 

	sudo tc qdisc del dev $(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+") root netem delay 100ms 

	or


	sudo tc qdisc del dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root handle 1: htb default 3  
	sudo tc class add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") parent 1:2 classid 1:3 htb rate 10Mbit  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") parent 1:3 bfifo limit 2mbit 

	sudo tc qdisc del dev $(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+") root netem delay 100ms 

	# or

	sudo tc qdisc del dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") root handle 1: htb default 3  
	sudo tc class add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") parent 1:2 classid 1:3 htb rate 1Mbit  
	sudo tc qdisc add dev $(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+") parent 1:3 bfifo limit 0.2mbit 

	sudo tc qdisc del dev $(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+") root  
	sudo tc qdisc add dev $(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+") root netem delay 100ms 

