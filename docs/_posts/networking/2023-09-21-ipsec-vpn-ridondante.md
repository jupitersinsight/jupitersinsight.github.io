---
title: IPSec VPN Site-to-Site Ridondante
subtitle: VPN ridondante con rotte statiche
layout: post
background: '/img/bg-post.jpg'
categories: networking fortigate
---

Come anticipato dal titolo, in questo articolo vediamo come configurare una VPN ridondante tra due peer (sempre FortiGate).  

Essendoci rotte verso le stesse destinazioni sta a noi decidere se configurarle con la stessa AD (_Administrative Distance_) in modo da ottenere inserire tutte le rotte nella Routing Table, oppure configurarne una principale e le altre di backup.  

La configurazione dei Firewall è molto simile a quella del primo post di cui lascio qui il [link](https://jupitersinsight.github.io/networking/fortigate/2023/08/30/ipsec-vpn-site2site.html).

## Topologia

<img src="/assets/fortinet/vpn-ridondanti/topologia.JPG" alt="Topologia" width="95%" height="auto">

## Configurazione FG1

**Configurazione Phase1 e Phase2**

```
FG1 # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "link1"
        set interface "port1"
        set local-gw 1.1.1.2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 3.3.3.2
        set psksecret ENC <chiave precondivisa>
    next
    edit "link2"
        set interface "port2"
        set local-gw 2.2.2.2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 4.4.4.2
        set psksecret ENC <chiave precondivisa>
    next
end
```

```
FG1 # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "link1"
        set phase1name "link1"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 192.168.1.0 255.255.255.0
        set dst-subnet 192.168.2.0 255.255.255.0
    next
    edit "link2"
        set phase1name "link2"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 192.168.1.0 255.255.255.0
        set dst-subnet 192.168.2.0 255.255.255.0
    next
end
```

**Configurazione policy**

```
FG1 # show firewall policy
config firewall policy
    edit 1
        set name "ACCETTA_link1-out"
        set uuid d16131c8-56ff-51ee-c8a3-88c6edf8f5e6
        set srcintf "port10"
        set dstintf "link1"
        set action accept
        set srcaddr "FG1_subnets"
        set dstaddr "FG2_subnets"
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "ACCETTA_link1-in"
        set uuid 69153096-5700-51ee-4384-02702befe036
        set srcintf "link1"
        set dstintf "port10"
        set action accept
        set srcaddr "FG2_subnets"
        set dstaddr "FG1_subnets"
        set schedule "always"
        set service "ALL"
    next
    edit 3
        set name "ACCETTA_link2-out"
        set uuid 74a59f86-57c8-51ee-2769-08a83a51b7f4
        set srcintf "port10"
        set dstintf "link2"
        set action accept
        set srcaddr "FG1_subnets"
        set dstaddr "FG2_subnets"
        set schedule "always"
        set service "ALL"
    next
    edit 4
        set name "ACCETTA_link2-in"
        set uuid a4b0ad42-57c8-51ee-3a3b-84b4c118e56b
        set srcintf "link2"
        set dstintf "port10"
        set action accept
        set srcaddr "FG2_subnets"
        set dstaddr "FG1_subnets"
        set schedule "always"
        set service "ALL"
    next
end
```

**Configurazione rotte statiche**

```
FG1 # show router static
config router static
    edit 1
        set gateway 1.1.1.1
        set device "port1"
    next
    edit 2
        set gateway 2.2.2.1
        set device "port2"
    next
    edit 3
        set dst 192.168.2.0 255.255.255.0
        set device "link1"
    next
    edit 4
        set dst 192.168.2.0 255.255.255.0
        set distance 254
        set blackhole enable
    next
    edit 5
        set dst 192.168.2.0 255.255.255.0
        set device "link2"
    next
end
```

## Configurazione FG2

**Configurazione Phase1 e Phase2**

```
FG2 # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "link1"
        set interface "port2"
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 1.1.1.2
        set psksecret ENC <chiave precondivisa>
    next
    edit "link2"
        set interface "port1"
        set local-gw 4.4.4.2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 2.2.2.2
        set psksecret ENC <chiave precondivisa>
    next
end
```

```
FG2 # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "link1"
        set phase1name "link1"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 192.168.2.0 255.255.255.0
        set dst-subnet 192.168.1.0 255.255.255.0
    next
    edit "link2"
        set phase1name "link2"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 192.168.2.0 255.255.255.0
        set dst-subnet 192.168.1.0 255.255.255.0
    next
end
```

**Configurazione policy**

```
FG2 # show firewall policy
config firewall policy
    edit 1
        set name "ACCETTA_link1-out"
        set uuid f998f6ec-5701-51ee-bacc-48c3aa0bcc04
        set srcintf "port10"
        set dstintf "link1"
        set action accept
        set srcaddr "FG2_subnets"
        set dstaddr "FG1_subnets"
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "ACCETTA_link1-in"
        set uuid 22bf8df6-5702-51ee-2eb2-671bb4a9f487
        set srcintf "link1"
        set dstintf "port10"
        set action accept
        set srcaddr "FG1_subnets"
        set dstaddr "FG2_subnets"
        set schedule "always"
        set service "ALL"
    next
    edit 3
        set name "ACCETTA_link2-out"
        set uuid 6fe97642-57c9-51ee-967c-154316ec1e3a
        set srcintf "port10"
        set dstintf "link2"
        set action accept
        set srcaddr "FG2_subnets"
        set dstaddr "FG1_subnets"
        set schedule "always"
        set service "ALL"
    next
    edit 4
        set name "ACCETTA_link2-in"
        set uuid 96f8d0f2-57c9-51ee-a56a-04357d448aa7
        set srcintf "link2"
        set dstintf "port10"
        set action accept
        set srcaddr "FG1_subnets"
        set dstaddr "FG2_subnets"
        set schedule "always"
        set service "ALL"
    next
end
```

**Configurazione rotte statiche**

```
FG2 # show router static
config router static
    edit 1
        set gateway 3.3.3.1
        set device "port2"
    next
    edit 2
        set gateway 4.4.4.1
        set device "port1"
    next
    edit 3
        set dst 192.168.1.0 255.255.255.0
        set device "link1"
    next
    edit 4
        set dst 192.168.1.0 255.255.255.0
        set distance 254
        set blackhole enable
    next
    edit 5
        set dst 192.168.1.0 255.255.255.0
        set device "link2"
    next
end
```

## ECMP

Bene, dopo questa lunga carrellata sulla configurazione dei firewall, vediamo prima di tutto l'output della routing table.

**RIB da FG1**

```
Routing table for VRF=0
S*      0.0.0.0/0 [10/0] via 1.1.1.1, port1, [1/0]
                  [10/0] via 2.2.2.1, port2, [1/0]
C       1.1.1.0/30 is directly connected, port1
C       2.2.2.0/30 is directly connected, port2
C       192.168.1.0/24 is directly connected, port10
S       192.168.2.0/24 [10/0] via link1 tunnel 3.3.3.2, [1/0]
                       [10/0] via link2 tunnel 4.4.4.2, [1/0]

```

**RIB da FG2**

```
Routing table for VRF=0
S*      0.0.0.0/0 [10/0] via 3.3.3.1, port2, [1/0]
                  [10/0] via 4.4.4.1, port1, [1/0]
C       3.3.3.0/30 is directly connected, port2
C       4.4.4.0/30 is directly connected, port1
S       192.168.1.0/24 [10/0] via link1 tunnel 1.1.1.2, [1/0]
                       [10/0] via link2 tunnel 2.2.2.2, [1/0]
C       192.168.2.0/24 is directly connected, port10
```

In entrambi i firewall le rotte per le subnet remote hanno la stessa distanza: ECMP (Equal-Cost-Multi-Path) routing.
Questo significa che per arrivare alla stessa destinazione esistono percorsi multipli e come amministratori è possibile manipolare il traffico in modo che sia bilanciato nel modo migliore.

```
FG1 (settings) # set v4-ecmp-mode
source-ip-based         Select next hop based on source IP.
weight-based            Select next hop based on weight.
usage-based             Select next hop based on usage.
source-dest-ip-based    Select next hop based on both source and destination IPs.
```

Come impostazione predefinita, le sessioni sono divise equamente tra le due interfacce e sono distinte in base all'indirizzo IP di origine (`source-ip-based`).  
O in aggiunta distinte anche in base alla destinazione (`source-dest-ip-based`).

È possibile configurare in base al _peso_ assegnato alla rotta in modo che, ad esempio, l'80% delle sessioni passi dalla interfaccia 1 e il restante 20% dall'interfaccia 2 (`weight-based`).

Oppure permettere l'utilizzo della seconda interfaccia solo quando il traffico sulla interfaccia principale eccede il limite di banda in ingresso e in uscita (`usage-based`).  

**Nota**: È possibile modificare il numero di _paths_ per rotte ECMP. Impostando il valore a **1**, si disattiva il routing ECMP.
```
FG1 (settings) # set ecmp-max-paths
ecmp-max-paths    Enter an integer value from <1> to <255> (default = <255>).
```

## Usare la priority per determinare una rotta principale

In base alla configurazione a inizio articolo, avviando il debug del flusso dei pacchetti su FG1, il comando ping lanciato da Client1 produce il seguente risultato:

```
FG1 # id=20085 trace_id=2 func=print_pkt_detail line=5845 msg="vd-root:0 received a packet(proto=1, 192.168.1.103:5->192.168.2.102:2048) tun_id=0.0.0.0 from port10. type=8, code=0, id=5, seq=1."
id=20085 trace_id=2 func=init_ip_session_common line=6024 msg="allocate a new session-00000080, tun_id=0.0.0.0"
id=20085 trace_id=2 func=vf_ip_route_input_common line=2605 msg="find a route: flag=04000000 gw-3.3.3.2 via link1"
id=20085 trace_id=2 func=fw_forward_handler line=881 msg="Allowed by Policy-1:"
id=20085 trace_id=2 func=ipsecdev_hard_start_xmit line=669 msg="enter IPSec interface link1, tun_id=0.0.0.0"
id=20085 trace_id=2 func=_do_ipsecdev_hard_start_xmit line=229 msg="output to IPSec tunnel link1"
id=20085 trace_id=2 func=esp_output4 line=844 msg="IPsec encrypt/auth"
id=20085 trace_id=2 func=ipsec_output_finish line=544 msg="send to 1.1.1.1 via intf-port1"
```
Possiamo vedere che la destinazione del ping, l'indirizzo IP 192.168.2.102, è stata raggiunta passando dal tunnel IPSec via link1.

Alterando la priority della rotta via link1, favoriamo il link2 per il passaggio dei dati:
```
FG1 (3) # show
config router static
    edit 3
        set dst 192.168.2.0 255.255.255.0
        set priority 5
        set device "link1"
    next
end
```
```
Routing table for VRF=0
S*      0.0.0.0/0 [10/0] via 1.1.1.1, port1, [1/0]
                  [10/0] via 2.2.2.1, port2, [1/0]
C       1.1.1.0/30 is directly connected, port1
C       2.2.2.0/30 is directly connected, port2
C       192.168.1.0/24 is directly connected, port10
S       192.168.2.0/24 [10/0] via link2 tunnel 4.4.4.2, [1/0]
                       [10/0] via link1 tunnel 3.3.3.2, [5/0]
```

Come da nuova configurazione, il traffico passa per il tunnel IPsec via link2:
```
FG1 # id=20085 trace_id=3 func=print_pkt_detail line=5845 msg="vd-root:0 received a packet(proto=1, 192.168.1.103:7->192.168.2.102:2048) tun_id=0.0.0.0 from port10. type=8, code=0, id=7, seq=1."
id=20085 trace_id=3 func=init_ip_session_common line=6024 msg="allocate a new session-0000008b, tun_id=0.0.0.0"
id=20085 trace_id=3 func=vf_ip_route_input_common line=2605 msg="find a route: flag=04000000 gw-4.4.4.2 via link2"
id=20085 trace_id=3 func=fw_forward_handler line=881 msg="Allowed by Policy-3:"
id=20085 trace_id=3 func=ipsecdev_hard_start_xmit line=669 msg="enter IPSec interface link2, tun_id=0.0.0.0"
id=20085 trace_id=3 func=_do_ipsecdev_hard_start_xmit line=229 msg="output to IPSec tunnel link2"
id=20085 trace_id=3 func=esp_output4 line=844 msg="IPsec encrypt/auth"
id=20085 trace_id=3 func=ipsec_output_finish line=544 msg="send to 2.2.2.1 via intf-port2"
```

## Configurare il link secondario come backup

Nel caso in cui si voglia utilizzare un solo link per il traffico dei dati e configurare il secondo come backup, il comando è `<phase1-interface secondaria> # set monitor <phase1-interface principale>`.

## Comandi per debug flusso pacchetti

In ordine i comandi sono:

```
diagnose debug flow filter
clear      Clear filter.
vd         Index of virtual domain.
vd-name    Name of virtual domain.
proto      Protocol number.
addr       IP address.
saddr      Source IP address.
daddr      Destination IP address.
port       port
sport      Source port.
dport      Destination port.
negate     Inverse filter.
```
```
diagnose debug enable
```
```
diagnose debug flow trace start/stop
```