---
title: IPSec VPN Site-to-Site_2
subtitle: La minaccia dell'autenticazione con certificato
layout: post
background: '/img/bg-post.jpg'
categories: networking fortigate
---

Abbiamo [già visto](https://jupitersinsight.github.io/networking/fortigate/2023/08/30/ipsec-vpn-site2site.html) come configurare una connessione VPN Site-to-Site usando l'autenticazione basata su _chiave condivisa_, o _pre-shared key_, oggi vedremo come usare l'autenticazione con certificato digitale.  

In questo articolo useremo i certificati built-in dei FortiGate, ***Fortinet_CA*** e ***Fortinet_Factory***. Per utlizzare certificati personalizzati, occorre importarli in entrambi i FortiGate prima di configurare il profilo della VPN.  

Alcuni dettagli e configurazioni (ad esempio, la configurazione del server DHCP) sono trattati nel post precedente.

## Topologia

<img src="/assets/fortinet/site2site2.JPG" alt="Topologia lab" width="95%" height="auto"> 

## Configurazione firewall "HQ"

Dopo aver configurato l'interfaccia WAN e le rotte statiche, si procede con la configurazione del valore **peer** che serve per autenticare l'altro capo del tunnel.  
```
config user peer
    edit "peer1"
        set ca "Fortinet_CA"
    next
end
```

**Configurazione profilo VPN**

```
config vpn ipsec phase1-interface
    edit "versoBranch"
        set interface "port2"
        set keylife 7200
        set authmethod signature
        set net-device enable
        set proposal des-sha256 des-sha384
        set dhgrp 14
        set remote-gw 190.255.255.2
        set certificate "Fortinet_Factory"
        set peer "peer1"
    next
end
```
```
config vpn ipsec phase2-interface
    edit "versoBranch"
        set phase1name "versoBranch"
        set proposal des-sha256
        set dhgrp 14
        set auto-negotiate enable
        set keylifeseconds 3600
        set src-subnet 192.168.90.0 255.255.255.0
        set dst-subnet 192.168.190.0 255.255.255.0
    next
end
```
Ricordiamoci di configurare la rotta statica _blackhole_ e di configurare le policy necessarie per il passaggio e la ricezione del traffico.  

**Ricorda**: oltre alla configurazione di rete, i firewall richiedono anche la configurazione delle policy per il corretto fluire del traffico.

## Configurazione firewall "Branch"

```
config user peer
    edit "peer2"
        set ca "Fortinet_CA"
    next
end
```

**Configurazione profilo VPN**

```
config vpn ipsec phase1-interface
    edit "versoHQ"
        set interface "port2"
        set keylife 7200
        set authmethod signature
        set net-device enable
        set proposal des-sha256 des-sha384
        set dhgrp 14
        set remote-gw 90.255.255.2
        set certificate "Fortinet_Factory"
        set peer "peer2"
    next
end

```
```
config vpn ipsec phase2-interface
    edit "versoHQ"
        set phase1name "versoHQ"
        set proposal des-sha256
        set dhgrp 14
        set auto-negotiate enable
        set keylifeseconds 3600
        set src-subnet 192.168.190.0 255.255.255.0
        set dst-subnet 192.168.90.0 255.255.255.0
    next
end
```
## Verifica di funzionamento tra i due endpoint

**Ping da client connesso a HQ a client connesso a Branch**
```
root@PC1:~# ping 192.168.190.102 -c 5
PING 192.168.190.102 (192.168.190.102) 56(84) bytes of data.
64 bytes from 192.168.190.102: icmp_seq=1 ttl=62 time=26.3 ms
64 bytes from 192.168.190.102: icmp_seq=2 ttl=62 time=21.6 ms
64 bytes from 192.168.190.102: icmp_seq=3 ttl=62 time=20.9 ms
64 bytes from 192.168.190.102: icmp_seq=4 ttl=62 time=19.0 ms
64 bytes from 192.168.190.102: icmp_seq=5 ttl=62 time=18.9 ms

--- 192.168.190.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4009ms
rtt min/avg/max/mdev = 18.907/21.346/26.299/2.690 ms
```

**Ping da client connesso a Branch a client connesso a HQ**
```
root@PC2:~# ping 192.168.90.102 -c 5
PING 192.168.90.102 (192.168.90.102) 56(84) bytes of data.
64 bytes from 192.168.90.102: icmp_seq=1 ttl=62 time=16.6 ms
64 bytes from 192.168.90.102: icmp_seq=2 ttl=62 time=13.3 ms
64 bytes from 192.168.90.102: icmp_seq=3 ttl=62 time=23.6 ms
64 bytes from 192.168.90.102: icmp_seq=4 ttl=62 time=25.4 ms
64 bytes from 192.168.90.102: icmp_seq=5 ttl=62 time=18.5 ms

--- 192.168.90.102 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 13.285/19.485/25.350/4.453 ms
```
## Da GUI come si vede quanto configurato da CLI?

Dettaglio certificato e peer configurati per la phase1  

<img src="/assets/fortinet/site2site-certificato/dettaglio-phase1.JPG" alt="Dettaglio phase1" width="95%" height="auto">

## Comandi per il troubleshooting

Oltre a quanto visibile da GUI (eventi, traffico permesso e bloccato, policy lookup e route lookup...) da CLI sono utili i comandi `diagnose vpn ipsec status`, `diagnose vpn tunnel list`, `diagnose vpn ike gateway list`, `diagnose debug application ike`, `diagnose firewall iprobe`, `get router info routing-table all`.