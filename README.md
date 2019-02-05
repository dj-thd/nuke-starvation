# nuke-starvation

## Description

This tool implements TCP starvation attack.
You can load test your servers given hostname and port. Requires Linux OS and root privileges (need to handle some iptables rules).

## Explaination

A TCP normal connection packet flow is as follows:

- **Client** sends a TCP **SYN** (SYNchronize) packet to **Server**
- **Server** receives **Client**'s **SYN**

- **Server** sends a **SYN-ACK** (SYNchronize ACKnowledgement) packet to **Client**
- **Client** receives **Server**'s **SYN-ACK**

- **Client** sends **ACK** (ACKnowledge) packet to **Server**
- **Server** receives **Client**'s **ACK**

TCP connection is **ESTABLISHED**

**Client** and **Server** interchange data and **ACK** packets

**Client** wants to close connection, then:

- **Client** sends **FIN** (FINish) packet to **Server**
- **Server** receives **Client**'s **FIN**

- **Server** sends a **FIN-ACK** (FINish ACKnowledge) packet to **Client**
- **Client** receives **Server**'s **FIN-ACK**

TCP connection is **CLOSED**

TCP starvation attack works by blocking transmission of **FIN** and **RST** packets from client to server,
opening as many connections as possible and then closing it from the point of view of the client,
as the client wont receive the **FIN-ACK** packets from the server, we lower the kernel TCP timeout values
to recycle connections as soon as possible while the server thinks these connections are still open.

A TCP starvation attack packet flow is as follows:

- **Client** sends a TCP **SYN** (SYNchronize) packet to **Server**
- **Server** receives **Client**'s **SYN**

- **Server** sends a **SYN-ACK** (SYNchronize ACKnowledgement) packet to **Client**
- **Client** receives **Server**'s **SYN-ACK**

- **Client** sends **ACK** (ACKnowledge) packet to **Server**
- **Server** receives **Client**'s **ACK**

TCP connection is **ESTABLISHED**
	
**Client** wants to close connection, then:

- **Client** OS try to send **FIN** (FINish) packet to **Server**, but is blocked by iptables
- **Server** dont receive anything, letting the connection **ESTABLISHED**

- **Client**'s connection is in **FIN_WAIT1** state
- After short timeout that we have tuned at the client, **Client**'s connection is **CLOSED** and resources are deallocated
- **Server**'s connection is still **ESTABLISHED**
- After **Server**'s established connection timeout or application timeout, **Server** close the connection

On unprotected servers this cause a big resource allocation to serve that requests, causing high CPU and
memory usage rendering the service unusable and in some cases killing the service application by the OS
out of memory process killer when the server runs out of memory. If the server is not properly protected,
a client can bring it down with a very little bandwidth usage.

This is not the same as a classic SYN flood, as the full 3-way TCP handshake is completed and from the
point of view of the server these are legit open TCP connections.

This can be mitigated by setting hard limits on simultaneous open TCP connections per client and TCP
connection rate per client, as well as tuning TCP timeouts with more aggressive values.

## Mitigation

### pfsense

Advanced settings / firewall -> set state timeouts
 - TCP first
 - TCP opening
 - TCP closing
 - TCP FIN wait

Advanced settings / system tunables -> set `net.inet.tcp.syncookies = 1`, add if not exist

On WAN->LAN NAT rules, set reasonable connection rates/second per host, and max connections per host

### linux

1. Kernel parameters:
 - Enable TCP syn cookies: `echo 1 > /proc/sys/net/ipv4/tcp_syncookies`
 - Increase TCP syn backlog: `echo 2048 > /proc/sys/net/ipv4/tcp_max_syn_backlog`
 - Decrease TCP syn-ack retries: `echo 3 > /proc/sys/net/ipv4/tcp_synack_retries`

2. iptables (adapt rules to your use case, ports and etc.):

Limit new connections per second/client ip to a reasonable value:
```
iptables -I INPUT -p tcp --syn -m hashlimit --hashlimit-above 20/s --hashlimit-burst 30 --hashlimit-mode srcip --hashlimit-htable-expire 60000 --hashlimit-name antistarvation-j DROP
```

Limit number of simultaneous established connections per client to a reasonable value:
```
iptables -I INPUT -p tcp --syn --dport 80 -m connlimit --connlimit-above 200 -j DROP
```

## TODOs

- Save current kernel parameters and restore them on exit
