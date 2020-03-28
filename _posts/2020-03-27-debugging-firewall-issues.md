---
layout: post
title:  "Debugging firewall issues"
date:   2020-03-27 12:27:43
categories: networking tools
---

<img src="/images/debugging-firewall-issues/firewall-146529_1280.png" title="Firewall" class="float-left" />

Challenges when working from home
---------------------------------

Nowadays more and more people work from home (especially when a deadly virus is
roaming around the world). Not everyone works fully
remote, but often employers allow their employees to save time on commute
by working up to 2 days a week (and even more if you are lucky) from the
comfort of their homes.

As with everything, remote work is not a silver bullet and apart from obvious
upsides (like removing commute time from the equation) there are downsides as well.

Usually, the experience of working over VPN (Virtual Private Network) is
far from ideal, but also some services that used to work in the office no longer
seem to work and coincidentally this usually happens while you are accessing
it from a private network.

Most often than not this is caused by the firewall rules that prevent your
computer from connecting to your precious service. Quite likely your company
has their own reasons for blocking access to some of its internal servers,
but also, depending on the level of your soft skills, you should be able to
convince your boss (or the dreaded IT security manager) to open this or that
port to a certain set of hostnames.

Problematic investigation
-------------------------

However, before you actually report this problem to anyone you should
first find out where the problem is. And a little bit of knowledge about
the TCP protocol may come in handy.

Suppose you are running some service on port _1234_.

You can check whether the service is waiting for connections
by executing the following command on the remote server:

{% highlight bash %}
netstat -an | grep 1234
tcp4       0      0  *.1234                 *.*                    LISTEN     
tcp6       0      0  *.1234                 *.*                    LISTEN
{% endhighlight %}

This confirms that there is a service that LISTENs on port 1234 for connections
over either IPv4 (tcp4) or IPv6 (tcp6).

What are the other statuses that a TCP connection could be in? Below state
transition diagram shows how it works:

<img src="/images/debugging-firewall-issues/tcp-state-transition-diagram.png" title="Firewall" style="clear: both;" />

There are actually more states than that but these are essential when talking
about initiating a TCP connection:

* CLOSED - represents no connection state at all. __[we start here]__
* LISTEN - represents waiting for a connection request from any remote TCP and port.
* SYN_SENT - represents waiting for a matching connection request after having sent a connection request.
* SYN_RECEIVED - represents waiting for a confirming connection request acknowledgment after having both received and sent a connection request.
* ESTABLISHED - represents an open connection, data received can be delivered to the user. The normal state for the data transfer phase of the connection. __[...and want to reach here]__

Please also notice the colours of the arrows: orange arrow represents the path
that the client follows and blue represents the transitions through which the server
goes.

Three way handshake
-------------------

Actually this process of establishing a connection in TCP is called __a three way
handshake__. In my opinion an easier way to understand the flow is to look at the
transitions of the client and the server separately:

<img src="/images/debugging-firewall-issues/tcp_three_way_handshake.png" title="Firewall" style="clear: both;" />

Why does it matter?
-------------------

Basically, if the firewall is dropping your packets the TCP client will
get stuck in the SYN_SENT state. Normally SYN_SENT is a state in which clients
stay only for a brief period of time.

In order to confirm that the firewall is rejecting your messages you can
run the following command:

{% highlight bash %}
netstat -an | grep SYN_SENT
{% endhighlight %}

In case your gut feeling was right you will see something like this:

{% highlight bash %}
tcp        0      1 127.0.0.1:43530        12.34.56.78:1234         SYN_SENT
{% endhighlight %}

From your perspective it almost always means that you will need to either
fix the rules in the firewall configuration yourself or,
using the art of charm, convince someone to do that for you.
