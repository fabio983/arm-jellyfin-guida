# ARM: accelerazione grafica su Intel di 6a generazione (i5-6500)

Guida per attivare la compressione hardware in Automatic Ripping Machine (ARM) con la GPU integrata di un Intel Core i5-6500, quella dell'HP EliteDesk 800 G3 usato nel video. Vale per tutte le CPU Intel di quella generazione (Skylake).

Sulle CPU Intel di 12a generazione il procedimento è diverso, serve un ffmpeg più recente: [accelerazione-gpu-intel-12gen.md](accelerazione-gpu-intel-12gen.md).

## Come funziona

Su questa generazione la compressione hardware con HandBrake QSV dentro ARM non funziona. La soluzione è aggiungere il driver video Intel all'immagine di ARM e usare ffmpeg con VAAPI al posto di HandBrake (`USE_FFMPEG: true`).

## 1. Immagine Docker personalizzata

Tre righe di Dockerfile partendo dall'immagine ufficiale di ARM:

```bash
mkdir -p ~/arm-build && cd ~/arm-build
cat > Dockerfile <<'EOF'
FROM automaticrippingmachine/automatic-ripping-machine:latest
RUN apt-get update && apt-get install -y intel-media-va-driver-non-free \
    && rm -rf /var/lib/apt/lists/*
EOF
sudo docker build --pull -t arm-vaapi .
```

## 2. Script di avvio

Si apre lo script di avvio creato dall'installazione di ARM:

```bash
sudo nano /home/arm/start_arm_container.sh
```

L'unica modifica rispetto alla configurazione standard è il nome dell'immagine, in fondo: al posto di quella ufficiale va `arm-vaapi`.

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
    -v "/home/arm/config:/etc/arm/config" \
    --privileged \
    --restart "always" \
    --name "arm" \
    --cpuset-cpus='1-3' \
    arm-vaapi
```

I valori di `ARM_UID`, `ARM_GID` e i percorsi dei volumi vanno adattati alla vostra macchina, come spiegato in [personalizzazioni-arm.md](personalizzazioni-arm.md). Se avete già avviato ARM una volta, ricreate il container:

```bash
sudo docker rm -f arm && sudo /home/arm/start_arm_container.sh
```

## 3. Impostazioni di ARM

Si va in Settings, poi nel tab Ripper settings. In alternativa si modifica direttamente `/home/arm/config/arm.yaml`.

```yaml
USE_FFMPEG: true
FFMPEG_CLI: ffmpeg
FFMPEG_LOCAL: ffmpeg
DEST_EXT: mkv
FFMPEG_PRE_FILE_ARGS: "-vaapi_device /dev/dri/renderD128"
FFMPEG_POST_FILE_ARGS: "-vf format=nv12,hwupload,deinterlace_vaapi=auto=1 -c:v hevc_vaapi -qp 22 -map 0:v:0 -map 0:a:m:language:ita? -map 0:a:m:language:eng? -map 0:s:m:language:ita? -map 0:s:m:language:eng? -map_chapters 0 -c:a copy -c:s copy -disposition:a 0 -disposition:a:0 default -disposition:s 0"
```

`FFMPEG_CLI` e `FFMPEG_LOCAL` vanno scritti in minuscolo: il valore di default `FFMPEG` non funziona.

In parole: compressione HEVC con la GPU a qualità 22, deinterlacciamento dei DVD, solo audio e sottotitoli in italiano e inglese con l'italiano come traccia principale, sottotitoli spenti di default, audio copiato senza ricomprimerlo.

## 4. MakeMKV: tenere solo italiano e inglese

Questi comandi aggiungono la regola di selezione delle tracce al file di configurazione di MakeMKV, così la copia del disco scarta fin dall'inizio le lingue che non servono:

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
