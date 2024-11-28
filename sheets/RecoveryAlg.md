# Set Recovery Algorithm


## Algorithms


### RACK - RFC 8985 (should be the default in newer Linux kernels)

	sudo sysctl -w net.ipv4.tcp_recovery=1 net.ipv4.tcp_max_reordering=300 net.ipv4.tcp_sack=1

### Adaptive dupthresh - RFC 6675 +  linux adaptive threhsold heuristic

Threshold is heuristically grown up to (in this case) `300`

	sudo sysctl -w net.ipv4.tcp_recovery=0 net.ipv4.tcp_max_reordering=300 net.ipv4.tcp_sack=1

### 3-dupack

	sudo sysctl -w net.ipv4.tcp_recovery=0 net.ipv4.tcp_max_reordering=3 net.ipv4.tcp_sack=1




Setting to enable Duplicate Selective ACK. (should be turned on by default)

	sudo sysctl -w net.ipv4.tcp_dsack=1 

Setting not to save recently used cwnd etc. Useful for experiments

	sudo sysctl -w net.ipv4.tcp_no_metrics_save=1