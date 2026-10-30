# ARM: personalizzazioni generiche

Le modifiche da fare ad Automatic Ripping Machine (ARM) dopo l'installazione, valide qualunque sia l'hardware. Seguono l'ordine del video.

Per l'accelerazione grafica con la GPU integrata servono modifiche in più, che dipendono dalla generazione del processore Intel:

* CPU Intel di 6a generazione (come l'i5-6500 del video): [accelerazione-gpu-intel-6gen.md](accelerazione-gpu-intel-6gen.md)
* CPU Intel di 12a generazione (come l'i5-12600K): [accelerazione-gpu-intel-12gen.md](accelerazione-gpu-intel-12gen.md)

## 1. Script di avvio

Dopo l'installazione, nella cartella home dell'utente `arm` c'è lo script `start_arm_container.sh`. Si scopre prima l'ID di utente e gruppo:

```bash
id -u arm
id -g arm
```

Poi si creano le cartelle, assegnate subito all'utente giusto:

```bash
sudo -u arm mkdir -p /home/arm/logs /home/arm/music /home/arm/media /home/arm/config
```

Infine si modifica lo script:

```bash
sudo nano /home/arm/start_arm_container.sh
```

Cosa cambiare:

* `ARM_UID` e `ARM_GID`: i valori trovati prima. Sulla mia macchina sono 1001 e 1001.
* `TZ`: il fuso orario. Per l'Italia è `Europe/Rome`.
* I percorsi dei volumi di logs, media e così via: quelli delle cartelle appena create.
* Le righe `--device` dei lettori: si possono cancellare in blocco. Il container gira con `--privileged`, che si trova subito sotto, e vede comunque tutti i lettori.
* `--cpuset-cpus`: i core da assegnare al container. Il valore consigliato dalla guida è uno in meno dei core totali. Con una CPU a 4 core come l'i5-6500 va `1-3`, con 8 core andrebbe `1-7`.

Poi si avvia:

```bash
sudo /home/arm/start_arm_container.sh
```

Con `--restart always` ARM riparte da solo a ogni accensione, quindi lo script si lancia una volta sola.

## 2. Primo accesso

Si apre un browser e si va su `http://IP-DEL-PC:8080`. Al primo avvio si clicca su Login e si entra con username `admin` e password `password`. La prima cosa da fare è cambiare la password da "Change admin password".

## 3. Impostazioni principali

Si va in Settings, poi nel tab Ripper settings. Le stesse chiavi si possono modificare nel file `/home/arm/config/arm.yaml`.

**N.B.:** se nell'interfaccia web i parametri in Ripper settings non sono scrivibili, dovete sistemare i permessi da terminale con:
```bash
sudo chown -R 1001:1001 /home/arm/config
```

### Parametri da modificare

**OMDB_API_KEY**: la chiave API di OMDb. Serve ad ARM per recuperare in automatico titolo e anno di film e serie TV. La chiave è gratuita e si ottiene dal sito che compare lasciando per qualche istante il mouse sulla "i" blu a destra nella riga.

**MINLENGTH**: 3600. ARM prende in considerazione solo i titoli di almeno un'ora, evitando che scambi degli extra per il film vero e proprio. Per i dischi delle serie, o se volete copiare anche i contenuti speciali, si può abbassare a mano dal singolo job (vedi sezione 5).

**ALLOW_DUPLICATES**: `true`. Senza questo i dischi di una stessa stagione verrebbero considerati dei doppioni da ARM, che li salterebbe a piè pari.

**HB_ARGS_DVD** e **HB_ARGS_BD**: indicano ad ARM quali lingue tenere durante la copia. Con questa stringa si tengono solo italiano e inglese, sia per l'audio che per i sottotitoli:

```
--audio-lang-list ita,eng --all-audio --subtitle-lang-list ita,eng --all-subtitles
```

Nel valore di default di `HB_ARGS_BD` c'è `--subtitle-burned`, che imprime i sottotitoli nel video. Sostituendo l'intera stringa con quella qui sopra viene eliminato.

Se usate l'accelerazione grafica con ffmpeg (guide linkate in alto), le lingue vengono gestite da MakeMKV e da `FFMPEG_POST_FILE_ARGS`, quindi queste due chiavi non hanno effetto sulla compressione.

## 4. Blu-ray: dump del firmware

Se avete dei Blu-ray da digitalizzare serve un passaggio in più, una volta sola per lettore, prima di inserire il primo disco. Senza questo MakeMKV si blocca e il rip non parte mai. Tutti i dettagli sono in [fix-makemkv-bluray.md](fix-makemkv-bluray.md).

## 5. Interventi manuali sul singolo disco

ARM aspetta 60 secondi dopo l'inserimento del disco per dare il tempo di correggere a mano.

**Titolo non riconosciuto.** Durante i 60 secondi si clicca su Title Search, si inserisce titolo e anno del film in inglese, si preme Search, si seleziona quello giusto e si conferma. Il titolo in ARM resta in inglese, ma Jellyfin mostra poi la versione tradotta in italiano.

**Copiare anche gli extra.** Due opzioni:

* abbassare a 0 `MINLENGTH` nelle impostazioni globali, che però vale per ogni disco;
* oppure farlo solo per il disco che interessa: durante i 60 secondi si clicca su "Edit Settings" del job, si imposta Minimum Length a 0 e si clicca su Submit.

**Scegliere a mano i titoli da copiare.** Prima di inserire il disco, in Settings si imposta Manual sotto Drive Mode del lettore e si clicca su Update. ARM non avvia la copia, scansiona i titoli e si mette in attesa. Durante i 60 secondi si imposta Minimum Length a 0 come sopra per farli trovare tutti. A scansione finita, cliccando sulla copertina del film si entra nel dettaglio del job e in fondo alla pagina c'è l'elenco dei titoli: si mette la spunta su quelli che servono, si conferma e ARM riprende a lavorare.

Lo stesso procedimento vale per le serie TV, perché i loro dischi contengono episodi, extra e a volte un titolo unico che li raggruppa tutti.

**Limite sulle serie TV.** ARM non riconosce da solo stagione ed episodi, quindi non può dare ai file il nome corretto, e non c'è modo di indicarglieli a mano. Per Jellyfin i file vanno rinominati a mano.

## 6. Libreria su un disco o su un NAS

Per salvare i file finiti altrove si aggiunge un volume allo script di avvio. Il percorso a destra deve corrispondere alla cartella media di Jellyfin, come spiegato in [installazione-jellyfin.md](installazione-jellyfin.md). Esempio:

```bash
-v "/srv/media:/home/arm/media/completed"
```

Poi si ricrea il container:

```bash
sudo docker rm -f arm && sudo /home/arm/start_arm_container.sh
```
