This experiment shows the basic behavior of TCP congestion control. You'll see the classic "sawtooth" pattern in a TCP flow's congestion window, and you'll see how a TCP flow responds to indications of congestion.

It should take about 1 hour to run this experiment.

To reproduce this experiment on GENI, you will need an account on the [GENI Portal](http://groups.geni.net/geni/wiki/SignMeUp), and you will need to have [joined a project](http://groups.geni.net/geni/wiki/JoinAProject). You should have already [uploaded your SSH keys to the portal and know how to log in to a node with those keys](http://groups.geni.net/geni/wiki/HowTo/LoginToNodes). If you're not sure if you have those skills, you may want to try [Lab Zero](http://tinyurl.com/geni-labzero) first.

* Skip to [Results](#results)
* Skip to [Run my experiment](#runmyexperiment)


## Results

In this experiment, we see the classic "sawtooth" pattern of the TCP congestion window, shown as the solid line in the plot below:

![](/blog/content/images/2017/04/tcp-cwnd-2.svg)
<small><i>Figure 1: Congestion window size (solid line) and slow start threshold (dotted line) of three TCP flows sharing the same bottleneck.</i></small>

The slow start threshold is shown as a dashed line.

We can also identify

* "Slow start" periods
* "Congestion avoidance" periods (when the congestion window is greater than the slow start threshold)
* Instances where 3 duplicate ACKs were received. (We are using [TCP Reno](http://intronetworks.cs.luc.edu/current/html/reno.html), which will enter "fast recovery" in response to 3 duplicate ACKs.)
* Instances of timeout. This will cause the congestion window to go back to 1 MSS, and trigger a "slow start" period.

For example, the following annotated image shows a short interval in the top TCP flow of Figure 1:

![](/blog/content/images/2017/04/tcp-one.svg)
<small><i>Figure 2: Events in a TCP flow. Slow-start periods are marked in yellow; congestion avoidance periods are in purple. We can also see when indicators of congestion occur: instances of timeout (red), and instances where three duplicate ACKs are received (blue) triggering fast recovery (green).</i></small>

And we can see how the TCP congestion control algorithm responds to indications of packet loss (timeout and duplicate ACKs). 



## Run my experiment

First, reserve a topology on GENI including two end hosts, and a router between them. The router will buffer traffic between the sender and the receiver. If the buffer in the router becomes full, it will drop packets, triggering TCP congestion control behavior.

In the GENI Portal, create a new slice, then click "Add Resources". Scroll down to where it says "Choose RSpec" and select the "URL" option, the load the RSpec from the URL: [https://git.io/vSioM](https://git.io/vSioM)

This will load the following topology in your canvas:

![](/blog/content/images/2017/04/tcp-topology.svg)

Click on "Site 1" and choose an InstaGENI site to bind to, then reserve your resources. Wait for your nodes to boot up (they will turn green in the canvas display on your slice page in the GENI portal when they are ready). Then, SSH into each node. 

### Set up experiment


On the end hosts ("sender" and "receiver"), install the `iperf` network testing tool, with the command 

```
sudo apt-get update
sudo apt-get -y install iperf3
```
On the sender host, install moreutils

```
sudo apt-get -y install moreutils
```

Also set TCP Reno as the default TCP congestion control algorithm on both, with

```
sudo sysctl -w net.ipv4.tcp_congestion_control=reno
```

On the router, turn on packet forwarding with the command

```
sudo sysctl -w net.ipv4.ip_forward=1
```

At the router, use htb to configure the router to act as a 1 Mbps bottleneck with a buffer size of 0.1 MB : 

1. Delete any pre-existing queues on the experiment interface 

   ```
   sudo tc qdisc del dev eth2 root
   ```
   If after this line, you get an output saying something like 
   
   ```
   RTNETLINK answers : No such file or directory
   ```
   this is completely normal. Proceed to the next steps.


2. Then run the following three commands, in the same order 

   ```
   sudo tc qdisc replace dev eth2 root handle 1: htb default 3

   sudo tc class add dev eth2 parent 1: classid 1:3 htb rate 1Mbit

   sudo tc qdisc add dev eth2 parent 1:3 handle 3: bfifo limit 0.1MB
   ```
   
   NOTE : Replace 10.10.2.2 with the IP address of the receiver node.

3. Repeat steps 1 and 2 but for the second experiment interface on the router e.g. Instead of `ip route get eth2`, do `ip route get eth1`

### Getting familiar with ss

`ss` is a linux utility used to monitor sockets and display socket statistics. In this lab, we will use `ss` to monitor the TCP sockets being used in our experiment and display useful TCP congestion control information like congestion window (CWND), round-trip time (RTT) and slow start threshold (ssthresh).

To get familiar with `ss`, it is strongly suggested that you go through the `man` page of ss by typing `man ss` in the terminal window on any of the nodes. Also, complete the following set of steps. You will need two terminals open for the "sender" and one for the "receiver" node.

On the "receiver", start an `iperf3` server 

```
iperf3 -s  -1
```
On the "sender", start one TCP flow going from the "sender" to the "receiver" using iperf3

```
iperf3 -c receiver -i 0 -t 10
```
This `iperf3` transfer will run for 10 seconds. Within this period (while `iperf3` is sending traffic), use another shell on the "sender" to run the following `ss` command

```
ss --no-header -eipn dst 10.10.2.2
```
Your output should look something like this

```
tcp           ESTAB            0                 787712                         10.10.1.2:55058                         10.10.2.2:5201             users:(("iperf3",pid=7201,fd=4)) timer:(on,496ms,0) uid:20003 ino:83439 sk:53 <->
	 ts sack reno wscale:7,7 rto:1236 rtt:636.44/98.934 mss:1448 pmtu:1500 rcvmss:536 advmss:1448 cwnd:142 bytes_acked:191174 segs_out:277 segs_in:59 data_segs_out:275 send 2.6Mbps lastsnd:16 lastrcv:1620 lastack:16 pacing_rate 5.2Mbps delivery_rate 956.8Kbps busy:1580ms rwnd_limited:676ms(42.8%) unacked:142 rcv_space:14480 rcv_ssthresh:64088 notsent:582096 minrtt:0.655
tcp           ESTAB            0                 0                              10.10.1.2:55056                         10.10.2.2:5201             users:(("iperf3",pid=7201,fd=3)) uid:20003 ino:83438 sk:54 <->
	 ts sack reno wscale:7,7 rto:208 rtt:5.722/9.291 ato:40 mss:1448 pmtu:1500 rcvmss:536 advmss:1448 cwnd:10 bytes_acked:124 bytes_received:4 segs_out:8 segs_in:7 data_segs_out:3 data_segs_in:3 send 20.2Mbps lastsnd:1624 lastrcv:1580 lastack:1580 pacing_rate 40.5Mbps delivery_rate 8.7Mbps busy:44ms rcv_space:14480 rcv_ssthresh:64088 minrtt:0.705
```
From the output, you can see that `ss` can be used to display information like port number being used(`pid=7201`), flow identifier (`fd=3` or `fd=4`) and detailed statistics of the ongoing TCP transfer. 

One thing you should note here is that while we generated a single `iperf3` flow in the above experiment, the `ss` output printed statistics of two different flows with different IDs (`fd=3` and `fd=4`). While the flow with ID 4 (`fd=4`) is the one being used for the data transfer, TCP uses another flow usually with ID=3 (`fd=3`) to send certain control information over the network. For the rest of the experiment, while the `ss` commands we use will generate the socket statistics for the control flow (`fd=3`), you would need to ignore it in your data analysis and plotting, since this experiment is only focused on how TCP congestion control is used to control the "data" traffic. 

Once you have made yourself familiar with `ss` and `iperf3`, you may proceed to the next part of the experiment.

### Generate data

Next, we will generate some TCP flows between the two end hosts, and use it to observe the behavior of the TCP congestion control algorithm.

On the "receiver", run

```
iperf3 -s -1
```

On the "sender", run

```
EXPID=sender
```
On the same "sender" terminal where you ran the previous command, run `ss` in an infinite loop so that you can monitor and display TCP socket statistics for the entire duration of your experiment.

```
rm -f "$EXPID"-ss.txt; while true; do ss --no-header -eipn dst 10.10.2.2 | ts '%.s' | tee -a "$EXPID"-ss.txt; done
```

Open another SSH session to the "sender" and run

```
iperf3 -f m -c receiver -P 3 -t 60 -i 0
```

to start the TCP flows. Here

* `-t 60` says to run for 60 seconds
* `-c receiver` says to send traffic to the host named "receiver"
* `-P 3` says to send 3 parallel TCP flows
* `-i 0` is used to diable periodic bandwidth reports


You will see a final status report in the `iperf3` window after the `iperf3` sender finishes, which should take about a minute.  In the window where `ss` is running, you should see a several lines of output for each TCP flow. The `ss` output will also have been saved to a file named `sender-ss.txt`

Use `scp` to transfer this file to your own laptop for processing.

### Data Analysis

Use the following commands to processes the raw `ss` output into a csv file whose colums are : timestamp, flow ID, cwnd (in MSS), ssthresh (in MSS) and Smoothed RTT (in ms).

```
cat $EXPID-ss.txt | sed -e ':a; /<->$/ { N; s/<->\n//; ba; }'  | grep "iperf3" > $EXPID-ss-processed.txt
cat $EXPID-ss-processed.txt | awk '{print $1}' > ts-$EXPID.txt
cat $EXPID-ss-processed.txt | grep -oP '\bcwnd:.*?(\s|$)' |  awk -F '[:,]' '{print $2}' | tr -d ' ' > cwnd-$EXPID.txt
cat $EXPID-ss-processed.txt | grep -oP '\brtt:.*?(\s|$)' |  awk -F '[:,]' '{print $2}' | tr -d ' '  | cut -d '/' -f 1   > srtt-$EXPID.txt
cat $EXPID-ss-processed.txt | grep -oP '\bssthresh:.*?(\s|$)' |  awk -F '[:,]' '{print $2}' | tr -d ' ' > ssthresh-$EXPID.txt
cat $EXPID-ss-processed.txt | grep -oP '\bfd=.*?(\s|$)' |  awk -F '[=,]' '{print $2}' | tr -d ')' | tr -d ' '   > fd-$EXPID.txt
paste ts-$EXPID.txt fd-$EXPID.txt cwnd-$EXPID.txt ssthresh-$EXPID.txt srtt-$EXPID.txt -d ',' > $EXPID-ss.csv
```

Transfer the `ss-sender.txt` and `ss-sender.csv` files to your laptop using SCP. Use the .csv file to generate the plots required in the Exercise section below.



## Notes

### Exercise

Create a plot of the congestion window size and slow start threshold for each TCP flow over the duration of the experiment, similar to Figure 1 in the [Results](#results) section.

Annotate your plot, similar to Figure 2 in the [Results](#results) section, to show:

* Periods of "Slow Start" 
* Periods of "Congestion Avoidance"
* Instances where 3 duplicate ACKs were received (which will trigger "fast recovery")
* Instances of timeout

Using your plot and/or experiment data, explain how the behavior of TCP is different in the "Slow Start" and "Congestion Avoidance" phases. Also, using your plot, explain what happens to both the congestion window and the slow start threshold when 3 duplicate ACKs are received.

### Additional Experiment : Try different congestion control schemes

Several TCP congestion control algorithms have been proposed in the last three decades. TCP Reno and TCP CUBIC are the most widely deployed flavors of TCP congestion control in today's systems. To run the same experiment using the TCP CUBIC algorithm, you can modify the default congestion control at the "sender" and "receiver" nodes using :

```
sudo sysctl -w net.ipv4.tcp_congestion_control=cubic
```

Then run the same three steps as in the **Generate Data** section and save the ss output to a different file. Use the given data analysis steps and transfer the files to your local machine and plot the relevant data. The main difference between TCP Reno and TCP CUBIC is the window increase function. While Reno uses the traditional linear increase (W=W+1), CUBIC implements a binary search increase which can reach the available bandwidth much faster than Reno. You may read more about cubic at [TCP CUBIC](https://www.cs.princeton.edu/courses/archive/fall16/cos561/papers/Cubic08.pdf)

While TCP CUBIC and Reno are designed with the goal of high throughput, they tend to cause high queuing delays in the network, due to their buffer filling nature. A more recent congestion control proposed by Google tries to maximise throughput and at the same time minimise queuing delay in the network. This is known as the BBR congestion control and you can read more about this at this link [TCP BBR](https://research.google/pubs/pub45646/) and also have a look at the code at [BBR code](https://github.com/google/bbr)

To use the BBR congestion control for your experiment, run

```
sudo modprobe tcp bbr
```

on the "sender" and "receiver" nodes. This will load the linux kernel module for TCP BBR. Then, follow the same steps as in the **Generate Data** section. Make one change to the `iperf3` command at the sender 

```
iperf3 -f m -c receiver -C bbr -P 3 -t 60 -i 0
```
The **-C bbr** option is used to specify the congestion control which `iperf3` should use while running the test. So, you can use a different congestion control scheme in this experiment without having to necessarily modify the default congestion control of the "sender" and "receiver" hosts.
 
