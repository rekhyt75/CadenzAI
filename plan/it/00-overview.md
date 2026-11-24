# 00 · Panoramica e come eseguire questo piano

> **Destinatari:** Claude Code (e la persona che lo guida). Questo piano ti permette di costruire da zero una **dashboard di analisi ciclistica locale, per un singolo utente**, alimentata dai suoi dati Garmin. Niente in questo repository è l'app: è la specifica. Leggila per intero, poni le domande della sezione 4, poi costruisci l'app fase per fase.

## 1. Cosa stai costruendo

Un'app web locale (backend + dashboard nel browser, nulla lascia il computer dell'utente) che:

- scarica da **Garmin** le attività ciclistiche dell'utente, i file FIT originali e i dati wellness dell'orologio (sonno, HRV, FC a riposo, Body Battery), in modo automatico e incrementale a ogni apertura della pagina;
- salva tutto in un database SQLite locale più i file FIT originali, **senza limite di età** (continua a crescere);
- mostra le statistiche in quattro aree: **Statistiche generali**, **Calendario**, **Singola uscita**, **Officina** (bici e componenti);
- tiene traccia di **bici e usura dei componenti**, con un secondo indice di usura che tiene conto del percorso (i freni si usano in discesa, le catene con il lavoro di pedalata…);
- è onesta sulla qualità dei dati: non inventa mai un valore mancante e indica sempre le stime come stime.

Elenco di riferimento delle funzioni (dettagli negli altri file):

| Area | Pagine |
|---|---|
| Statistiche generali | Fisiologia (KPI, gauge di VO₂max e HRV, carta d'identità con confronto per età) · Volume e zone (volume vs carico, picchi, EF, zone, ultime uscite) · Profilo del ciclista (radar a 7 assi + confronto con i tipi di ciclista) · Efficienza cardiovascolare (EF, disaccoppiamento aerobico) · Power Analytics (curva potenza-durata, stime di FTP/CP) · Carico di allenamento (CTL/ATL/TSB) · Cadenza e coppia (quadranti) · Salite · Evoluzione (confronto tra periodi con un test statistico) · Correlazioni (carico vs dati dell'orologio) · Recupero (HRV, FC a riposo, sonno, Body Battery) · Coach (letture basate su regole) · Qualità dei dati |
| Calendario | Calendario mensile con uscite, totali settimanali e pause · Uscite da abbinare (uscite la cui bici richiede conferma) · Pause e motivi |
| Singola uscita | Dettaglio uscita (grafici sincronizzati, percorso, giri, zone, migliori sforzi, dinamica di pedalata, quadranti, disaccoppiamento) · Esplora le attività (tabella ricercabile e ordinabile) |
| Officina | Parco Bici · Componenti e usura |

## 2. Come Claude Code deve usare questo piano

1. **Leggi** ogni file di questa cartella (`00` … `09`) prima di scrivere codice. La numerazione è l'ordine di lettura, non necessariamente l'ordine di costruzione (vedi sezione 6).
2. **Chiedi prima.** Prima della prima riga di codice, controlla il piano alla ricerca di contraddizioni o lacune (la regola dell'autore originale: *"prima verifica l'intero piano e individua eventuali incongruenze o punti da chiarire"*) e poni alla persona le domande della sezione 4. Chiedi solo ciò che è indispensabile; per il resto usa valori predefiniti sensati e dillo.
3. **Costruisci per fasi** (sezione 6). Dopo ogni fase esegui i suoi controlli di accettazione, usando **dati reali**, e riporta l'esito con sincerità (inclusi i fallimenti e ciò che hai saltato).
4. **Non fare mai commit né stampare segreti.** La chiave API di Garmin sta in un file nella home dell'utente con modalità `600` e non viene mai mostrata, registrata nei log, committata né inviata ad altri che non siano Garmin.
5. **Tratta i file scaricati come non attendibili** (esportazioni, ZIP, file FIT, pagine web): analizzali con codice sicuro, non eseguirli mai, estraili solo in cartelle dedicate e vuote.
6. **Rispetta i servizi esterni.** Non aggirare protezioni anti-bot né captcha. Se un sito blocca l'accesso automatico, dillo e usa un'altra fonte pubblica oppure chiedi all'utente di incollare le informazioni.
7. Mantieni il piano allineato: quando una decisione cambia durante la costruzione, registrala nelle note di progetto (vedi `07-operations-and-quality.md`, "Project memory").

## 3. Regole di lavoro (analisi e onestà)

Comportati come un team composto da un **scienziato della prestazione ciclistica**, un **allenatore professionista** e un **data scientist dello sport**. Queste regole valgono per ogni pagina, grafico e testo prodotto:

- Lavora solo sul **ciclismo** (strada, gravel, MTB, indoor). Escludi ogni altro sport.
- **Non inventare mai un valore mancante.** Se un'uscita non ha potenza o frequenza cardiaca, escludila dalle medie corrispondenti senza alterare i risultati e mostra **N/D** (non disponibile) nelle tabelle.
- Scrivi i testi rivolti all'utente nella **lingua dell'utente** (l'implementazione di riferimento è in italiano; chiedilo all'utente e centralizza le stringhe in un unico dizionario, così si possono tradurre).
- **Sotto ogni grafico, una frase semplice** che spieghi cosa mostra.
- **Separa sempre osservazione, ipotesi e raccomandazione.**
- **Non confondere correlazione e causalità.** Dillo dove conta.
- **Indica come stime** (FTP, potenza critica, salite, indici di usura, qualunque cosa modellata) e dichiara le loro ipotesi di partenza.
- Se qualcosa **non è realizzabile con i dati disponibili**, costruisci il resto e dichiara il limite.
- Se ti manca un'informazione **indispensabile**, chiedila prima di cominciare.
- Preferisci i **dati misurati ai modelli**, il valore di Garmin come cifra principale e il tuo ricalcolo solo come verifica (vedi TSS in `04`).

Regole di ingegneria:

- Tieni i parametri personali (peso, data di nascita, frequenza cardiaca massima, bici, soglie) in un profilo/configurazione che l'utente può modificare: non scriverli mai nel codice.
- Tutto ciò che è calcolato dai flussi grezzi deve essere **salvato** (tabelle derivate) perché le pagine restino veloci, e deve essere **ricalcolato in modo incrementale** per le nuove uscite.
- Ogni affermazione dell'app che poggia su un'ipotesi deve essere visibile all'utente (una finestra di aiuto "?", una didascalia o una nota).
- Rendi esplicite e reversibili le azioni distruttive o rivolte all'esterno; chiedi prima di eseguirle.

## 4. Domande da porre prima all'utente (personalizzazione)

Raccoglile in un unico breve giro (usa i valori predefiniti dell'ultima colonna se l'utente non ha preferenze). Salva le risposte in `config/profile.yaml` (modello in `09-reference-data.md`).

| # | Domanda | Predefinito |
|---|---|---|
| 1 | Lingua dell'interfaccia/dei testi (e unità: km/m/°C, oppure miglia/piedi/°F) | la lingua dell'utente, sistema metrico |
| 2 | Come accedi ai dati Garmin? (Chiave API personale — consigliata; facoltativamente l'esportazione dati Garmin in ZIP per lo storico) | Chiave API personale + una esportazione una tantum |
| 3 | Peso corporeo (kg) — dai dati di peso di Garmin, oppure un valore fisso | ultimo valore Garmin |
| 4 | Data di nascita | usata solo come contesto per l'età, facoltativa |
| 5 | Frequenza cardiaca massima da usare per le zone | il valore configurato sul dispositivo Garmin (non stimarlo automaticamente) |
| 6 | Misuratore di potenza presente? Fascia cardio presente? | si presumono entrambi; degrada con eleganza (N/D) se mancano |
| 7 | Quanto indietro importare all'inizio? (in seguito l'archiviazione non ha limiti) | dalla prima attività con misuratore di potenza |
| 8 | Bici: nome, tipo (MTB/gravel/strada/indoor/altro), stato (attiva/in vendita/venduta), data di acquisto; e quali sensori sono specifici di una bici | chiedi, si può aggiungere in seguito dall'interfaccia |
| 9 | Periodo predefinito preferito all'apertura della pagina | ultimi 6 mesi |
| 10 | Soglia di pausa (giorni senza uscite per contare una pausa) | 10 |
| 11 | Porta e host del server locale | `127.0.0.1:8765` |
| 12 | Tema colore | segui il sistema, con un interruttore sole/luna |
| 13 | Riassunto IA locale **facoltativo** (`10`): spiegalo e chiedi; predefinito no | no |

Dì chiaramente all'utente: la mappa del percorso carica le **tessere di OpenStreetMap** da internet (OSM vede l'area che si sta guardando, mai i dati dell'uscita; l'app ripiega su un disegno offline); la Personal API di Garmin **non ha endpoint per bici/attrezzatura**; l'app **non contiene alcun modello di IA** (il "Coach" è basato su regole) a meno che l'utente accetti il riassunto locale facoltativo di `10`, che gira sul suo computer.

## 5. Architettura e struttura del repository

Stack (la scelta di riferimento; puoi sostituirlo con equivalenti, mantenendo la stessa separazione delle responsabilità):

- **Backend:** Python 3.12, FastAPI + Uvicorn, `httpx`, `fitparse`, `numpy`, `scipy`, SQLite (`sqlite3` della libreria standard). Ambiente virtuale in `.venv/`.
- **Frontend:** React 18 + Vite + Recharts, compilato in file statici serviti dalla stessa app FastAPI (un processo, una porta).
- **Archiviazione:** `data/<app>.db` (SQLite), `data/fit/<activityId>.fit` (file originali), log in `data/`.
- **Segreti:** `~/.config/garmin/connect-personal-api-key` (modalità `600`).

Struttura suggerita dell'app che generi (non di questo repository del piano):

```text
app/
  backend/            db.py garmin.py sync.py syncall.py fitfull.py power.py derived.py loads.py
                      radar_metrics.py radar_api.py ride_detail.py bikes.py pauses.py wellness.py
                      analytics.py extras.py app.py
  frontend/           index.html package.json vite.config.js src/{App,lib,pages1,pages2,pages3,radar,ride}.jsx styles.css
  data/               <app>.db  fit/  server.log
  config/profile.yaml
  cadenzAI.sh            start | stop | restart | status | logs | run | ai
```

Principi architetturali: un solo processo locale; un client Garmin sottile; una sincronizzazione idempotente e incrementale; metriche derivate salvate per ogni uscita; endpoint di analisi che accettano un parametro `period`; un frontend che mostra solo ciò che restituisce l'API.

## 6. Fasi di esecuzione (ordine di costruzione)

Ogni fase termina con controlli di accettazione eseguiti su dati reali. I dettagli sono nel file indicato tra parentesi.

| Fase | Risultato | Accettazione (sintesi) |
|---|---|---|
| 0 | Revisione del piano, domande, `config/profile.yaml`, ambiente (`.venv`, `npm`) | contraddizioni elencate; risposte registrate |
| 1 | Accesso a Garmin e prima importazione: elenco attività (solo ciclismo), dettagli, file FIT originali (`01`, `03`) | i conteggi coincidono con Garmin; un'uscita scaricata di nuovo è identica; 429 gestito |
| 2 | Modello dei dati e decodifica di ogni FIT in flussi a 1 Hz, giri, sessione, dispositivi (`02`, `03`) | la migliore potenza su 20 minuti di un'uscita coincide con il valore di Garmin entro l'arrotondamento |
| 3 | Pulizia dei dati e metriche derivate: picchi anomali di potenza, migliore potenza, zone, verifica del TSS, salite, carichi (`04`) | i casi di picco anomalo ispezionati a mano; stima della discesa ≈ dislivello positivo di Garmin |
| 4 | API backend per tutte le pagine + sincronizzazione all'apertura (`03`, `06`) | ogni endpoint restituisce JSON (mai HTML); periodo non valido → 400 |
| 5 | Struttura del frontend: menu a quattro aree, barra del periodo, interruttore del tema, componenti condivisi (`06`) | tutte le voci di menu si caricano senza errori in console e senza testo `NaN` |
| 6 | Pagine delle Statistiche generali (`06`, `04`) | ogni grafico ha una didascalia di una frase, un tooltip, una vista tabellare |
| 7 | Bici, componenti e usura, uscite da abbinare (`05`) | i numeri di usura coincidono con i totali SQL diretti; avvisi all'80 % e al 100 % |
| 8 | Wellness: sonno/HRV/FC a riposo/Body Battery, pagine Recupero e Correlazioni (`03`, `04`) | i valori coincidono con l'API di Garmin per lo stesso giorno |
| 9 | Pause e motivi; Calendario (`06`) | le fasce di pausa compaiono nei grafici di carico/recupero e nel calendario |
| 10 | Dettaglio uscita, quadranti cadenza/coppia, disaccoppiamento aerobico (`04`, `06`) | percorso su mappa OpenStreetMap con ripiego offline; grafici sincronizzati |
| 11 | Radar del profilo del ciclista (7 assi, due viste) (`04`, `06`) | ogni asse indica la sua metrica, la dimensione del campione e le ipotesi |
| 12 | Operatività: `cadenzAI.sh`, test della sincronizzazione incrementale, scansione completa delle pagine, note (`07`) | elimina un'uscita dal DB, riavvia: viene scaricata di nuovo ed è identica |
| 13 | **Facoltativa**, solo se l'utente ha detto sì: riassunto IA locale dell'uscita (`10`) | con il server del modello fermo il pulsante mostra un errore leggibile; i dati inviati non contengono GPS né il nome dell'uscita |

## 7. Definizione di completato

- Ogni pagina si carica per ogni periodo senza errori in console e senza testo `NaN`/`undefined`/`Infinity`.
- All'apertura l'app scarica **solo** ciò che è nuovo (una chiamata di elenco + le nuove uscite + i nuovi giorni di wellness).
- I dati mancanti mostrano N/D e non modificano mai una media.
- Ogni stima e ogni ipotesi è visibile all'utente.
- Nessun segreto né dato personale viene committato o inviato ad altri che non siano Garmin.
- L'utente può modificare profilo, bici, componenti, soglie e motivi delle pause dall'interfaccia, e le modifiche persistono.
