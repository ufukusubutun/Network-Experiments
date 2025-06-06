Getting hardware timestamps at ns resolution when capturing packets with tcpump.

Tested on cloudlab, with c6620 nodes running Ubuntu 24. (older ubuntu versions do not have the necesary NIC driver installled by default.)


To be cleaned up. (there are some irrelevant tcp port filters )

As I am working on Cloudlab testbed with virtual interfaces, I do the following trick to capture the packets from the physical interface corresponding to my virtual interface and then filter packets during capture to only contain those from the desired virtual interface.

      vlan_iface=$(ip route get 10.14.1.2 | grep -oP "(?<= dev )[^ ]+")
      vlan_id=$(echo "$vlan_iface" | grep -oP '\d+$')
      interface=$(ip -d link show "$vlan_iface" | sed -n 's/.*@\(.*\):.*/\1/p')
      
      sudo tcpdump -B 4096 -n -i "$interface" vlan "$vlan_id" and tcp dst portrange 50000-59600 -j adapter_unsynced --time-stamp-precision=nano -s 64 -w egress_cap1.pcap

