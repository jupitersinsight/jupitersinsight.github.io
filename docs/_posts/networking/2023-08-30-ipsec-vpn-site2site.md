---
title: IPSec VPN Site-to-Site
subtitle: Connessione VPN punto-punto usando Firewall FortiGate
layout: post
background: '/img/bg-post.jpg'
categories: networking fortigate
---

In questo articolo vediamo come configurare una connessione Site-to-Site tra due firewall FortiGate.  
I due firewall sono connessi allo stesso router che per semplificazione si assume l'importante ruolo di rappresentare tutto ciò che potrebbe dividere una sede principale da un distaccamento.

## Topologia

<img src="/assets/fortinet/site2site.JPG" alt="Topologia lab" width="95%" height="auto" >

## Configurazione del router "ISP"

Il router che chiameremo "ISP" ha due interfacce configurate ognuna con il proprio indirizzamento:

```
GigabitEthernet2/0         188.255.255.1   YES manual up                    up
GigabitEthernet3/0         88.255.255.1    YES manual up                    up
```
Quindi, GE2/0 ha indirizzo IP 188.255.255.1/32 mentre GE3/0 ha indirizzo IP 88.255.255.1/32.  

Per completezza, qui sotto abbiamo la routing table. 

```
Gateway of last resort is not set

     188.255.0.0/30 is subnetted, 1 subnets
C       188.255.255.0 is directly connected, GigabitEthernet2/0
     88.0.0.0/30 is subnetted, 1 subnets
C       88.255.255.0 is directly connected, GigabitEthernet3/0

```

## Configurazione firewall "HQ"  

**Configurazione interfacce di rete**  

Iniziamo a configurare le porte dove _port2_ corrisponde alla WAN, connessa al router "ISP", e _port10_ corrisponde alla LAN.

```
HQ # show system interface port2
config system interface
    edit "port2"
        set vdom "root"
        set ip 188.255.255.2 255.255.255.252
        set allowaccess ping
        set type physical
        set snmp-index 2
    next
end
```
```
HQ # show system interface port10
config system interface
    edit "port10"
        set vdom "root"
        set ip 192.168.188.1 255.255.255.0
        set allowaccess ping
        set type physical
        set snmp-index 10
    next
end
```

Al client connesso alla LAN possiamo assegnare staticamente un indirizzo IP, oppure configurare nel FortiGate un server DHCP.

```
HQ # show system dhcp server
config system dhcp server
    edit 1
        set ntp-service local
        set default-gateway 192.168.188.1
        set netmask 255.255.255.0
        set interface "port10"
        config ip-range
            edit 1
                set start-ip 192.168.188.100
                set end-ip 192.168.188.150
            next
        end
    next
end
```
**Configurazione profilo VPN**

Passiamo ora alla configurazione del profilo VPN.  
Prima di tutto configuriamo la _phase1_ che contiene le informazioni necessarie affinché i due peer (initiator e responder) stabiliscano un tunnel sicuro.  
Le informazioni **devono** corrispondere per entrambi i peer.  

In questo laboratorio usiamo l'autenticazione basata su _pre-shared key_.  

Il valore SA (Security Association) della _phase1_ che prende il nome di IKE SA serve come scritto sopra per stabilire una sessione criptata per lo scambio dei valori SA della _phase2_, IPSEC SA, che contiene le informazioni per criptare e decriptare il traffico dati.  

Gli IKE SA sono rinegoziati in base al valore `keylife` che identifica in secondi la sua vita massima. Per quanto riguarda invece gli IPSEC SA, la vita massima può essere specificata in secondi, volume del traffico criptato/decriptato o entrambi.  

```
HQ # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "verso_Branch"
        set interface "port2"
        set local-gw 188.255.255.2
        set keylife 7200
        set peertype any
        set net-device enable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 88.255.255.2
        set psksecret chiave_segreta
    next
end
```
```
HQ # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "verso_Branch"
        set phase1name "verso_Branch"
        set proposal des-sha256
        set dhgrp 14
        set keylifeseconds 3600
        set src-subnet 192.168.188.0 255.255.255.0
        set dst-subnet 192.168.88.0 255.255.255.0
    next
end
```

**Configurazione rotte statiche**

Affinché il traffico fluisca tra le due subnet è necessario configurare manualmente le rotte statiche tra le estremità del tunnel (non necessario nel caso di topologie VPN mesh o che fanno uso di protocolli di routing dinamici).  

La **policy 3** è necessaria al fine di evitare che il traffico IPSEC utilizzi la rotta di default quando il tunnel non è attivo.

```
HQ # show router static
config router static
    [...]
    edit 2
        set dst 192.168.88.0 255.255.255.0
        set device "HQ"
    next
    edit 3
        set dst 192.168.88.0 255.255.255.0
        set distance 254
        set blackhole enable
    next
end
```
**Configurazione delle policy**

Quanto fatto finora è sufficiente affinché il tunnel funzioni ma senza le adeguate policy per il traffico dati, la comunicazione tra le subnet attraverso il tunnel è bloccato di default dal firewall: qualsiasi tentativo di comunicazione tra 192.168.188.0/24 e 192.168.88.0/24 è negato dalla policy implicita con id 0, **Implict deny**.

Procediamo quindi alla creazione di un oggetto/alias che identifica l'indirizzo/subnet a cui poi faremo riferimento.

```
HQ # show firewall address
config firewall address
[...]
    edit "HQ_192.168.188.0/24"
        set uuid a8ec56b2-4743-51ee-9f05-bd78f77fb087
        set subnet 192.168.188.0 255.255.255.0
    next
    edit "Branch_192.168.88.0/24"
        set uuid d0a44cdc-4743-51ee-1ebb-21e78a524cb4
        set subnet 192.168.88.0 255.255.255.0
    next
end
```

Aggiungiamo gli oggetti a dei gruppi in modo da rendere più agevole un'eventuale aggiunta o rimozione di subnet dalle policy.

```
HQ # show firewall addrgrp
config firewall addrgrp
    [...]
    edit "HQ_subnet"
        set uuid 9fc182a0-4744-51ee-6f4c-851f9900c966
        set member "HQ_192.168.188.0/24"
    next
    edit "Branch_subnet"
        set uuid b0d813d8-4744-51ee-6fa3-105650cc647b
        set member "Branch_192.168.88.0/24"
    next
end
```

Configuriamo infine le policy di **inbound** (ingresso) e **outbound** (uscita).  

La policy 2 corrisponde al traffico in ingresso dal tunnel IPsec e diretto verso la LAN, mentre la policy 3 corrisponde al traffico dalla LAN al tunnel IPSec.

```
HQ # show firewall policy
config firewall policy
    edit 2
        set name "Accetta_TrafficoVersoBranch"
        set uuid 6d5c26f0-47dd-51ee-b724-d9369ecfea08
        set srcintf "port10"
        set dstintf "verso_Branch"
        set action accept
        set srcaddr "HQ_subnet"
        set dstaddr "Branch_subnet"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 3
        set name "Accetta_TrafficoDaBranch"
        set uuid eea71ff8-47dd-51ee-9336-6b6219b8d58b
        set srcintf "verso_Branch"
        set dstintf "port10"
        set action accept
        set srcaddr "Branch_subnet"
        set dstaddr "HQ_subnet"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

## Configurazione firewall "Branch"

Per la configurazione del FortiGate "Branch", ripetiamo gli stessi passaggi.

**Configurazione interfacce di rete**

```
Branch # show system interface port2
config system interface
    edit "port2"
        set vdom "root"
        set ip 88.255.255.2 255.255.255.252
        set allowaccess ping
        set type physical
        set snmp-index 2
    next
end
```
```
Branch # show system interface port10
config system interface
    edit "port10"
        set vdom "root"
        set ip 192.168.88.1 255.255.255.0
        set allowaccess ping
        set type physical
        set snmp-index 10
    next
end
```

Configurazione DHCP per client della LAN 192.168.88.0/24.

```
Branch # show system dhcp server
config system dhcp server
    edit 1
        set ntp-service local
        set default-gateway 192.168.88.1
        set netmask 255.255.255.0
        set interface "port10"
        config ip-range
            edit 1
                set start-ip 192.168.88.100
                set end-ip 192.168.88.150
            next
        end
    next
end
```

**Configurazione profilo VPN**

```
Branch # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "verso_HQ"
        set interface "port2"
        set local-gw 88.255.255.2
        set keylife 7200
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 188.255.255.2
        set psksecret chiave_segreta
    next
end
```
```
Branch # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "verso_HQ"
        set phase1name "verso_HQ"
        set proposal des-sha256
        set dhgrp 14
        set keylifeseconds 3600
    next
end
```

**Configurazione oggetti e gruppi**

```
Branch # show firewall address
config firewall address
    [...]
    edit "Branch_192.168.88.0/24"
        set uuid e57546ea-4746-51ee-7acf-7f20168f6d43
        set subnet 192.168.88.0 255.255.255.0
    next
    edit "HQ_192.168.188.0/24"
        set uuid fcb6a01a-4746-51ee-4b22-88548a9c538b
        set subnet 192.168.188.0 255.255.255.0
    next
end
```
```
Branch # show firewall addrgrp
config firewall addrgrp
    [...]
    edit "HQ_subnet"
        set uuid 73307f7c-4747-51ee-61b2-f9772ab9820c
        set member "HQ_192.168.188.0/24"
    next
    edit "Branch_subnet"
        set uuid 7c84da1e-4747-51ee-b4d0-d96101764615
        set member "Branch_192.168.88.0/24"
    next
end
```

**Creazione policy per traffico inbound e outbound**

```
Branch # show firewall policy
config firewall policy
    edit 2
        set name "Accetta_TrafficoVersoHQ"
        set uuid afdbc77e-4747-51ee-c479-6353a7b69b64
        set srcintf "port10"
        set dstintf "verso_HQ"
        set action accept
        set srcaddr "Branch_subnet"
        set dstaddr "HQ_subnet"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 3
        set name "Accetta_TrafficoDaHQ"
        set uuid f9ded03c-4747-51ee-28da-7d7cec794fff
        set srcintf "verso_HQ"
        set dstintf "port10"
        set action accept
        set srcaddr "HQ_subnet"
        set dstaddr "Branch_subnet"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

**Configurazione rotte statiche**

```
Branch # show router static
config router static
    [...]
    edit 2
        set dst 192.168.188.0 255.255.255.0
        set device "verso_HQ"
    next
    edit 3
        set dst 192.168.188.0 255.255.255.0
        set distance 254
        set blackhole enable
    next
end
```

## Verifica di funzionamento tra i due endpoint

**Ping da client connesso a HQ a client connesso a Branch**
```
ipterm-1 console is now available... Press RETURN to get started.
udhcpc: started, v1.30.1
udhcpc: sending discover
udhcpc: sending select for 192.168.188.102
udhcpc: lease of 192.168.188.102 obtained, lease time 604800
root@ipterm-1:~# ping 192.168.88.102 -c3
PING 192.168.88.102 (192.168.88.102) 56(84) bytes of data.
64 bytes from 192.168.88.102: icmp_seq=2 ttl=62 time=20.3 ms
64 bytes from 192.168.88.102: icmp_seq=3 ttl=62 time=23.1 ms

--- 192.168.88.102 ping statistics ---
3 packets transmitted, 2 received, 33.3333% packet loss, time 2031ms
rtt min/avg/max/mdev = 20.261/21.665/23.069/1.404 ms
```

**Ping da client connesso a Branch a client connesso a HQ**
```
ipterm-2 console is now available... Press RETURN to get started.
udhcpc: started, v1.30.1
udhcpc: sending discover
udhcpc: sending select for 192.168.88.102
udhcpc: lease of 192.168.88.102 obtained, lease time 604800
root@ipterm-2:~# ping 192.168.188.102 -c2
PING 192.168.188.102 (192.168.188.102) 56(84) bytes of data.
64 bytes from 192.168.188.102: icmp_seq=1 ttl=62 time=18.2 ms
64 bytes from 192.168.188.102: icmp_seq=2 ttl=62 time=17.6 ms

--- 192.168.188.102 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 17.596/17.891/18.187/0.295 ms
```

## Da GUI come si vede quanto configurato da CLI?

Di seguito esploriamo come vediamo da interfaccia Web quanto abbiamo configurato tramite riga di comando.  

Prendiamo in esame il FortiGate HQ.

Prima di tutto vediamo le policy inbound e outbound, e la policy di default che nega tutto il traffico.  

<img src="/assets/fortinet/site2site-psk/policy.JPG" alt="Policy lab" width="95%" height="auto">

Qui invece vediamo gli _indirizzi_ e i _gruppi di indirizzi_.  

<img src="/assets/fortinet/site2site-psk/oggetti+gruppi.JPG" alt="Indirizzi e gruppi lab" width="95%" height="auto">

Le rotte statiche.

<img src="/assets/fortinet/site2site-psk/rotte.JPG" alt="Rotte lab" width="95%" height="auto">

Il tunnel IPSec e la sua configurazione.  
Cliccando il pulsante **Edit** è possibile modificare la configurazione.  

<img src="/assets/fortinet/site2site-psk/tunnel1.JPG" alt="Tunnel1 lab" width="95%" height="auto">  

<img src="/assets/fortinet/site2site-psk/tunnel2.JPG" alt="Tunnel2 lab" width="95%" height="auto">

Elenco degli eventi relativi alle VPN (troubleshoot) accessibile da **Log & report** >> **Events** >> **VPN Events**.  

<img src="/assets/fortinet/site2site-psk/eventi-vpn.JPG" alt="Eventi VPN lab" width="95%" height="auto">


## Comandi per il troubleshooting

Oltre a quanto visibile da GUI (eventi, traffico permesso e bloccato, policy lookup e route lookup...) da CLI sono utili i comandi `diagnose vpn ipsec status`, `diagnose vpn tunnel list`, `diagnose vpn ike gateway list`, `diagnose debug application ike`, `diagnose firewall iprobe`, `get router info routing-table all`.