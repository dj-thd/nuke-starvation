# nuke-starvation

## Description

This tool implements TCP starvation attack.
You can load test your servers given hostname and port. Requires Linux OS and root privileges (need to handle some iptables rules).

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
