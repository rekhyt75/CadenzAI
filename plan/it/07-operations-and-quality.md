# 07 · Operatività, server, test e lezioni apprese

## 1. Server backend

App FastAPI servita da Uvicorn su `127.0.0.1:<port>` (predefinita 8765), con lo stesso processo che serve il frontend compilato (`frontend/dist`: `/assets/*` e un fallback su `index.html` per qualsiasi percorso non API). All'avvio (lifespan): crea/migra le tabelle, poi **avvia la sincronizzazione in background** in un thread protetto da un lock. Resta in ascolto solo su localhost.

**Convenzioni dell'API**

- Tutti gli endpoint dei dati sono `GET /api/...` e accettano `period` ∈ `all | 2y | 1y | 6m | 3m | 1m` (predefinito `6m`); qualsiasi altro valore → **HTTP 400**. Uscita sconosciuta → **404** con un messaggio.
- Endpoint: `meta, sync (GET/POST), overview, power, radar, ride/{id} (0 = latest), quadrant, decoupling, compare, recovery, correlations, climbs, equipment, nobike (rides to match), coach, quality, activities, bikes (CRUD), activities/{id}/bike, components (CRUD), pauses (GET/POST/DELETE)`.
- **Ogni endpoint deve restituire JSON.** Poiché una route catch-all di file statici serve `index.html` per i percorsi sconosciuti, un endpoint cancellato per errore restituisce in silenzio HTML e la pagina resta bloccata su "Caricamento...". Dopo **ogni** modifica al backend esegui uno smoke test che chiami ogni endpoint e interpreti la risposta come JSON.
- Modelli Pydantic per i body; restituisci `{ "ok": true }` per le mutazioni; per le richieste non valide invia il messaggio del server in `detail` (la UI lo mostra).

## 2. Lo script `cadenzAI.sh`

Comandi: `start | stop | restart | status | logs [N] | run | test | ai start|stop|status`. Comportamento da riprodurre:

- `start`: se è già in esecuzione dillo ed esci con 0; altrimenti rifiuta con garbo se **un altro programma** usa la porta (non toccarlo mai); avvia Uvicorn **completamente staccato** e attendi (≤ 20 s) che `GET /api/meta` risponda 200; scrivi il file PID `data/server[-<port>].pid`; stampa l'URL.
- `stop`: trova il PID (file PID, altrimenti chi è in ascolto sulla porta) e verifica con `/proc/<pid>/cmdline` che sia **uvicorn `app:app` avviato proprio su questa porta** prima di inviare `SIGTERM`; passa a `SIGKILL` dopo 10 s; avvisa se è in corso un aggiornamento Garmin ("riprende al prossimo avvio"). Esci con 0 se non è in esecuzione.
- `restart` = stop + start. `status`: PID, uptime, URL, codice HTTP, lo stato della sincronizzazione e il percorso del log; **codice di uscita 0 se è in esecuzione, 3 se non lo è**. `logs`: senza argomento → segui il log; con un numero → ultime N righe (convalidalo). `run`: in primo piano.
- Con l'IA locale facoltativa (`10`), `start` avvia anche Ollama (solo loopback) se è installato e non è in esecuzione, e `stop` lo ferma solo se lo ha avviato lo script. `status` stampa una riga «modello locale» (non installato / spento / attivo con i modelli installati); `ai start|stop|status` gestisce solo il server del modello; `CADENZAI_AI=0` ne impedisce l'avvio.
- Host/porta da variabili d'ambiente (`<APP>_HOST`, `<APP>_PORT`); il file PID è **per porta**.

Lezioni (ognuna è costata un'esecuzione fallita nella build di riferimento):

1. **Non usare mai `pkill -f '<pattern>'`** quando il pattern compare anche nella tua riga di comando: uccide la tua shell. Termina per PID.
2. **Verifica che il PID sia tuo e su questa porta** prima di terminarlo: un file PID di un'altra porta una volta ha fatto uccidere a `stop` l'istanza sbagliata.
3. **Stacca il processo correttamente.** Avvia con le redirezioni applicate alla subshell *prima* del fork e `exec setsid nohup uvicorn …`; altrimenti il figlio eredita lo stdout del chiamante e `./cadenzAI.sh restart | tail` non termina mai (qualsiasi chiamante che attende la fine dell'output si blocca).
4. Gli avvisi di "completato" dei task in background significano solo che il *launcher* ha finito: controlla il log e il DB.

### 2.1 Windows

Su Windows preferisci **WSL 2** (Ubuntu): tutto ciò che precede funziona senza modifiche al suo interno. Per una build Windows nativa, chiedi uno `cadenzAI.ps1` (PowerShell) con gli stessi comandi, codici di uscita, file PID per porta e URL, sostituendo le parti valide solo su Linux:

| Linux | Windows (PowerShell) |
|---|---|
| `setsid nohup uvicorn … &` | `Start-Process -WindowStyle Hidden -RedirectStandardOutput/-RedirectStandardError <log> -PassThru` (nessuna finestra di console, output nel log) |
| chi ascolta sulla porta (`ss`/`lsof`) | `Get-NetTCPConnection -LocalPort <port> -State Listen` → `OwningProcess` |
| `/proc/<pid>/cmdline` | `(Get-CimInstance Win32_Process -Filter "ProcessId=<pid>").CommandLine` deve contenere `app:app` e `--port <port>` |
| `SIGTERM`, poi `SIGKILL` | `Stop-Process -Id <pid>`, poi `-Force` dopo 10 s |
| `tail -f` / `tail -n N` | `Get-Content <log> -Wait` / `-Tail N` |

Lezioni per Windows: i percorsi dell'ambiente virtuale sono `.venv\Scripts\python.exe`; consenti gli script `.ps1` per l'utente corrente con `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`; non terminare mai per nome del processo (`Stop-Process -Name python`), solo per PID verificato; su Windows i file aperti non si possono cancellare, quindi ferma il server prima di toccare `data/`.

## 3. Build ed esecuzione

```bash
python3 -m venv .venv && .venv/bin/pip install fastapi uvicorn httpx fitparse numpy scipy pydantic
cd frontend && npm install && npm run build      # produces frontend/dist
./cadenzAI.sh start                                  # http://127.0.0.1:8765
```

Su Windows (PowerShell): `py -3.12 -m venv .venv; .venv\Scripts\pip install fastapi uvicorn httpx fitparse numpy scipy pydantic`, poi gli stessi comandi `npm` e `.\cadenzAI.ps1 start`.

Ricompila il frontend dopo ogni modifica alla UI (`npm run build`); riavvia il server dopo le modifiche al backend.

## 4. Protocollo di verifica (da eseguire alla fine di ogni fase)

1. **Controlli sui dati** elencati in `04` per la metrica che hai appena costruito (stampa i numeri, confrontali con Garmin dove esiste un riferimento).
2. **Smoke test degli endpoint**: ogni endpoint × un paio di periodi restituisce JSON; periodo non valido → 400; id sconosciuto → 404.
3. **Scansione completa delle pagine in un browser reale** (Chrome DevTools MCP o Playwright): per ogni voce di menu e per almeno due periodi, attendi che "Caricamento" scompaia, poi verifica che non ci siano errori in console né `NaN`, `undefined`, `Infinity` o `[object` nel testo visibile; elenca le pagine con e senza il filtro del periodo.
4. **Controlli di interazione**: apertura/chiusura dei modali (Esc, clic all'esterno, ✕, ritorno del focus), apertura/chiusura degli accordion e persistenza dopo il ricaricamento, ordinamento delle colonne (con N/D in fondo), interruttori (tema, viste, asse x), link dalle tabelle alla pagina dell'uscita, uscita precedente/successiva.
5. **Guardalo.** Fai screenshot della pagina che hai modificato in modalità scura **e** chiara e controlla layout, sovrapposizioni, contenuti tagliati, collisioni tra etichette. I grafici che si animano all'ingresso producono screenshot disegnati a metà: è uno dei motivi per disattivare le animazioni.
6. **Test di idempotenza** della sincronizzazione (`03` §3) e **conteggio di integrità dei dati** (uscite, file FIT, righe per ogni tabella derivata) prima/dopo ogni migrazione.
7. **Riferisci con onestà:** cosa hai verificato, cosa non hai potuto verificare, cosa è fallito.

## 5. Memoria del progetto (note)

Tieni un file breve (per Claude Code: la sua directory di memoria, oppure `NOTES.md`) con le **decisioni dell'utente che il codice non può mostrare**: peso e FC max scelti, regole sulle bici, soglie, nomi, cosa è stato deliberatamente non costruito e le preferenze di lavoro dell'utente (ad es. "non fermarti dopo aver detto che procedi", "aspetta tutte le risposte a un elenco di domande"). Aggiornalo ogni volta che una decisione cambia (nomi dei menu, conservazione, valori predefiniti).

## 6. Lezioni apprese (checklist degli errori da evitare)

| Area | Lezione |
|---|---|
| API Garmin | `endDate` obbligatorio; `activityType` è un oggetto; gli endpoint gear danno 404; gestisci il 429 con attese; mantieni riprendibili i lavori lunghi |
| Tempo | gli orari FIT sono UTC; collega `tzinfo` prima di `.timestamp()`; abbina i file dell'export per orario di inizio più vicino |
| FIT | `left_right_balance` ha un bit di flag; scarta i campi `unknown*`; conserva i file originali |
| Campi dei dati | `restingHeartRate` dell'export è una costante, il valore giornaliero è `currentDayRestingHeartRate`; verifica ogni campo dell'export con l'API |
| Pulizia | il primo filtro dei picchi anomali era troppo aggressivo (rimuoveva partenze da fermo reali), il secondo troppo debole (non vedeva le spinte a bassa velocità): convalida sui gruppi rimossi più grandi e sulla curva del miglior 1 s/5 s |
| Metriche | dividi il disaccoppiamento sui **campioni in movimento**, non sul tempo di calendario; riscaldamento della CTL sui primi 42 giorni; ore in movimento (non trascorse) per la sospensione |
| Soglie | non pubblicare una soglia di usura senza una fonte: le 50 h non motivate hanno causato un falso allarme |
| DB | le tabelle create da una versione precedente richiedono migrazioni `ALTER … ADD COLUMN`; non cancellare mai per età |
| Backend | un endpoint cancellato restituisce l'HTML della SPA → il frontend resta su "Caricamento..." per sempre: fai lo smoke test di tutti gli endpoint; non dare a un modulo il nome di un modulo della libreria standard |
| Frontend | l'effetto del focus di un modale non deve dipendere da un `onClose` inline (ruba il focus a ogni tasto: tienilo in un riferimento); gli elenchi in cache in un contesto vanno ricaricati quando si apre una pagina che dipende da altre modifiche; gli hook prima dei return anticipati; fai l'escape di `<` nel testo JSX; niente card ad altezza fissa attorno a contenuti di dimensione ignota; niente barre di scorrimento interne; grafici sincronizzati tramite `syncId` |
| Operatività | niente `pkill -f`; verifica proprietà e porta del PID; stacca completamente il processo; leggi i log, non i codici di uscita del launcher |
| Database | un percorso di lettura non deve mai scrivere (fallisce con «database is locked» mentre la sincronizzazione scrive); conferma le importazioni lunghe a piccoli passi e non tenere mai una scrittura aperta durante chiamate di rete lente |
| Dati personali | nessun valore personale nel codice: peso, frequenza cardiaca massima, sesso, data di nascita e massa della bici stanno nella tabella del profilo e si modificano da una pagina; un valore mancante dà N/D |
| Layout | metti ogni riga di riquadri in un suo contenitore e limita la larghezza delle colonne: una sola griglia che si adatta sposta i riquadri tra le righe quando la finestra cresce, e allargarli per riempire la larghezza è stato rifiutato; una scheda con un gauge occupa due colonne |
| Sicurezza locale | il server è locale ma il browser no: convalida l'intestazione `Host` (DNS rebinding) e l'`Origin` di ogni scrittura (CSRF: un POST senza corpo non richiede il permesso CORS); aggiungi una content-security policy e `X-Frame-Options`; tieni riservati database e cartella dei dati (`600` e `700`); rifiuta un indirizzo di ascolto non locale salvo forzatura |
| Affidabilità | SQLite in modalità WAL con attesa di 30 secondi e schema applicato una volta per processo; una prova di fumo che chiama ogni endpoint GET e si aspetta JSON, i casi 400 e 404 e i rifiuti qui sopra; un error boundary attorno a ogni pagina, così una pagina rotta non svuota l'app |
| Refactoring | prima di toccare una funzione complessa, salva il JSON degli endpoint principali per varie uscite reali (chiavi ordinate) e controlla che due snapshot di fila siano identici; dopo ogni refactor lo snapshot deve essere identico byte per byte. Misura prima la complessità (con uno strumento, non a occhio: una lista troncata una volta ha nascosto la vera seconda funzione peggiore) e tratta i rami morti che trovi come segnalazioni, non come correzioni silenziose: un dato del prompt dell'IA non era mai comparso perché leggeva una chiave inesistente |
| Collaborazione | l'utente potrebbe usare l'app mentre testi: usa una finestra del browser separata, non confermare mai una finestra di dialogo distruttiva che non hai aperto e ripristina ogni stato che hai cambiato durante i test (ad es. elimina i componenti di prova) |
| Onestà | di' quando un valore è un'approssimazione, quando un controllo non è stato eseguibile e quando la tua prima risposta era sbagliata |
