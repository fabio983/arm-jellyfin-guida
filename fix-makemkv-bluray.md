# Fix per i Blu-ray bloccati con MakeMKV (ARM e non solo)

Con i Blu-ray, MakeMKV recente può bloccarsi senza dare nessun errore, e la copia del disco non parte mai. Il problema non è di ARM ma di MakeMKV stesso, quindi la soluzione vale anche se usate `makemkvcon` da soli, senza ARM.

Ho risolto con uno script, pubblicato su GitHub con guida in italiano e in inglese:

**[github.com/fabio983/makemkv-fw-dump](https://github.com/fabio983/makemkv-fw-dump)**

## Sintomi

* In ARM, il log del job si ferma subito dopo `Starting MakeMKV rip`.
* `makemkvcon` resta al 100% di CPU e non finisce mai.
* Il lettore non gira e non legge.
* Lanciando `makemkvcon` a mano compare solo `Downloading latest SDF ...`.
* I DVD funzionano di solito, i Blu-ray no.

## Causa

Con le versioni recenti di MakeMKV (dalla 1.17.7 in poi, quindi anche la 1.18.4 che arriva dentro ARM) manca il dump del firmware del lettore, il file `dump_FW_<ID>.tgz` nella cartella `.MakeMKV`. Le versioni più vecchie lo creano da sole alla prima lettura di un disco. Quelle recenti no, e restano in attesa all'infinito. Il problema è tracciato nella issue #1545 del progetto ARM.

Lo script fa quella lettura al posto vostro. Costruisce MakeMKV 1.17.6 dentro un container Docker temporaneo, legge un disco, recupera il file del dump e ripulisce. Sul sistema non installa nulla, a parte un'immagine Docker che si può rimuovere dopo.

## Quando va fatto

Una volta sola per lettore, dopo il primo avvio di ARM (serve la cartella `.MakeMKV` creata da ARM) e prima di inserire il primo Blu-ray. I DVD non sono colpiti. Se cambiate lettore va rifatto, perché il dump è specifico del lettore.

## Uso con ARM

Serve `lsscsi` (`sudo apt install lsscsi`). Poi:

```bash
wget https://raw.githubusercontent.com/fabio983/makemkv-fw-dump/main/makemkv-fw-dump.sh
chmod +x makemkv-fw-dump.sh
sudo ./makemkv-fw-dump.sh --install
```

Lo script ferma ARM, chiede di inserire un Blu-ray, crea il dump, lo copia nella cartella di ARM e fa ripartire ARM.

Non inserite il disco prima che lo script lo chieda, altrimenti ARM potrebbe avviare un job ed espellere il disco quando viene fermato.

Senza `--install` il dump viene salvato solo nella cartella di output, così potete controllarlo prima di copiarlo.

## Uso senza ARM

Se usate `makemkvcon` con i vostri script, oppure ARM non è ancora installato, lanciate lo script senza `--install`:

```bash
sudo ./makemkv-fw-dump.sh
```

Lo script scarica la chiave beta gratuita di MakeMKV. Per usare la vostra chiave si passa `-k`:

```bash
sudo ./makemkv-fw-dump.sh -k "T-XXXXXXXXXXXX"
```

Poi si copia il dump nella cartella `.MakeMKV` dell'utente che lancia `makemkvcon`, accanto a `settings.conf`:

```bash
mkdir -p ~/.MakeMKV
cp makemkv-fw-dump-*/dump_FW_*.tgz ~/.MakeMKV/
```

Se installerete ARM più tardi, avviatelo una volta perché crei la sua cartella `.MakeMKV` e copiate il dump lì.

## Dettagli e opzioni

Tutte le opzioni dello script (`-d`, `-k`, `-o`, `-c`, `--install`, `--remove-image`), la procedura per copiare il dump a mano e le configurazioni testate sono nel README del repository.
