---
title: Come configurare l'invio di log da Fortigate a server Syslog
subtitle: TLS-Edition
layout: post
background: '/img/bg-post.jpg'
categories: networking fortigate log
---

In questo articolo vedremo come configurare l'invio di Log da un Firewall Fortinet a un server Syslog remoto usando dei certificati per la protezione delle informazioni.

#### Fonti:
- [**Technical Tip: Send Syslog over TLS to a rsyslog server**](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Send-Syslog-over-TLS-to-a-rsyslog-server/ta-p/248101)
- [**XCA - X Certificate and Key management**](https://hohnstaedt.de/xca-doc/html/index.html)
- [**FortiOS CLI Reference**](https://docs.fortinet.com/document/fortigate/6.0.0/cli-reference/260508/log-syslogd-syslogd2-syslogd3-syslogd4-setting)
- [**Fortinet GURU - Certificates overview**](https://www.fortinetguru.com/2019/05/certificates-overview/)
- [**Fortigate syslog and TLS**](https://www.reddit.com/r/fortinet/comments/139a92p/fortigate_syslog_and_tls/?rdt=46970)  
<br>  

#### Prerequisiti:
- Firewall Fortinet fisico o VM con licenza
- Un server con rsyslog installato
- Un indirizzo IP Pubblico o DNS Dinamico
- Port-forwarding della porta di ricezione  
<br>

#### Step:
- Creazione dei certificati usando XCA
- Preparazione del FortiGate
- Preparazione di RSYSLOG
- Comandi di troubleshooting


Iniziamo!
<br>
### Creazione dei certificati

Prima di tutto generiamo i nostri certificati *self signed* usando XCA.  
All'apertura del programma generiamo subito un database: `File` -> `Nuovo database`.   
Prima di concludere la creazione del database, XCA ci chiede di inserire una password che sarà poi usata per proteggere le chiavi private generate.

<img src="/assets/xca/psw-prompt.JPG">

Procediamo quindi alla creazione del certificato CA: `Certificati` -> `Nuovo certificato`.  
Scegliamo il modello del nuovo certificato, *[default]CA* e poi applichimo le estensioni. 

<img src="/assets/xca/ca_1.JPG" width="95%" height="auto">

Compiliamo poi i campi secondo necessità e infine generiamo una chiave.  

<img src="/assets/xca/ca_2.JPG" width="95%" height="auto">

Nella tab estensioni è possibile configurare altri valori tra cui la durata di validità del certificato.

A questo punto creiamo il certificato del server: `Selezioniamo` il certificato CA -> `Click destro` -> `Nuovo certificato`.  
XCA già propone di firmare il nuovo certificato usato il certificato CA, scegliamo come modello *[default] TLS_server*.  

Ancora una volta applichiamo le estensioni e poi ci spostiamo nella tab *Soggetto*. Anche qui compiliamo secondo necessità e generiamo una nuova chiave.  
Apportiamo modifiche anche alla tab *Estensioni*, se necessario, con attenzione al campo *X509v3 Subject Alternative Name* nel quale è possibile inserire informazioni che sono poi associate al certificato (ad esempio un indirizzo email).

<img src="/assets/xca/srv_1.JPG" width="95%" height="auto">

A questo punto bisogna esportare i certificati: `Selezionare` un certificato -> `Click destro` -> `File`.  
Il certificato CA deve essere esportato come file *PEM* mentre il certificato del server come *PEM + key*.

Il certificato host, *.pem*, dovrebbe contenere sia il certificato sia la chiave nel formato:
```
-----BEGIN CERTIFICATE-----
>>>> CERTIFICATO <<<
-----END CERTIFICATE-----

-----BEGIN RSA PRIVATE KEY-----
>>>> CHIAVE <<<
-----END RSA PRIVATE KEY-----
```

La sezione del certificato è da copiare in un file *.pem* (ad esempio, *certificato_host.pem*) mentre la sezione della chiave è da copiare in un file *.key* (ad esempio, *chiave_host.key*).

Se, invece, aprendo il certificato del server con un editor di testo, come [**NotePad++**](https://notepad-plus-plus.org/downloads/), doveste notare che manca la sezione della chiave, è necessario esportarla manualmente da XCA.  
Spostandosi nella tab *Chiavi private* è possibile ripetere l'operazione come sopra ma per esportare la chiave, la quale deve essere in formato *PEM private*. 

I file che dovremmo avere alla fine delle operazioni sono:
- **Certificato CA (con estensione *.crt* o *.pem*)**
- **Certificato del Server (con estensione *.pem*)**
- **Chiave privata del certificato del Server (con estensione *.key*)**

### Preparazione del FortiGate

Prima di tutto carichiamo il certificato CA nel Fortigate (via GUI è più semplice): `System` -> `Certificates` -> `Create/Import` -> `CA Certificate` -> `File` -> `Upload`.  
Il certificato è caricato come *Remote CA Certificate* e il FortiGate si occuperà da solo di utilizzarlo nell'handshake TLS. 

Nei FortiGate è possibile configurare fino a 4 istanze syslog ed è altamente consigliato eseguire la configurazione tramite CLI (via SSH, Jconsole o come preferite).  

Al comando `config log syslogd settings` inseriamo la configurazione per la prima istanza syslogd (o la prima libera):
```
set status enable
set server <IP o FQDN>
set mode reliable
set port <Porta remota>
set enc-algorithm high
set format csv
```

Mentre al comando `config log syslogd filter` inseriamo la configurazione: 
```
set severity information
```

A questo punto il FortiGate inizierà a inviare i log alla destinazione indicata in configurazione.

### Preparazione del Server RSYSLOG

Per configurare RSYSLOG è necessario modificare il file *rsyslog.conf* al percorso */etc/rsyslog.conf* ed è sufficiente un editor qualiasi come *nano*, *vi* o *vim*.

La configurazione è:
```
global(
DefaultNetstreamDriver="gtls"
DefaultNetstreamDriverCAFile="/home/syslog/<Certificato_CA.pem>"
DefaultNetstreamDriverCertFile="/home/syslog/<Certificato_SERVER>.pem"
DefaultNetstreamDriverKeyFile="/home/syslog/<Chiave_SERVER>.key"
)

module(
load="imtcp"
StreamDriver.Name="gtls"
StreamDriver.Mode="1"
StreamDriver.Authmode="anon"
)

input(
type="imtcp"
port="<Porta in ascolto>"
)

action(
type="omfwd"
target="<IP o FQDN>"
port="<Porta remota>"
protocol="tcp"
tcp_framing="octet-counted"
)
```

Questo esempio di configurazione apre la porta TCP specificata in ascolto per connessioni e tramite il driver *gtls* effettua l'handshake TLS.  
Nella parte finale è descritta una *action* che permette di inoltrare a un'altro servizio i log ricevuti. 

Da notare che l'opzione **[tcp_framing="octet-counted"](https://www.rsyslog.com/doc/configuration/modules/omfwd.html?highlight=tcp_framing)** è necessaria quando in FortiGate si imposta `mode reliable`.  
Questa opzione infatti consente a RSYSLOG di suddivedere i pacchetti contenute nel flusso TCP non in base a un delimiter come **LF**, ma in base al campo specificato a inizio pacchetto che contiene appunto la lunghezza dello stesso.

Nel mio caso, l'inoltro avviene verso una istanza di **Graylog** che si occupa di effettuare il parsing dei log, suddividerli in indici (in **Wazuh-Indexer**) e di arricchirli con informazioni di geolocalizzazione per creare mappe interattive in **Grafana**.

A questo punto anche RSYSLOG è configurato. Per rendere effettive le modifiche occorre riavviare il servizio col comando `systemctl restart rsyslog`.

### Troubleshooting

**FortiGate**  
In FortiGate è possibile avviare una Packet Capture, filtrando ad esempio sulla porta di destinazione o per host, e analizzare successivamente il *.pcap* generato.  
I possibli errori sono:
- **Handshake TLS non va a buon fine**: controllare che i certificati siano corretti
- **TCP Syn da parte del FortiGate riceve Reset-Ack come risposta**: l'indirizzo IP o la porta remoti sono errati (porta remota chiusa)

**RSYSLOG**  
In quanto RSYSLOG è indirizzato a sistemi Linux i comandi sono sicuramente noti e usati spesso per troubleshooting di comunicazioni di rete.  
Per verificare se l'istanza di RSYSLOG sia in ascolto su indirizzo e porta corretti: `netstat -tupln`.  
Per verificare il traffico in arrivo: `tcpdump -i <interfaccia> -A -vv 'tcp port <porta>'`. Questo comando stampa a video il contenuto in ASCII dei pacchetti in arrivo alla porta TCP specificata sull'interfaccia specificata:
- **Se non arrivano pacchetti**: verificare la configurazione del FortiGate, di eventuali router nel mezzo e di RSYSLOG
- **Se arrivano pacchetti e il contenuto è leggibile**: avete configurato la ricezione in RSYSLOG senza l'utilizzo di certificati

