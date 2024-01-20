---
title: Geolocalizzare indirizzi IP in Graylog
subtitle: "I don't know where you are but NOW I will find you"
layout: post
background: '/img/bg-post.jpg'
categories: networking fortigate log
---

Che ne direste di geolocalizzare quell'indirizzo IP che tenta di accedere alla vostra VPN?

#### Fonti:
- [**How to set up Graylog GeoIP configuration**](https://graylog.org/post/how-to-set-up-graylog-geoip-configuration/)
- [**Geolocation - Graylog Documentation**](https://go2docs.graylog.org/5-1/making_sense_of_your_log_data/geolocation.html?tocpath=Sorting%20and%20Enriching%20Logs%7CProcessing%20Pipelines%7CLog%20Enrichment%7C_____2)
- [**GeoLite2 Free Geolocation Data**](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data)
- [**GeoIP Update**](https://dev.maxmind.com/geoip/updating-databases#using-geoip-update)

#### Prerequisiti
- Una istanza di Graylog 5.0 attiva
- Log FortiGate a disposizione


#### Step:
- Preparazione delle risorse GeoIP di MaxMind
- Configurazione di Graylog

Iniziamo!

### Preparazione delle risorse GeoIP di MaxMind

Come primo step procediamo alla registrazione di un account al sito [maxmind.com](https://www.maxmind.com/en/home) e [generiamo una license key](https://www.maxmind.com/en/accounts/current/license-key/) che ci servirà poi per autorizzare l'aggiornamento automatico dei database.  

Dalla propria area riservata è possibile scaricare i database al [link](https://www.maxmind.com/en/accounts/current/geoip/downloads) che andranno copiati al percorso `/etc/graylog/server/`.

A questo punto procediamo all'installazione di `GeoIP Update`:
```
$ sudo add-apt-repository ppa:maxmind/ppa

$ sudo apt update
$ sudo apt install geoipupdate
```

Sempre dalla propria area riservata è possibile scaricare un [file di configurazione generico](https://www.maxmind.com/en/accounts/current/license-key/GeoIP.conf) che andrà salvato come `GeoIP.conf` al percorso `/etc/GeoIP.conf`.  

All'interno del file di configurazione bisogna inserire la propia license key.  
Infine configuriamo un cron job che chiami in automatico geoipupdate, `crontab -e`:
```
33 4 * * 6,4 /usr/local/bin/geoipupdate
```

### Configurazione di Graylog

Il primo step prevede la creazione di un `Data Adapter`: `System` -> `Lookup Tables` -> `Data Adapters` -> `Create Data Adapter`.  

<img src="/assets/graylog/geoip_1.JPG" width="95%" height="auto">

Creiamo poi una `Cache`: `System` -> `Lookup Tables` -> `Caches` -> `Create Cache`.  

<img src="/assets/graylog/geoip_2.JPG" width="95%" height="auto">

Creiamo infine la `Lookup Table`: `System` -> `Lookup Tables` -> `Lookup Tables` -> `Create Lookup Table`.

<img src="/assets/graylog/geoip_3.JPG" width="95%" height="auto">

Giunti a questo punto creiamo una pipeline che processi i nostri log e faccia uso di quanto creato finora: `System` -> `Pipelines` -> `Manage rules`.  

Creaiamo la prima regola, `Create rule`, la nominiamo e inseriamo la seguente regola, appunto:
```
rule "GeoIP lookup: GeoIP"
when
 has_field("<campo da ricercare>")
then
 let geo = lookup("geoip", to_string($message.<campo da ricercare>));
 set_field("<campo da ricercare>_Coordinates", geo["coordinates"]);
 set_field("<campo da ricercare>_Country", geo["country"].iso_code);
 set_field("<campo da ricercare>_City", geo["city"].names.en);
end
```

Da notare che `<campo da ricercare>` può essere l'indirizzo IP sorgente o destinazione. Nel caso si vogliano geolocalizzare entrambi, si aggiunge una seconda condizione *when* oppure si crea una regola dedicata.  

Tornando nella sezione principale di `System/Pipelines`, aggiungiamo la regola allo `Stage`.  

<img src="/assets/graylog/geoip_4.JPG" width="75%" height="auto">

Cliccando `Edit Connections` è possibile specificare a quali Stream applicare la regola.

Se tutto funziona correttamente il risultato finale sarà qualcosa di simile:  

<img src="/assets/graylog/geoip_5.JPG" width="26%" height="auto">