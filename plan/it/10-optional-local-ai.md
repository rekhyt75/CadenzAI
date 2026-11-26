# 10 · Facoltativo: «riassunto IA» di un'uscita con un modello linguistico locale

> **Questa funzione è FACOLTATIVA e DISATTIVATA per impostazione predefinita.** Non costruirla se l'utente non dice di sì. Chiedilo una sola volta, durante le domande della fase 0 (`00-overview.md`, sezione 4), con la spiegazione qui sotto, e registra la risposta in `config/profile.yaml` (`ai_summary.enabled`). Il resto del plan funziona senza modifiche anche senza di essa.

## 1. Che cos'è e che cosa non è

Un pulsante **su richiesta** (icona del robottino, etichetta «Riassunto IA») nella pagina del dettaglio uscita che chiede a un **modello linguistico eseguito sul computer dell'utente** di scrivere un breve riassunto di quell'uscita: osservazioni, ipotesi e cose da guardare. È una comodità che riformula numeri che l'app ha già calcolato. **Non** è un allenatore, non decide nulla e non deve mai essere l'unico posto in cui compare un numero.

## 2. Chiedi prima all'utente (che cosa dirgli)

Spiega a parole semplici, poi chiedi «Vuoi il riassunto IA facoltativo? (predefinito: no)»:

- Il modello gira **in locale** (Ollama in ascolto su `127.0.0.1`); i dati delle uscite **non lasciano mai il computer**.
- **Download** una tantum da internet: il programma Ollama (circa 1.5 GB) e il modello (circa 2.5 GB per quello consigliato). Per questo non viene inviato alcun dato personale.
- Servono **spazio su disco** (circa 4 GB) ed è **lento su una CPU** (circa 1 minuto per riassunto su una CPU portatile da 16 thread).
- Un modello piccolo **può sbagliare**: per questo il testo è etichettato, i numeri in ingresso si possono ispezionare e i numeri della pagina restano il riferimento.
- L'utente può rimuoverlo in qualsiasi momento (cancella la cartella di Ollama e quella dei modelli).

Se l'utente rifiuta, salta tutto questo file e non mostrare il pulsante.

## 3. Vincoli di privacy (non negoziabili)

1. L'URL del modello deve essere un **indirizzo di loopback** (`127.0.0.1`, `localhost`, `::1`). Il codice si rifiuta di funzionare in caso contrario (così un'impostazione sbagliata non può mandare dati a un servizio remoto).
2. Il modello riceve **solo numeri aggregati calcolati dall'app**: nessun tracciato GPS, nessun nome dell'uscita, nessun identificativo, nessuna data oltre al mese. Mostra all'utente esattamente che cosa è stato inviato («Mostra i dati forniti al modello»).
3. Il riassunto viene generato **solo quando si preme il pulsante**, mai in background, e viene salvato in cache in locale.
4. Ollama ascolta solo su `127.0.0.1`. Non esporlo mai su `0.0.0.0`.
5. Il README e i testi dell'app devono dichiarare con sincerità le eccezioni a «niente lascia il tuo computer»: le chiamate a Garmin, le tessere della mappa e i download una tantum di Ollama e del modello.

## 4. Installazione (spazio utente, senza `sudo`)

- Scarica l'archivio Linux ufficiale di Ollama dalla pagina delle release su GitHub (fissa la versione), in una **cartella nuova e vuota**; elenca il contenuto prima di estrarlo (`tar -t`), estrai in `~/.local/opt/ollama`, cancella l'archivio. Su macOS e Windows usa invece l'installatore ufficiale e mantieni le stesse impostazioni.
- Conserva i modelli in `~/.local/share/cadenzai-models` (`OLLAMA_MODELS`).
- Avvia con `OLLAMA_HOST=127.0.0.1:11434 ollama serve`, poi `ollama pull <model>`.
- Integra con `cadenzAI.sh` (`07`): `start` avvia Ollama staccato **solo se è installato e non è già in esecuzione**, ne salva il PID in `data/ollama.pid`, e `stop` lo ferma **solo se lo ha avviato lo script** (verifica che la riga di comando del processo contenga `ollama serve`).

## 5. Scelta del modello e prestazioni misurate (macchina di riferimento: 16 thread, 31 GB di RAM, nessuna GPU dedicata utilizzabile)

| Modello | Dimensione | Tempo per riassunto (CPU) | Qualità nel test |
|---|---|---|---|
| `qwen2.5:3b` | 1.9 GB | circa 40 s | troppo vago; legge male i numeri, inventa consigli |
| `qwen2.5:7b` | 4.7 GB | circa 95–115 s | nessun errore di fatto dopo le correzioni al prompt qui sotto, ma consigli generici; prima inventava il significato di «TSS» |
| `qwen3:4b` | 2.5 GB | circa 70 s | **inutilizzabile**: è la versione che ragiona sempre, il ragionamento finisce nel testo |
| `qwen3:4b-instruct` (**consigliato e predefinito**) | 2.5 GB | circa 55–90 s | confronti corretti, usa più numeri; una lieve sovra-interpretazione (ha chiamato «elevato» un disaccoppiamento aerobico del 6 %) |
| `qwen3:8b` | 5.2 GB | circa 105–130 s | scritto bene ma il più lento; ha affermato che il dislivello era sopra la media quando nessuna media del dislivello era stata data |

Stesse due uscite e stesso prompt per ogni modello, temperatura 0.2; i tempi includono il caricamento del modello dopo un cambio. Due uscite non sono un test statistico: ripeti la prova sulle uscite dell'utente se vuoi esserne certo.

**Quale Qwen 3 scaricare.** Prendi il tag `-instruct`. Il tag semplice `qwen3:4b` è la versione che ragiona sempre e `think: false` non la spegne. Per le versioni ibride come `qwen3:8b` invia `think: false`, e solo ai modelli Qwen 3 (gli altri rifiutano il parametro). Togli dalla risposta ogni `<think>…</think>` come rete di sicurezza.

**Lezioni sul prompt (hanno contato più del modello).** I modelli piccoli sbagliano il verso di un confronto (uno ha scritto che un carico di 244 era sopra una media di 305): scrivi nel codice il confronto con le settimane precedenti, come riga «Lettura», e di' al modello di usare solo quella riga. Un breve glossario delle sigle gli impedisce di inventarne il significato.

Lezioni: la GPU integrata tramite Vulkan era **da 3 a 4 volte più lenta** della CPU, quindi passa `num_gpu: 0` e `num_thread` ≈ 3/4 dei core per impostazione predefinita (modificabili con le variabili d'ambiente `CADENZAI_LLM_GPU_LAYERS`, `CADENZAI_LLM_THREADS`); l'elaborazione del prompt è la parte lenta, quindi tieni breve l'input (circa 700 token). Lascia che l'utente scelga un altro modello con `CADENZAI_LLM_MODEL`. Rimisura sul computer dell'utente e digli il tempo reale.

## 6. Backend

- Endpoint: `GET /api/ai/status` (il server è attivo, il modello è installato), `GET /api/ride/{id}/summary` (testo in cache oppure `{ "testo": null }`), `POST /api/ride/{id}/summary?force=true|false` (genera o rigenera). Un errore restituisce HTTP 503 con un messaggio leggibile (la UI lo mostra).
- Tabella `ride_ai(activity_id PRIMARY KEY, model, text, facts, created)`: cache del testo **e dei dati esatti che sono stati inviati**.
- Richiesta a `POST /api/chat` del server locale con `stream: false`, `keep_alive: "10m"`, `temperature: 0.2`, `num_predict: 320`, `num_ctx: 2048`, più `think: false` solo per i modelli Qwen 3.

## 7. Che cosa entra nel prompt

**Dati** (etichetta → valore, `N/D` se mancano): tipo di uscita, mese, distanza, dislivello, tempo totale e in movimento, velocità media, potenza media, NP, IF, TSS, FTP di riferimento, VI, FC media (oppure «N/D (fascia non registrata)»), cadenza, lavoro, temperatura, disaccoppiamento aerobico (oppure N/D con il motivo), quota del tempo di pedalata per quadrante coppia/cadenza, tempo per zona di potenza, migliori sforzi con la loro posizione di sempre, numero di salite e la più lunga (distanza **e il suo** dislivello), bilanciamento sinistra/destra, bici, e contesto: numero di uscite, km medi e TSS medio dei 28 giorni precedenti.

**Letture deterministiche**, scritte da regole, così il modello riformula invece di interpretare i numeri grezzi: intensità dall'IF (≥ 0.95 molto alta, ≥ 0.85 alta, ≥ 0.75 medio-alta, ≥ 0.55 moderata, altrimenti leggera); regolarità della potenza dal VI (≥ 1.3 molto irregolare, ≥ 1.1 irregolare, altrimenti regolare); cadenza (< 70 bassa, < 90 nella norma, altrimenti alta, con la nota che la cadenza bassa è comune sulle salite ripide in MTB); carico dal TSS (≥ 300 molto alto, ≥ 200 alto, ≥ 100 medio, altrimenti basso). Un'altra lettura confronta l'uscita con le 4 settimane precedenti (carico e distanza: sotto, in linea con o sopra la loro media, entro il 10 %). Ogni lettura è una riga «Lettura» tra i dati.

**Prompt di sistema** (nella lingua dell'utente; italiano nella build di riferimento), con queste regole:

```text
You are the assistant of a cycling app. Write in Italian, clear and sober, to a cyclist who knows their own data.
Use ONLY the numbers provided: do not invent values, do not estimate those marked N/D (say they are not available).
The "Lettura" lines are already verified interpretations: rephrase them, do not contradict them and do not reinterpret the raw numbers.
An N/D value only means that it is missing: say so in one sentence and draw no conclusion from it (neither about fitness nor anything else).
Do not give generic advice about tools or training that is not tied to the numbers.
Do not confuse acronyms (VI is the regularity of power, not of cadence; IF is intensity, not a percentage).
Acronyms, to be used as given: TSS = Training Stress Score (the load of the ride); IF = Intensity Factor (intensity relative to FTP); NP = normalized power; VI = Variability Index (regularity of power, NP over mean power); EF = Efficiency Factor (NP over mean HR); FTP = functional threshold power; aerobic decoupling = how much heart rate drifts up at the same power in the second half. Do not invent the meaning of other acronyms.
For any comparison with the previous weeks use ONLY the "Lettura" line that states it: do not compare numbers yourself.
Structure: three short parts with bold titles: Osservazioni (2-4 facts from the numbers), Ipotesi (possible explanations, always introduced by "forse", "potrebbe" or "è possibile"; never as certainties), Da guardare (1-2 prudent suggestions; no medical advice).
Maximum 140 words. Do not repeat the list of data.
```

## 8. Interfaccia utente

- Un pulsante con una divertente **icona di robottino con scintilla** e l'etichetta «Riassunto IA», posto sulla riga dell'intestazione dell'uscita **subito dopo «Successiva»** (`06`). Un leggero dondolio al passaggio del mouse solo se l'utente non ha chiesto di ridurre i movimenti.
- Apre una **finestra modale**: una riga di stato durante la generazione («il robottino sta leggendo i tuoi numeri… sul tuo computer può volerci da 20 secondi a un minuto»), poi il testo (con i titoli in grassetto), poi una riga «Scritto da `<model>` in locale il `<date>` in `<n>` s. È un modello piccolo e **può sbagliare**: i numeri veri sono quelli della pagina. Nulla è stato inviato fuori dal computer.»
- Pulsanti: «Rigenera» e «Mostra/Nascondi i dati forniti al modello» (una tabella a due colonne con i dati salvati; i valori lunghi vanno a capo).
- Stato di errore: un messaggio leggibile che dice come avviare il server del modello e scaricare il modello.
- Il componente ha come chiave l'id dell'uscita, così la finestra si chiude quando cambia l'uscita.

## 9. Controlli di accettazione

1. Con il server del modello fermo, il pulsante mostra un errore leggibile e il resto della pagina non ne risente.
2. Impostando l'URL su un host diverso dal loopback il backend si rifiuta di chiamarlo (provalo).
3. La tabella dei dati mostrata all'utente **non** contiene coordinate GPS né il nome dell'uscita.
4. Le percentuali nei dati sono percentuali (le quote dei quadranti sono salvate in **secondi**: dividi per la loro somma).
5. Un'uscita senza frequenza cardiaca produce «N/D» e il riassunto non ne trae alcuna conclusione (leggi tre testi generati).
6. Una seconda pressione restituisce subito il testo in cache; «Rigenera» ne produce uno nuovo.
7. `./cadenzAI.sh stop` non ferma un Ollama che l'utente ha avviato a mano.
8. Su tre uscite reali, ogni confronto con le settimane precedenti nel testo generato coincide con i numeri dei dati.

## 10. Limiti onesti

Un modello da 4 miliardi di parametri su una CPU produce testo plausibile, a volte generico. Consideralo una riformulazione leggibile dei numeri, non un'analisi. Modelli più grandi o un'API scriverebbero meglio ma manderebbero dati fuori dal computer; se l'utente insiste, è un'altra funzione, con il suo consenso esplicito, e non va costruita di nascosto sopra questa.
