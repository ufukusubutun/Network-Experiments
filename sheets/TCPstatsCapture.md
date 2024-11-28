# TCP Stats Capture

Capture detailed TCP statistics thru trace-cmd


 become root
	sudo su

 one time only
	apt install trace-cmd

 repeat on each reboot
	echo 1 > /sys/kernel/debug/tracing/events/tcp/tcp_probe/enable

 before each connection
	trace-cmd record --date -e tcp_probe

 after flow ends, use Ctrl+C to stop recording
 then play back with
	trace-cmd report

 to write to a file 
	trace-cmd report > expname-log.txt


Parser and plotting examples can be found in:

TODO clean up and upload jupyter notebook.

