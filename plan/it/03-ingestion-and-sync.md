# 03 · Acquisizione, decodifica FIT e sincronizzazione all'apertura

## 1. Ordine della pipeline (per uscita)

```text
list (summary)  →  detail  →  FIT file  →  decode  →  clean power  →  best power  →  derived (zones, TSS check, climbs)
   →  loads (descent / braking / pedalling work)  →  ride stats (VI, decoupling, torque map)  →  radar metrics
   →  bike assignment  →  commit
```

Ogni fase è una funzione con una modalità "elabora solo ciò che manca", così lo stesso codice serve per la prima importazione, la sincronizzazione incrementale e le riparazioni.

## 2. Prima importazione

1. **Elenco.** Scorri l'elenco delle attività (50 per chiamata) dalla data di inizio scelta fino a *domani*. Tieni solo i tipi ciclistici: `cycling, road_biking, gravel_cycling, mountain_biking, indoor_cycling, virtual_ride, e_bike_fitness, e_bike_mountain, cyclocross, track_cycling, bmx, recumbent_cycling, handcycling, indoor_handcycling`. Ricava il tipo da `activityType.typeKey`. Inserisci le colonne di riepilogo (`distance, duration, movingDuration, elevationGain, averageHR, maxHR, avgPower, maxPower, normPower, trainingStressScore, intensityFactor`).
2. **Dettaglio** per ogni uscita senza `detail_loaded`: salva `functionalThresholdPower` (FTP dell'uscita), il lavoro totale, l'elenco dei sensori (`metadataDTO.sensors`: produttore, tipo di dispositivo, seriale) e se è presente un sensore di cambio elettronico (informativo).
3. **File FIT:** dall'esportazione se presente (vedi `01`), altrimenti da `download-service` (ZIP → l'unico `.fit`). Salva l'originale come `data/fit/<activityId>.fit` e non eliminarlo mai.
4. **Decodifica** con `fitparse`: raccogli i messaggi `record` (ordinati per timestamp; i timestamp sono UTC naive, quindi assegna `tzinfo=UTC` **prima** di qualsiasi chiamata a `.timestamp()` — mescolare ora locale e UTC una volta ha spostato l'abbinamento di 1–2 ore), `lap`, `session`, `device_info`. Scarta i campi FIT il cui nome inizia con `unknown`. Salva il flusso come in `02`.
5. Esegui le derivazioni di `04` e fai il commit **dopo ogni uscita**.

Aspettati operazioni lunghe: usa un registro di avanzamento, tollera `429` (sezione 4) e fai in modo che le riesecuzioni riprendano da dove si erano fermate.

## 3. Sincronizzazione incrementale all'apertura della pagina (il comportamento della "Phase 5")

All'avvio del server (e quando l'utente preme "Aggiorna"), in un **thread in background** protetto da un lock (così due esecuzioni non si sovrappongono mai):

1. Calcola `since = date(latest stored ride) - 2 days`; elenca le attività da `since` a domani (1–2 chiamate). Inserisci solo gli id sconosciuti.
2. Per ogni nuova uscita: dettaglio → FIT → decodifica → derivazioni → assegnazione della bici (`05`).
3. Wellness: prendi l'ultimo giorno salvato meno 2 giorni (i dati in ritardo si assestano) e scarica i riepiloghi giornalieri, i riepiloghi del sonno (a blocchi di ≤ 28 giorni) e l'HRV (a blocchi di ≤ 180 giorni) fino a oggi. Esegui l'upsert.
4. Esponi lo stato su `GET /api/sync` → `{stato: "in corso" | "completato" | "errore", rapporto, errore}` e avvia con `POST /api/sync`. L'interfaccia interroga ogni ~3 s mentre è in esecuzione e mostra una riga di stato più un pulsante manuale "Aggiorna".
5. Produci un **rapporto di ciò che non è disponibile nelle nuove uscite** e mostralo in un riquadro di avviso: uscite senza potenza (escluse dalle metriche di potenza), senza frequenza cardiaca (escluse da EF e zone FC), con una bici che richiede conferma (l'API non espone l'attrezzatura).

**Test (obbligatorio):** elimina un'uscita recente da ogni tabella ed elimina il suo file FIT, esegui la sincronizzazione e verifica che venga scaricata esattamente quell'uscita e che i valori salvati siano identici a prima (controlla anche che i numeri di usura dei componenti diminuiscano quando l'uscita viene rimossa e tornino al valore iniziale dopo).

## 4. Limiti di richieste e robustezza

- Minimo 1 s tra le richieste; `429` → attendi `Retry-After` o 65 s, riprova fino a 6 volte; qualunque altro codice diverso da 200 → solleva un errore con il percorso (mai la chiave).
- Un `401` provoca un solo rinnovo del token e un solo nuovo tentativo.
- Ogni operazione di rete fa il commit per elemento; un'esecuzione interrotta riprende.
- Non tenere mai la connessione al DB aperta durante una lunga attesa di rete senza fare commit.

## 5. Importazione del wellness

**Storico dall'esportazione** (se fornita): file giornalieri e file del sonno, solo dalla data di inizio scelta. Sii esplicito nella scelta dei campi (verifica ciascuno con l'API su 3 giorni recenti prima di fidarti):

- FC a riposo → `currentDayRestingHeartRate` (non `restingHeartRate`);
- stress medio → la voce `TOTAL` di `allDayStress.aggregatorList` (`averageStressLevel`);
- Body Battery massimo/minimo/inizio giornata/inizio sonno → le voci `bodyBattery.bodyBatteryStatList` `HIGHEST`, `LOWEST`, `STARTOFDAY`, `SLEEPSTART`;
- punteggio del sonno → `sleepScores.overallScore` (esportazione) oppure `sleepScores.overall.value` (API) — le due fonti differiscono nella forma; normalizza.

**Giorni recenti dall'API:** riepiloghi giornalieri (`heartRate.restingValue`, `movement.steps.value`, `stress.avgLevel`, `bodyBattery.maxValue/minValue`, `intensityMinutes`), riepiloghi del sonno, HRV. L'HRV è disponibile **solo** tramite l'API.

I valori mancanti restano `NULL` (mai stimati). Riporta quanti giorni non hanno sonno, HRV o FC a riposo nella pagina Qualità dei dati.

## 6. Assenza di misuratore di potenza e di frequenza cardiaca

Gestisci come normali le uscite senza potenza (misuratore mancante, o uscita su un'altra bici) e senza frequenza cardiaca (batteria della fascia scarica): salvale, escludile dalle metriche che richiedono quel canale e mostra **N/D**. Elencale in Qualità dei dati con il conteggio per mese. Non imputare valori.

## 7. Insidie incontrate durante la costruzione dell'app di riferimento

- `endDate` è obbligatorio nell'elenco delle attività; se omesso restituisce in silenzio `[]`.
- `activityType` è un oggetto, non una stringa.
- I timestamp FIT sono in UTC; il confronto in ora locale dava "0 di N abbinati".
- L'abbinamento per orario dei file FIT dell'esportazione può produrre due candidati quando due uscite iniziano a un minuto di distanza: scegli il **più vicino** e verifica con un'asserzione che nessun file sia usato due volte.
- Un modulo chiamato `profile.py` oscura la libreria standard; evita nomi come `profile`, `json`, `types`.
- `PATH` delle operazioni lunghe: eseguile in background e **controlla il log/DB**, non il messaggio di uscita del lanciatore.
