---
title: Come configurare il parsing dei log Fortigate in Graylog
subtitle: MOMENTACT Edition
layout: post
background: '/img/bg-post.jpg'
categories: networking fortigate log
---

In un articolo precedente avevo accennato a un setup in cui una istanza Graylog riceve e rielabora log generati da firewall Fortinet, in questo articolo vediamo come funziona!

#### Fonti:
- [**Extractors for Graylog 5.0**](https://github.com/alias454/graylog-fortinet-content-pack/issues/4)
- [**Extractors per Graylog 5.0 - mantenuto e modificato da me**](https://github.com/jupitersinsight/graylog-extractors/blob/main/Fortinet.Extractor.txt)

#### Prerequisiti
- Una istanza di Graylog 5.0 attiva
- Log FortiGate a disposizione


#### Step:
- Creazione di un Input
- Importazione Extractors
- Creazione di Streams

Iniziamo!

### Creazione di un Input

Come suggerisce il nome, in Graylog gli Input non sono altro che dei listener in ascolto su indirizzi IP e porte specificati dall'amministratore. 
Ogni Input è legato al tipo di comunicazione che ci si aspetta, ad esempio *TCP Raw/Plaintext*, *Syslog su UDP*, *Beats*.

Nel nostro caso creiamo l'Input come **Syslog TCP** configurando l'indirizzo IP (0.0.0.0 se si vuole attivare l'input su tutti i segmenti di rete configurati nel server), la porta di ascolto, eventuali certificati per la crittografia del flusso TCP e infine spuntando la voce *null frame delimiter?*.  

<img src="/assets/graylog/input_1.JPG" width="65%" height="auto">

Nel momento in cui Graylog riceve dati, ad esempio da una istanza di rsyslog, i contatori delle connessioni attive e delle operazioni I/O mostrano le statistiche sul flusso di dati.  

<img src="/assets/graylog/input_2.JPG" width="50%" height="auto">

Cliccando il pulsante `Show received messages` è possibile vedere in tempo reale i messaggi ricevuti ma... che orrore!

```
<network blob> date=2024-01-20,time=<time>,devname="<devname>",devid="<devid>",logid="<logid>",type="utm",subtype="dns",eventtype="dns-response",level="information",vd="root",eventtime=<event-time>,tz="+0100",policyid=26,sessionid=338799,srcip=<source_ip>,srcport=49276,srcintf="<source_interface>",srcintfrole="lan",dstip=<destination_ip>,dstport=53,dstintf="<destination-interface>",dstintfrole="wan",proto=17,profile="<profile>",srcmac="<source_mac>",xid=1,qname="<qname>",qtype="A",qtypeval=1,qclass="IN",ipaddr="<ip_address>",msg="Domain was allowed because it is in the domain-filter list",action="pass",domainfilteridx=1,domainfilterlist="<domain_lilst>"
```
Se lasciassimo i messaggi inalterati, senza un adeguato parsing, sarebbe impossibile effettuare ricerche e tantomento creare dashboard personalizzate.  

### Importazione Extractors

Graylog mette a disposizione la creazione di Extractors che altro non sono che regex applicate per singolo Input le quali ci aiutano a creare nuovi campi a partire dal valore estratto.  
Nel nostro caso, il risultato finale sarà:

<img src="/assets/graylog/extractor_1.JPG" width="23%" height="auto">

Accanto al pulsante `Show received messages` si trova il pulsante `Manage Extractors` che ci permette di gestire gli extractor esistenti e di crearne di nuovi: `Create extractor` -> `Load Message` -> individuare la porzione del messaggio su cui applicare l'extractor, ad esempio `message` -> scegliere il tipo di extractor, nel nostro caso `Regular expression`.  

A questo punto nel campo `Regular expression` inseriamo la regex, ad esempio `type=\"?([a-z]+)", e poi inseriamo il nome del nuovo campo, nominiamo l'extractor e salviamo.  

Tra le fonti ho inserito due link, il secondo dei quali è mantenuto da me, dai quali è possibile scaricare un file .txt che contiene già numerosi extractor che possono essere importati da `Manage Extractors` -> 	`Actions` -> `Import extractors`.  

**Importante**, gli extractor si applicano **solo** ai nuovi messaggi!.

### Creazione di Streams

A questo punto, avendo a disposizione una potente arma per creare nuovi campi dai nostri log, possiamo sfruttare gli Streams di Graylog che ci permottono di reindirizzare i messaggi ricevuti in indici (ad esempio `fortigate.vpn.ipsec.failed`) sulla base di match su coppie campo/valore.  

Per procedere alla creazione di uno Stream: `Streams` -> `Create stream`.  
Alla voce `Index Set` è possibile scegliere un indice creato in precedenza e di evitare che i messaggi siano memorizzati anche nello stream `Default Stream`.  

Cliccando il pulsante `Manage Rules` e poi `Add Stream rule` è possibile creare una o più regole che facciano match sui messaggi (una sola o tutte).  

<img src="/assets/graylog/stream_1.JPG" width="80%" height="auto">

La creazione di Stream, oltre a permettere il salvataggio dei messaggi in indici dedicati, permette anche di snellire le ricerche dall'explorer che si apre da `Show received messages` dell'input di riferimento.  

<img src="/assets/graylog/stream_2.JPG" width="80%" height="auto">