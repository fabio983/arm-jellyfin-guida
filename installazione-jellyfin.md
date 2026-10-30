# Installazione di Jellyfin con Docker

Guida per installare Jellyfin con Docker, pensata per leggere i file digitalizzati da Automatic Ripping Machine (ARM). Jellyfin si può installare in molti altri modi, elencati nella documentazione ufficiale. Docker è il metodo più universale e facile da replicare.

## 1. Cartelle

```bash
sudo mkdir -p /opt/jellyfin/config /opt/jellyfin/cache
sudo chown -R 1000:1000 /opt/jellyfin
```

## 2. Gruppo della GPU

```bash
getent group render | cut -d: -f3
```

Il numero che esce serve per dare a Jellyfin l'accesso alla scheda grafica integrata.

## 3. File docker-compose

```bash
cd /opt/jellyfin
sudo nano docker-compose.yml
```

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 1000:1000
    group_add:
      - "RENDER_GID"
    network_mode: host
    volumes:
      - /opt/jellyfin/config:/config
      - /opt/jellyfin/cache:/cache
      - /srv/media:/media:ro
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    restart: unless-stopped
```

Cosa cambiare e perché:

* `RENDER_GID` va sostituito con il numero del passaggio 2.
* `/srv/media` è il disco con i film e le serie rippati da ARM, montato in sola lettura. Si può cambiare a piacere, anche facendolo puntare a un NAS.
* Se volete che i file finiti scritti da ARM vengano letti da Jellyfin senza fare nulla, la cartella `completed` di ARM e quella `media` di Jellyfin devono combaciare. Se usate `/srv/media` come sopra, nello script di avvio di ARM aggiungete `-v "/srv/media:/home/arm/media/completed"` e ricreate il container (vedi [personalizzazioni-arm.md](personalizzazioni-arm.md)). Da lì Jellyfin trova le cartelle `movies` e `tv`.
* `network_mode: host` serve perché le app di Jellyfin trovino il server da sole in rete.

Riferimenti: guida ufficiale "Container" e pagina "Hardware Acceleration, Intel" della documentazione di Jellyfin.

## 4. Avvio

```bash
sudo docker compose up -d
sudo docker ps
```

Il sistema scarica in automatico l'ultima versione di Jellyfin e avvia il servizio.

## 5. Configurazione iniziale

Si fa da browser all'indirizzo `http://IP-DEL-SERVER:8096`. La procedura guidata chiede poche cose:

* lingua e creazione dell'utente amministratore;
* le librerie: una per i film e una per le serie TV. Si clicca su "Aggiungi libreria multimediale", si sceglie il tipo di contenuto e nella sezione "Cartelle" si clicca sul `+` per selezionare la cartella, per esempio `/media/movies/` per i film;
* lingua italiana e paese Italia, sia per la libreria che per i metadati. Si ripete lo stesso per le serie TV.

## 6. Accelerazione hardware

Dal Pannello di Controllo, in alto a destra, tab Riproduzione, sezione Transcodifica. Le voci dipendono dalla scheda grafica. Con una GPU Intel integrata:

* accelerazione hardware: Intel QuickSync (QSV);
* dispositivo: `/dev/dri/renderD128`;
* formati: si selezionano quelli supportati dalla scheda nell'elenco sottostante.

## 7. Lingua di riproduzione

Nelle preferenze utente, sezione Profilo, si imposta la lingua di riproduzione preferita su italiano, e lo stesso per i sottotitoli. Jellyfin sceglie così la traccia giusta da solo, a prescindere da quale sia marcata come predefinita nel file.
