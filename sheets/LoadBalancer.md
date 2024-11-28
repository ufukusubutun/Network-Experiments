# Building a Simple Load Balancer

Load Balancer implemented by forwarding packets to interfaces by filtering and marking.


Contains both a) per-packet load balancer and b) per-flow (hashed) load balancer implemented with iptables.


## Preliminary

Cloudlab line topology with a parallel link in the middle

TODO add image.

https://www.cloudlab.us/show-profile.php?uuid=647c9b55-a75f-11ef-828b-e4434b2381fc


at each node

	sudo apt-get update
	sudo apt -y install moreutils


## Instructions

At client: set the fwd bottlencek

	IF=$(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+")

	sudo tc qdisc del dev $IF root  
	sudo tc qdisc add dev $IF root handle 1: htb default 3  
	sudo tc class add dev $IF parent 1:2 classid 1:3 htb rate 130Mbit  
	sudo tc qdisc add dev $IF parent 1:3 bfifo limit  26mbit 


At router: set the bottlencek

	IF_P1=$(ifconfig | grep -B 1 "10.10.2.1 " | awk 'NR % 2 == 1' | awk '{print $1}' |  tr -d ':')
	IF_P2=$(ifconfig | grep -B 1 "10.10.2.3 " | awk 'NR % 2 == 1' | awk '{print $1}' |  tr -d ':')
	IF_BACK=$(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+")
	sudo ip route del $(ip route | grep 10.0.0.0)
	sudo ip route add 10.10.3.0/24 via 10.10.2.2 dev $IF_P1
	sudo ip route add 10.10.3.0/24 via 10.10.2.4 dev $IF_P2 table 100
	sudo ip rule add fwmark 3 table 100 

	sudo tc qdisc del dev $IF_P1 root  
	sudo tc qdisc add dev $IF_P1 root handle 1: htb default 3  
	sudo tc class add dev $IF_P1 parent 1:2 classid 1:3 htb rate 50Mbit quantum 1514
	sudo tc qdisc add dev $IF_P1 parent 1:3 bfifo limit 1mbit 	
	# was testing reordering at unequal paths - to also do so, ignore the line above and instead run the two lines below
	#sudo tc qdisc add dev $IF_P1 parent 1:3 handle 3: netem delay 7ms #3ms
	#sudo tc qdisc add dev $IF_P1 parent 3: bfifo limit 1mbit # 20ms fully backed at 50Mbps


	sudo tc qdisc del dev $IF_P2 root  
	sudo tc qdisc add dev $IF_P2 root handle 1: htb default 3  
	sudo tc class add dev $IF_P2 parent 1:2 classid 1:3 htb rate 50Mbit quantum 1514 
	sudo tc qdisc add dev $IF_P2 parent 1:3 bfifo limit 1mbit 

	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 130Mbit  
	sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit  2.6mbit 

	tc -s -d qdisc show dev $IF_P1
	tc -s -d qdisc show dev $IF_P2


At aggr: add delay

	IF=$(ip route get 10.10.3.2 | grep -oP "(?<= dev )[^ ]+")

	sudo tc qdisc del dev $IF root  
	sudo tc qdisc add dev $IF root netem delay 5ms

At aggr: add reverse btlnck

	IF_P1_BACK=$(ifconfig | grep -B 1 "10.10.2.2 " | awk 'NR % 2 == 1' | awk '{print $1}' |  tr -d ':')
	IF_P2_BACK=$(ifconfig | grep -B 1 "10.10.2.4 " | awk 'NR % 2 == 1' | awk '{print $1}' |  tr -d ':')
	sudo ip route del $(ip route | grep 10.0.0.0)
	sudo ip route add 10.10.1.0/24 via 10.10.2.1 dev $IF_P1_BACK
	sudo ip route add 10.10.1.0/24 via 10.10.2.3 dev $IF_P2_BACK table 100
	sudo ip rule add fwmark 3 table 100 

	sudo tc qdisc del dev $IF_P1_BACK root  
	sudo tc qdisc add dev $IF_P1_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_P1_BACK parent 1:2 classid 1:3 htb rate 50Mbit  
	sudo tc qdisc add dev $IF_P1_BACK parent 1:3 bfifo limit 10mbit 

	sudo tc qdisc del dev $IF_P2_BACK root  
	sudo tc qdisc add dev $IF_P2_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_P2_BACK parent 1:2 classid 1:3 htb rate 50Mbit  
	sudo tc qdisc add dev $IF_P2_BACK parent 1:3 bfifo limit 10mbit 



At server: add reverse btlnck

	IF_BACK=$(ip route get 10.10.1.1 | grep -oP "(?<= dev )[^ ]+")

	sudo tc qdisc del dev $IF_BACK root  
	sudo tc qdisc add dev $IF_BACK root handle 1: htb default 3  
	sudo tc class add dev $IF_BACK parent 1:2 classid 1:3 htb rate 130Mbit  
	sudo tc qdisc add dev $IF_BACK parent 1:3 bfifo limit  26mbit 




At router
	
	sudo ip rule add fwmark 3 table 100 



### for per-packet load balancer

At router: 
set the tagging and forwarding rules for per packet forwarding

	#flush first! 
	sudo iptables -t mangle -F
	#then set
	sudo iptables -A PREROUTING -m statistic --mode nth --every 2 --packet 0 -t mangle --destination 10.10.3.2/24 --source 10.10.1.1/1 -j MARK --set-mark 3
	sudo iptables -A PREROUTING -m mark --mark 3 -t mangle -j RETURN


to display
	
	sudo iptables -L -n -t mangle

### for per-flow load balancer

At router: 
set the tagging and forwarding rules for flow hashing

	#flush first! 
	sudo iptables -t mangle -F
	# Mark all flows to 2
	sudo iptables -t mangle -A PREROUTING -i $IF_BACK -m conntrack --ctstate NEW -j CONNMARK --set-mark 2
	# Mark half of the flows to 1
	sudo iptables -t mangle -A PREROUTING -i $IF_BACK -m conntrack --ctstate NEW -m statistic --mode random --probability 0.5 -j CONNMARK --set-mark 1
	# restore and overwrite (?) the mark of previously marked flows
	sudo iptables -t mangle -A PREROUTING -i $IF_BACK -m conntrack --ctstate ESTABLISHED,RELATED -j CONNMARK --restore-mark

	# Mark packets 3 if they belong to flows marked 2
	sudo iptables -t mangle -A PREROUTING -m connmark --mark 1 -j MARK --set-mark 3
	# if a packet is marked with 3 return
	sudo iptables -A PREROUTING -m mark --mark 3 -t mangle -j RETURN

	# not used in this example - you can mark more packets with different marks as follows 
	#sudo iptables -t mangle -A PREROUTING -m connmark --mark 2 -j MARK --set-mark 4
	#sudo iptables -A PREROUTING -m mark --mark 4 -t mangle -j RETURN

to display

	sudo iptables -L -n -t mangle








