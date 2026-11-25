# 01 · Fonti dei dati: accesso a Garmin, esportazione e sicurezza

## 1. Garmin Connect Personal API (fonte primaria)

Un livello API personale, di sola lettura, che dà a **una persona l'accesso ai propri dati con una sola chiave** (nessuna domanda da presentare). La disponibilità può cambiare: al momento della stesura è la via consigliata; conferma sulle pagine sviluppatori di Garmin prima di costruire e avvisa l'utente se non è disponibile per lui (in tal caso ripiega sull'esportazione dati, sezione 3).

**Creazione della chiave (la fa l'utente):** Garmin Connect → Impostazioni → *Personal API token* (`https://connect.garmin.com/app/settings/personalApiToken`). Scegli la durata più lunga (fino a un anno; ricorda all'utente quando scade). Garmin mostra la chiave **una sola volta**.

**Conservazione della chiave:** un file di una sola riga, `~/.config/garmin/connect-personal-api-key`, cartella con modalità `700`, file con modalità `600`. La variabile d'ambiente `GARMIN_PERSONAL_API_KEY` o `GARMIN_PERSONAL_API_KEY_FILE` può avere la precedenza. **Non stamparla, non registrarla nei log, non inserirla in un URL, non fare commit, non passarla a nessun servizio che non sia Garmin.** **Su Windows** usa `%USERPROFILE%\.config\garmin\connect-personal-api-key` e limitane l'accesso al tuo utente con `icacls` (vedi il README, passo 2); le regole sono le stesse. Quando ispezioni il file per controllarne il formato, non mostrarne mai il contenuto (verifica solo lunghezza e prefisso, con il valore mascherato).

**Scambio del token:**

```http
POST https://personal.connectapi.garmin.com/personalapitoken-service/personal-api/personal-api-key/token
Authorization: ApiKey <key>
Accept: */*
```

Il corpo della risposta è un JWT (durata di circa 24 h; leggi `exp` dal payload). Tienilo **solo in memoria**, rinnovalo quando restano meno di ~10 minuti e una volta a ogni `401`. Tutte le chiamate dati usano `Authorization: Bearer <jwt>` verso lo stesso host.

**Endpoint usati (tutti `GET`, di sola lettura):**

| Scopo | Percorso | Note |
|---|---|---|
| Elenco attività | `/activitylist-service/activities/search/activities` | parametri `start`, `limit` (≤ 50), `startDate`, `endDate`. **`endDate` è obbligatorio**: senza, l'elenco è vuoto. `activityType` in ogni elemento è un oggetto (`typeKey`), non una stringa. |
| Dettaglio attività | `/activity-service/activity/{id}` | `summaryDTO` (potenza, NP, TSS, IF, FTP di quell'uscita, dislivello, lavoro…), `metadataDTO.sensors` (sensori esterni con numeri di serie), `splitSummaries`. **Nessun campo attrezzatura/bici.** |
| FIT originale | `/download-service/files/activity/{id}` | restituisce uno **ZIP** che contiene `<id>_ACTIVITY.fit` |
| Riepiloghi giornalieri | `/usersummary-service/v2/dailysummaries?startDate&endDate` | passi, FC a riposo, stress, Body Battery, minuti di intensità |
| Riepiloghi del sonno | `/sleep-service/sleep/dailySleepSummariesByDate?startDate&endDate` | ≤ 28 giorni per chiamata |
| HRV | `/hrv-service/hrv/daily/{start}/{end}` | ≤ 180 giorni per chiamata; contiene la fascia di riferimento (`balancedLow/Upper`) e lo stato |
| VO₂max | `/metrics-service/metrics/maxmet/daily/{start}/{end}` | ≤ 170 giorni per chiamata (un intervallo più lungo risponde 400); un valore solo nei giorni in cui cambia; usa il blocco `cycling`, altrimenti quello `generic`, e indica quale era |
| Peso | `/weight-service/weight/range/{start}/{end}?includeAll=true` | grammi; può essere un'impostazione manuale del profilo, non una misura |
| Stress per durata | `/usersummary-service/stats/stress/daily/{start}/{end}` | secondi a riposo, con stress basso, medio e alto per giorno; intervalli di 28 giorni vanno bene |
| Fitness age | `/fitnessage-service/fitnessage/v2/{date}` | età reale, fitness age, età raggiungibile e componenti; una data per chiamata, quindi campiona una volta a settimana |
| Zone | `/biometric-service/heartRateZones/v2` | soglie inferiori delle zone per sport, FC massima, FC alla soglia del lattato |
| Profilo | `/userprofile-service/userprofile/settings` | unità, nome visualizzato (non salvare i campi personali) |

**Limiti noti da comunicare all'utente:**

- Tutti gli **endpoint di attrezzatura/gear restituiscono 404** (non sono nella lista consentita). La bici assegnata a un'uscita **non** può quindi essere letta tramite l'API; vedi `05` per come le bici vengono abbinate in alternativa.
- La allow-list è corta. Oltre agli endpoint sopra risponde con FC giornaliera, idratazione, obiettivi di peso, registro alimentare e allenamenti delle attività. Training readiness, stato di allenamento, endurance e hill score, previsioni di gara e attrezzatura **non** sono esposti: se provi un percorso a caso ottieni quasi sempre 404. Parte di ciò che è esposto è vuoto per molti utenti (SpO₂, peso, composizione corporea).
- Alcune metriche (training readiness, previsioni di gara) non sono esposte. **Il VO₂max è esposto** (vedi la tabella degli endpoint): controlla ogni metrica sull'API prima di dichiararla mancante.
- L'API ha un limite di richieste. Implementa:
  - un intervallo minimo di ~1 s tra le chiamate;
  - su `429`, attendi `Retry-After` (o 65 s se assente) e riprova, fino a 6 volte, stampando l'attesa;
  - rendi le operazioni lunghe **riprendibili** (commit dopo ogni uscita; salta ciò che è già salvato).

## 2. Server MCP facoltativo (accesso conversazionale)

Se l'utente vuole interrogare Garmin anche da Claude Code, si può installare un server MCP di terze parti che incapsula la Personal API. **Verificalo prima di fidarti** consegnandogli la chiave:

- leggi il sorgente (di solito è piccolo) e controlla che ogni chiamata di rete vada solo all'host di Garmin;
- controlla come viene letta la chiave e dove vengono memorizzati i token (modalità `600`, la chiave stessa mai scritta);
- fissa una versione esatta invece di "latest";
- ricorda che riceve la chiave: preferisci un server di sola lettura e ruota la chiave in caso di dubbio.

Il backend della dashboard **non** deve dipendere dal server MCP: il backend parla direttamente con l'API (un server MCP è utilizzabile solo da Claude, non dall'app web).

## 3. Esportazione dati Garmin (storico e attrezzatura)

Garmin Connect → Account → *Esporta i tuoi dati*. Lo ZIP/le cartelle contengono, tra l'altro:

| Cartella / file | Uso |
|---|---|
| `DI_CONNECT/DI-Connect-Uploaded-Files/*.zip` | **ogni file FIT caricato** (attività e file di monitoraggio dell'orologio) |
| `DI_CONNECT/DI-Connect-Aggregator/UDSFile_*.json` | riepiloghi wellness giornalieri (passi, FC a riposo, stress, Body Battery, minuti di intensità) |
| `DI_CONNECT/DI-Connect-Wellness/*_sleepData.json` | un record per notte (fasi, punteggio, respirazione) |
| `DI_CONNECT/DI-Connect-Fitness/*_gear.json` | **l'elenco di bici/scarpe e il legame tra ogni attività e la sua attrezzatura** |
| `DI_CONNECT/DI-Connect-Fitness/*_summarizedActivities.json` | riepiloghi delle attività |
| `DI_CONNECT/DI-Connect-Metrics/*` | carico di allenamento, max-MET (VO₂max) e storico della capacità ciclistica |
| `DI_CONNECT/DI-Connect-User/user_profile.json` | sesso e data di nascita: servono solo a collocare l'utente in una fascia d'età (`04` §15); tienili in una tabella locale, mai nel repository |

Regole per usare l'esportazione:

- È un'**istantanea** (termina nel giorno richiesto): le nuove uscite arrivano comunque dall'API.
- **Non scaricare di nuovo i file FIT presenti nell'esportazione.** Abbinali alle attività per **orario di inizio**: leggi solo il messaggio `file_id` di ogni FIT (il suo `time_created`, **in UTC**), indicizza tutti gli ZIP senza estrarli e accoppia ogni uscita con il file più vicino entro ±10 min (verifica che l'abbinamento sia univoco e che lo scarto massimo sia di pochi secondi). Estrai solo i file abbinati, nel tuo `data/fit/<activityId>.fit` (nomi di file scelti da te, mai presi dall'archivio).
- **Nell'esportazione non c'è alcun file HRV**: ricava l'HRV dall'API.
- Alcuni campi dell'esportazione sono fuorvianti: ad esempio nel file giornaliero `restingHeartRate` è un valore di riferimento costante; la FC a riposo giornaliera è `currentDayRestingHeartRate` (confronta con l'API per alcuni giorni prima di fidarti di un campo).
- `*_gear.json` fornisce `gearDTOS` (elenco con `gearPk`, tipo "Bike"/"Shoes", modello, date) e `gearActivityDTOs` (dizionario gearPk → elenco di `{gearPk, activityId}`). Usalo per assegnare le bici alle uscite **storiche** (le scarpe stanno negli stessi elenchi: filtra per tipo).

## 4. Checklist di sicurezza e privacy

- Il file della chiave ha modalità `600`; il repository non contiene mai né la chiave, né il DB, né i file FIT, né l'esportazione (aggiungi `data/`, `raw_data/`, `.venv/`, `node_modules/` a `.gitignore`).
- Analizza le esportazioni con codice semplice: `python -I` (modalità isolata) per gli script ad hoc, leggi i membri dello ZIP in memoria, mai `eval`, non eseguire mai nulla trovato in un download, non fidarti mai dei nomi di file dentro un archivio.
- Non inviare a terzi alcuna uscita, posizione o dato personale. L'unica eccezione voluta è la mappa del percorso, che carica le **tessere di OpenStreetMap** (`https://tile.openstreetmap.org/{z}/{x}/{y}.png`): il server delle tessere vede l'area guardata e l'IP, mai i dati dell'uscita. Rispetta la politica d'uso delle tessere OSM (attribuzione visibile, nessun download massivo, nessuna cache delle tessere) e offri sempre il disegno offline. Il riassunto IA locale facoltativo (`10`) **non** aggiunge dati in uscita (il modello gira su `127.0.0.1`), solo il download una tantum del programma Ollama e del modello.
- Quando cerchi online specifiche di prodotti o documentazione, invia solo nomi generici di prodotto, mai dati personali.
- Se la chiave viene mai mostrata per errore, dillo all'utente e consiglia di ruotarla.

## 5. Politica di conservazione

Conserva **tutto** dalla prima uscita importata in poi, **senza alcun limite di 24 mesi (o altro)**. Nulla nel codice può eliminare uscite, flussi o righe di wellness in base all'età. I valori di riferimento ("il tuo record personale", "il tuo massimo") sono calcolati sull'intero storico salvato. Il filtro del periodo cambia solo ciò che viene mostrato.
