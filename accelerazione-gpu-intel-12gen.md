# ARM: accelerazione grafica su Intel di 12a generazione (i5-12600K)

Guida per attivare la compressione hardware in Automatic Ripping Machine (ARM) con la GPU integrata di un Intel Core i5-12600K. Vale in generale per le CPU Intel di 12a generazione.

Per le CPU Intel di 6a generazione (come l'i5-6500 dell'HP EliteDesk 800 G3 usato nel video) c'è una guida a parte: [accelerazione-gpu-intel-6gen.md](accelerazione-gpu-intel-6gen.md).

## Perché jellyfin-ffmpeg

La versione di ffmpeg integrata in ARM, anche aggiungendo i driver Intel, è troppo vecchia e corrompe i file HEVC sulle CPU Intel di 12a generazione.

La soluzione è usare `jellyfin-ffmpeg`. Porta con sé la propria libva e il driver iHD, è pensato proprio per VAAPI/QSV su Intel e non tocca le librerie di sistema dell'immagine di ARM.

## 1. Immagine Docker personalizzata

Si parte dall'immagine ufficiale di ARM e si aggiunge jellyfin-ffmpeg:

```bash
mkdir -p ~/arm-build && cd ~/arm-build
cat > Dockerfile <<'EOF'
FROM automaticrippingmachine/automatic-ripping-machine:latest
RUN apt-get update && apt-get install -y curl gpg ca-certificates \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://repo.jellyfin.org/ubuntu/jellyfin_team.gpg.key \
       | gpg --dearmor -o /etc/apt/keyrings/jellyfin.gpg \
    && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/jellyfin.gpg] https://repo.jellyfin.org/ubuntu jammy main" \
       > /etc/apt/sources.list.d/jellyfin.list \
    && apt-get update && apt-get install -y --no-install-recommends jellyfin-ffmpeg7 \
    && rm -rf /var/lib/apt/lists/*
EOF
```

Poi si costruisce l'immagine, chiamandola `arm-vaapi`:

```bash
sudo docker build --pull -t arm-vaapi .
```

## 2. Script di avvio

Si apre lo script di avvio creato dall'installazione di ARM:

```bash
sudo nano /home/arm/start_arm_container.sh
```

L'unica modifica rispetto alla configurazione standard è il nome dell'immagine, in fondo: al posto di quella ufficiale va `arm-vaapi`. Questo è il mio script completo come esempio:

```bash
#!/bin/bash
docker run -d \
    -p "8080:8080" \
    -e ARM_UID="1001" \
    -e ARM_GID="1001" \
    -e TZ="Europe/Rome" \
    -v "/home/arm:/home/arm" \
    -v "/home/arm/music:/home/arm/music" \
    -v "/home/arm/logs:/home/arm/logs" \
    -v "/home/arm/media:/home/arm/media" \
    -v "/mnt/truenas/media/Film:/home/arm/media/completed/movies" \
    -v "/mnt/truenas/media/Serie TV:/home/arm/media/completed/tv" \
    -v "/home/arm/config:/etc/arm/config" \
    --privileged \
    --restart "always" \
    --name "arm" \
    --cpuset-cpus='1-3' \
    arm-vaapi
```

I volumi verso `/mnt/truenas/...` sono specifici del mio server. Cambiateli con le vostre cartelle di destinazione, come spiegato nella guida alle [personalizzazioni di ARM](personalizzazioni-arm.md).

## 3. Ricreare il container

```bash
sudo docker rm -f arm && sudo /home/arm/start_arm_container.sh
```

## 4. Impostazioni di ARM

Si va in Settings, poi nel tab Ripper settings. In alternativa si modifica direttamente `/home/arm/config/arm.yaml`.

```yaml
USE_FFMPEG: true
FFMPEG_CLI: /usr/lib/jellyfin-ffmpeg/ffmpeg
FFMPEG_LOCAL: /usr/lib/jellyfin-ffmpeg/ffmpeg
FFMPEG_PRE_FILE_ARGS: -vaapi_device /dev/dri/renderD128
FFMPEG_POST_FILE_ARGS: -vf format=nv12,hwupload,deinterlace_vaapi=auto=1 -c:v hevc_vaapi -qp 22 -map 0:v:0 -map 0:a? -map 0:s? -map_chapters 0 -c:a copy -c:s copy -disposition:a 0 -disposition:a:0 default -disposition:s 0
```

In parole: compressione HEVC con la GPU a qualità 22, deinterlacciamento dei DVD, audio e sottotitoli copiati senza ricomprimerli, traccia audio italiana come predefinita e sottotitoli spenti di default.

## 5. MakeMKV: tenere solo italiano e inglese

Qui il filtro sulle lingue lo fa MakeMKV in fase di copia del disco, quindi ffmpeg riceve già solo le tracce giuste. Questi comandi aggiungono la regola di selezione al file di configurazione di MakeMKV:

```bash
sudo sh -c 'echo "app_DefaultSelectionString = \"-sel:all,+sel:(ita|eng|nolang),-sel:(core),-15:(ita)\"" >> /home/arm/.MakeMKV/settings.conf'
sudo chown 1001:1001 /home/arm/.MakeMKV/settings.conf
sudo cat /home/arm/.MakeMKV/settings.conf
```

L'ultimo comando serve a controllare che la riga sia stata scritta. Il file `settings.conf` esiste solo dopo il primo avvio di ARM.

Cosa fa ogni parte della stringa:

* `-sel:all` deseleziona tutto.
* `+sel:(ita|eng|nolang)` riseleziona italiano, inglese e le tracce senza lingua. `nolang` serve per il video, che di solito non ha un tag di lingua, e per eventuali tracce non etichettate.
* `-sel:(core)` scarta il "core" che MakeMKV estrae dalle tracce HD dei Blu-ray (TrueHD, DTS-HD): si tiene solo la traccia completa.
* `-15:(ita)` sposta le tracce italiane davanti, così `-disposition:a:0 default` in ffmpeg marca come predefinito l'audio italiano.

## Prossimi passi

* Impostazioni generiche di ARM (chiave OMDB, durata minima, lingue di HandBrake): [personalizzazioni-arm.md](personalizzazioni-arm.md)
* Blu-ray che si bloccano con MakeMKV: [fix-makemkv-bluray.md](fix-makemkv-bluray.md)
* Jellyfin: [installazione-jellyfin.md](installazione-jellyfin.md)
