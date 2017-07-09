---
layout: post
title:  "Understanding EIGRP. Part 1: Metric"
date:   2017-07-08 20:00:00 +0200
categories: blog
tags: [ccie, routing, eigrp]
comments: true
permalink: /:categories/understanding-eigrp-metric/
---
This is the first post in a series about routing protocol EIGRP.

Today we will deep dive into how metric is calculated in EIGRP classic mode.
EIGRP is a distance-vector routing protocol, but what does it really mean? How does a distance-vector routing protocol differ from link-state?
Distance-vector is also referred to as "routing by rumor". It means that a router makes its decision based on the metrics reported by his neighbors.
This is unlike a link-state routing protocol where every router in an area (OSPF) or level (IS-IS) knows the whole topology for this area/level and calculates the best path locally using the shortest-path algorithm (Dijkstra). Because routers have the same information, it results in a consistent choice of the best path.
<!--more-->

Let's start by defining several EIGRP terms that will be used throughout this post:
**Reported Distance (RD)** - best metric to the destination from neighbor's perspective.
**Computed Distance (CD)** - calculated metric to the destination which takes into account reported distance from the neighbor as well as parameters of the interface where an update was received.
Metric in EIGRP is **composite** and calculated from four individual components: minimal bandwidth, cumulative delay, lowest reliability, highest load.
There are also 5 K-values in classic mode that are just coefficients ranging from 0 to 255.
Let's look on the formula of metric calculation in classic mode:

$$ Metric = 256 \left(K1 * bandwidth + \frac{K2 * bandwidth}{256 - load} + K3 * delay\right)\frac{K5}{reliability + K4} $$

$$ bandwidth = \frac{10^7}{min\_bandwidth} $$

$$ delay = \frac{total\_delay}{10} $$

where
* **min_bandwidth** is a minimum bandwidth across the path in kbps,
* **total_delay** is a cumulative delay across the path in microseconds,
* **load** is the highest load across the path,
* **reliability** is the lowest reliability across the path.

If **K5=0**, the formula changes to:

$$ Metric = 256 \left(K1 * bandwidth + \frac{K2 * bandwidth}{256 - load} + K3 * delay\right) $$

Even though **minimum MTU** is calculated and included in an update, MTU has absolutely **no effect** on EIGRP metric calculation or path selection.
Change of a load or reliability does not trigger an update in contrast to change of delay or bandwidth.
K-values must match across the whole EIGRP domain to establish adjacencies and are exchanged only in Hello messages. By default, **K1=K3=1** and **K2=K4=K5=0** so only bandwidth and delay impact metric calculation.
An interesting question is *"How does EIGRP know about minimum bandwidth or total delay across the path if it routes by rumor?"*
To answer this question the first thing you should know is that in an update EIGRP ***never sends*** calculated values (Computed Distance or Reported Distance), EIGRP sends only individual metric components.

Let's look on the topology below:
![EIGRP metric topology](https://s3.us-east-2.amazonaws.com/media.dmfigol.me/eigrp-metric-topology.svg)

We have three routers: R1, R2 and R3. On R3 there is a loopback 0 with IP address 10.1.3.3/32. We will analyze the metric to this prefix from all three routers.
I also set bandwidth on R1 e0/0 to 5 Mbps.
Let's look on Loopback0 interface statistics and parameters:
{% highlight text %}
R3#show interfaces lo0
Loopback0 is up, line protocol is up
  Hardware is Loopback
  Internet address is 10.1.3.3/32
  MTU 1514 bytes, BW 8000000 Kbit/sec, DLY 5000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
...
<omitted for brevity>
{% endhighlight %}

Bandwidth is **8 Gbps** and delay is **5000 microseconds**.
I have EIGRP enabled already, so let's look on EIGRP topology table for the prefix 10.1.3.3/32 on R3.
{% highlight text %}
R3#show ip eigrp topology 10.1.3.3/32
EIGRP-IPv4 Topology Entry for AS(100)/ID(10.1.3.3) for 10.1.3.3/32
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 128256
  Descriptor Blocks:
  0.0.0.0 (Loopback0), from Connected, Send flag is 0x0
      Composite metric is (128256/0), route is Internal
      Vector metric:
        Minimum bandwidth is 8000000 Kbit
        Total delay is 5000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1514
        Hop count is 0
        Originating router is 10.1.3.3
{% endhighlight %}

We can see that **Computed Distance** is **128256** and **Reported Distance** is **0**.
The latter one is 0, because this router originated the prefix. Computed distance is calculated from the individual metric components which are listed in the same output.
Let's calculate the metric manually (I usually use [IPython](https://ipython.org/){:target="_blank"} for this purpose):
{% highlight python %}
In [1]: bw = 8000000

In [2]: delay = 5000

In [3]: K1 = 1

In [4]: K3 = 1

In [5]: 256 * (int(K1 * 10**7 / bw) + K3 * delay / 10)
Out[5]: 128256.0

{% endhighlight %}
Bingo! The same value.
Note: because division does not give an integer value, I had to use function **int**, which takes only integer part of the value.
The only thing I didn't explain yet is what the values of individual metric components listed in *show ip eigrp topology* output are. In this case they are the same as loopback interface parameters. So are those values always equal to the interface parameters?
The answer is no.
Let's have a look on R2 topology table:
{% highlight text %}
R2#show ip eigrp topology 10.1.3.3/32
EIGRP-IPv4 Topology Entry for AS(100)/ID(10.1.2.2) for 10.1.3.3/32
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 409600
  Descriptor Blocks:
  10.0.23.3 (Ethernet0/1), from 10.0.23.3, Send flag is 0x0
      Composite metric is (409600/128256), route is Internal
      Vector metric:
        Minimum bandwidth is 10000 Kbit
        Total delay is 6000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 1
        Originating router is 10.1.3.3
{% endhighlight %}
**Computed Distance** is **409600** and **Reported Distance** is **128256**.
Before we understand how we received those, let's take a look on the metric components values:
* Minimum **bandwidth** is **10000** Kbit
* Total **delay** is **6000** microseconds

An interface where an update was received has the following parameters:
{% highlight text %}
R2#show interfaces e0/1
Ethernet0/1 is up, line protocol is up
  Hardware is AmdP2, address is aabb.cc00.0210 (bia aabb.cc00.0210)
  Internet address is 10.0.23.2/24
  MTU 1500 bytes, BW 10000 Kbit/sec, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
...
<omitted for brevity>
{% endhighlight %}
Bandwidth is **10000 kbps** and delay is **1000 microseconds**.
The values listed in *show ip eigrp topology* are obtained using the following process:
* Minimum bandwidth: min(8 Gbps, 10 Mbps) = 10 Mbps = **10000 kbps**
* Total delay: 5000 + 1000 = **6000 microseconds**

If we plug the resulting values into EIGRP formula we should receive Computed Distance **409600**:
{% highlight python %}
In [8]: bw = 10000

In [9]: delay = 6000

In [10]: K1 = 1

In [11]: K3 = 1

In [12]: 256 * (int(K1 * 10**7 / bw) + K3 * delay / 10)
Out[12]: 409600.0
{% endhighlight %}
Perfect!
I told you already that both Reported Distance and Computed Distance are never sent in EIGRP update. So how does R2 get the value **128256** for Reported Distance?
### EIGRP metric calculation process when an update is received
1. EIGRP update contains individual metric components that the neighbor is using to calculate **Computed Distance**
2. Receiving router takes those values and plugs them into the formula. Result of this calculation is **Reported Distance**
3. Next, EIGRP calculates new values of metric components using old values and values of the interface where an update was received:
   * Lowest bandwidth between two: *min(bw_update, bw_link)*
   * Sum of delays: *dly_update + dly_link*
   * Lowest reliability: *min(reliability_update, reliability_link)*
   * Highest load: *max(reliability_update, reliability_link)*
4. EIGRP plugs these new values into the formula again and gets **Computed Distance**

When this router is going to send an update, it will include these new individual components.

To reinforce this information let's calculate metric once again for R1 in our topology. R2 will send an update including these values:
* Bandwidth: 10000 kbps
* Delay: 6000 microseconds
* Reliability: 255
* Load: 1

Interface Ethernet0/0 on R1 has the following parameters:
{% highlight text %}
R1#show interfaces e0/0
Ethernet0/0 is up, line protocol is up
  Hardware is AmdP2, address is aabb.cc00.0100 (bia aabb.cc00.0100)
  Internet address is 10.0.12.1/24
  MTU 1500 bytes, BW 5000 Kbit/sec, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
...
<omitted for brevity>
{% endhighlight %}
* Bandwidth: 5000 kbps
* Delay: 1000 microseconds
* Reliability: 255
* Load: 1

Reported Distance should be the same as Computed Distance from R2 perspective: **409600** (calculated above).
Let's now calculate new metric components and **Computed Distance**:
* Bandwidth: *min(bw_update, bw_link) = min(10000, 5000) = **5000***
* Delay: *dly_update + dly_link = 6000 + 1000 = **7000***
* Reliability: *min(reliability_update, reliability_link) = min(255, 255) = **255***
* Load: *max(reliability_update, reliability_link) = max(1, 1) = **1***

{% highlight python %}
In [14]: bw = 5000

In [15]: delay = 7000

In [16]: K1 = 1

In [17]: K3 = 1

In [18]: 256 * (int(K1 * 10**7 / bw) + K3 * delay / 10)
Out[18]: 691200.0
{% endhighlight %}

So RD should be **409600** and CD should be **691200**.
Let's check:
{% highlight text %}
R1#show ip eigrp topology 10.1.3.3/32
EIGRP-IPv4 Topology Entry for AS(100)/ID(10.1.1.1) for 10.1.3.3/32
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 435200
  Descriptor Blocks:
  10.0.12.2 (Ethernet0/0), from 10.0.12.2, Send flag is 0x0
      Composite metric is (691200/409600), route is Internal
      Vector metric:
        Minimum bandwidth is 5000 Kbit
        Total delay is 7000 microseconds
        Reliability is 255/255
        Load is 1/255
        Minimum MTU is 1500
        Hop count is 2
        Originating router is 10.1.3.3
{% endhighlight %}

Great! It wasn't so hard, was it?
This is it for this post. In the next EIGRP post we will see what Feasible Distance, Successor, Feasible Successor are. Additionally I will show that Feasible Distance may differ from the lowest Computed Distance and how EIGRP may go to Active state even when there is a Feasible Successor.
Useful EIGRP Resources:
* [My slidedeck](https://www.slideshare.net/DmitryFigol/routing-protocol-eigrp) with a quick overview of EIGRP for CCIE R&S
* [EIGRP RFC 7868](https://tools.ietf.org/html/rfc7868)
* Book [CCIE R&S 5.0 Official Certification Guide, Volume 1](https://www.safaribooksonline.com/library/view/ccie-routing-and/9780133481617/)
* Book [EIGRP for IP: Basic Operation and Configuration](https://www.safaribooksonline.com/library/view/eigrp-for-ip/9780321618146/)

Thanks for reading! If you liked the post, don't hesitate to share and subscribe!
Till next time!
