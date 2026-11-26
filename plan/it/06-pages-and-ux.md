# 06 · Pagine, navigazione e regole di design

## 1. Navigazione

Una barra laterale a sinistra con **quattro aree comprimibili** (stato ricordato; l'area della pagina corrente è sempre visibile), routing tramite hash (`#page` o `#page/param`, ad es. `#ride/<id>`; un listener `hashchange` deve aggiornare la vista) e una barra superiore.

| Area | Voce | Usa il filtro del periodo |
|---|---|---|
| **Statistiche generali** | Fisiologia · Volume e zone · Profilo del ciclista · Efficienza cardiovascolare · Power Analytics · Carico di allenamento · Cadenza e coppia · Salite · Evoluzione · Correlazioni · Recupero · Stress e sonno | sì |
| | Coach · Qualità dei dati | no (finestre fisse) |
| **Calendario** | Calendario · Uscite da abbinare · Pause e motivi | no |
| **Singola uscita** | Dettaglio uscita · Esplora le attività | no / sì |
| **Officina** | Parco Bici · Componenti e usura | parco sì / componenti no |

**Barra superiore:** nelle pagine che usano il periodo, la riga di filtro `[ Tutti i tempi | Ultimi 2 anni | Ultimo anno | Ultimi 6 mesi | Ultimi 3 mesi | Ultimo mese ]` più "dal <data>"; nelle altre, una briciola di pane "Area › Pagina". A destra: testo di stato della sincronizzazione + pulsante "Aggiorna" e un **interruttore del tema sole/luna**; l'icona della scheda è una favicon SVG (`favicon.svg`, in `frontend/public`). **Apri su "Ultimi 6 mesi".** "Tutti i tempi" significa tutto ciò che è salvato (nessun limite di età, vedi `01` §5). La riga di filtro delimita **ogni numero, grafico e tabella della pagina**; durante il ricaricamento mantieni il rendering precedente attenuato (nessuno scheletro, nessun salto del layout).

Badge della barra laterale: componenti oltre l'80 %, uscite da abbinare, pause senza motivo. "Dettaglio uscita" nel menu riapre l'ultima uscita vista. Ogni titolo di uscita in ogni tabella e nel calendario rimanda a `#ride/<id>`.

## 2. Specifiche delle pagine

### 2.1 Fisiologia
**Deve stare in una schermata, senza lunghi scroll.** Due righe di riquadri, poi la carta d'identità. Prima riga: un unico riquadro con distanza, dislivello e ore (largo due colonne), NP media, FTP, TSS totale e TSS/settimana (con l'IF medio sotto). Seconda riga, in un contenitore a parte: il gauge del VO₂max e quello dell'HRV notturna, ciascuno largo due colonne, con un pulsante «Andamento» che apre una finestra con il grafico e una sua scelta del periodo (`04` §15). Dai alle colonne una **larghezza massima** e lascia lo spazio libero a destra: i riquadri non devono allargarsi con la finestra. La carta d'identità ha i migliori valori di sempre a 5 s, 1 min, 5 min, 20 min, 60 min in W e W/kg, una barra di confronto per durata con la tabella per età e sesso (`04` §15) e una breve osservazione/ipotesi.

### 2.1b Volume e zone
Tutto ciò che affollava Fisiologia: ore settimanali (con interruttore per i km), TSS settimanale, rapporto ATL/CTL; **picchi nel periodo (barre) contro il miglior valore di sempre (linea con marcatori)**; EF delle uscite > 90 min nel tempo; tempo per zona di potenza e per zona di frequenza cardiaca (mostra a parte le ore a 0 W); tabella delle **ultime 15 uscite** (data, titolo con link, km, dislivello, potenza media, NP, FC media, EF, TSS — quello di Garmin, con il valore proprio tra parentesi solo se distante > 2 % — e bici con una pillola rossa "incerta").

### 2.2 Profilo del ciclista
Due schede: il radar a 7 assi (interruttore di vista *Forma attuale* / *Confronto con i tipi*, una finestra modale "?" con fonti e limiti, un menu dei tipi con predefinito **"più simile a te"**) e una tabella "le sette qualità, una per una" (definizione, metrica, valore, punteggio, dimensione del campione) più la classifica di somiglianza cliccabile. Apri su **Confronto con i tipi**. Vedi `04` §10. Nella tabella di destra la nota «misurato con» di ogni qualità è un tooltip dietro un «?» accanto al nome, per tenere corta la tabella.

### 2.3 Efficienza cardiovascolare
EF per uscita (> 90 min), EF mediano mensile, tempo nelle zone di frequenza cardiaca, grafico del **disaccoppiamento aerobico** (punti per uscita, linee dello zero e della mediana) con una tabella ordinabile; testo di osservazione/ipotesi.

### 2.4 Power Analytics
Curve potenza-durata (due anni scelti dall'utente + il periodo selezionato, asse del tempo logaritmico, interruttore W o W/kg), FTP per trimestre (stima contro Garmin), tabella della potenza critica/W′ (tutta la storia e ultimi 12 mesi), tabella della migliore potenza per anno.

### 2.5 Carico di allenamento
PMC (CTL/ATL/TSB, linea dello zero, bande delle pause), TSS settimanale, CTL a fine settimana; didascalia sul riscaldamento di 6 settimane.

### 2.6 Cadenza e coppia
Mappa di calore coppia-cadenza per il periodo (SVG, il passaggio del mouse legge la cella: intervallo di cadenza, intervallo di coppia, minuti, % del tempo), tabella delle quote per quadrante, barre impilate della quota di quadrante per mese, testo di osservazione/ipotesi/limite.

### 2.7 Salite
Grafico a dispersione (lunghezza contro guadagno, colore = bici, area sensibile ≥ 24 px) e una tabella ordinabile (data, uscita con link, bici, km, guadagno, pendenza, durata, potenza, FC, VAM). Didascalia: stima dell'algoritmo.

### 2.8 Evoluzione
Due intervalli di date (predefiniti secondo `04` §11), una casella "escludi le pause per infortunio/malattia e i 14 giorni successivi", una tabella per metrica (mediane, variazione %, p, p corretto, verdetto in linguaggio semplice) e le due curve di potenza.

### 2.9 Correlazioni
Una mappa di calore a matrice (divergente blu/grigio/rosso), **una guida alla lettura a destra della matrice** (righe = colonne = gli stessi indicatori; significato dei colori; convenzione di forza |ρ| < 0.1 trascurabile, 0.1–0.3 debole, 0.3–0.5 moderata, ≥ 0.5 forte; diagonale; celle vuote; p contro ρ; non è causalità; due esempi calcolati dai dati; i valori della cella sotto il cursore), un interruttore per escludere i giorni di infortunio/malattia e un elenco ordinato comprimibile **senza area di scorrimento interna**.

### 2.10 Recupero
Tre gauge in alto (FC a riposo, massimo del Body Battery, punteggio del sonno) rispetto al solito dell'utente, con una nota che non è una soglia clinica (`04` §15). HRV (notturno e media settimanale), FC a riposo (+ media a 7 giorni), punteggio del sonno (+ media a 7 giorni), Body Battery (massimo/minimo giornaliero), con bande delle pause. I giorni mancanti restano vuoti, mai riempiti.

### 2.10b Stress e sonno
Quattro riquadri (fitness age con età reale e raggiungibile, punteggio del sonno dell'ultima notte con la classe di Garmin, stress dell'ultimo giorno con la qualifica di Garmin, respirazione) e quattro schede: ore al giorno a riposo e con stress basso, medio e alto (barre impilate), respiri al minuto (linea della media con la banda minimo–massimo), fitness age nel tempo e una tabella delle voci del sonno dell'ultima notte con la classe di Garmin e l'intervallo ottimale di ciascuna. Tutto viene dall'API (`01`) e non viene mai ricalcolato; le classi sono di Garmin, la traduzione è tua e deve dirlo.

### 2.11 Coach e Qualità dei dati
Vedi `04` §13–14.

### 2.12 Calendario
Griglia mensile (si parte dal lunedì) con navigazione (precedente/successivo/oggi) e un riepilogo del mese (uscite, km, ore, TSS). Ogni uscita è un'etichetta colorata (colore = bici) che mostra km · ore e TSS (o "senza potenza"), con link al suo dettaglio. I giorni di pausa sono tratteggiati, con "Pausa: <motivo>" sul primo giorno. Una **colonna del totale settimanale** (km, ore, TSS). Legenda dei colori delle bici. Nascondi la colonna della settimana sugli schermi stretti.

### 2.13 Dettaglio uscita
**Cinque schede dentro la pagina** (non schede del browser né indirizzi separati), in una barra sotto l'intestazione fissa così resta sempre in vista: *Panoramica* (riquadri KPI, grafici sincronizzati, la bici dell'uscita), *Percorso e salite*, *Pedalata* (coppia contro cadenza a sinistra, Power phase a destra, su due colonne), *Zone e sforzi* (zone di potenza e di frequenza cardiaca, migliori sforzi, disaccoppiamento aerobico) e *Giri*. Si disegna solo la scheda attiva (la mappa deve essere visibile quando viene creata); la scheda scelta viene ricordata e resta passando all'uscita precedente o successiva; le frecce della tastiera spostano tra le schede. Lo scopo è tenere ogni scheda entro circa una schermata.
**Intestazione fissa** (non scorre; resta appena sotto la barra dei filtri sempre visibile, su sfondo opaco del colore della pagina e con un sottile bordo inferiore): i pulsanti precedente/successiva e il selettore dell'uscita, poi il titolo, poi il sottotitolo (data/ora, tipo, pillola della bici + nota di incertezza). Il pulsante facoltativo «Riassunto IA» (`10`), se l'utente ha scelto quella funzione, sta sulla stessa riga, subito dopo «Successiva». L'intestazione deve già trovarsi nella sua posizione fissa quando la pagina è in cima (nessun margine sopra la barra dei filtri, e il margine inferiore della barra annullato con un margine superiore negativo), altrimenti «salta» di qualche pixel appena inizia lo scorrimento. **Precedente / Successiva non devono ricaricare la pagina:** tieni montata la pagina dell'uscita (non usare l'id dell'uscita nella chiave del componente), continua a mostrare i dati vecchi leggermente attenuati finché arrivano quelli della nuova uscita, non tornare in cima alla pagina e mantieni viva l'istanza della mappa (si ridisegna solo la sovrapposizione). Tutto ciò che sta sotto l'intestazione scorre con il corpo della pagina: dieci riquadri (distanza e dislivello, tempo e tempo in movimento, velocità media, potenza media e NP, IF · TSS e FTP dell'uscita, VI, FC media/max, EF, cadenza, lavoro e temperatura).

- **Grafici sincronizzati** (cursore condiviso tramite `syncId`): altitudine (area), potenza (con una linea di riferimento dell'FTP), frequenza cardiaca, cadenza, velocità — **un asse per grafico** — con un interruttore Tempo/Distanza per l'asse x; ogni punto è la media di N secondi (indica N). Salta il grafico della FC e stampa una nota quando l'uscita non ha FC.
- **Il Training Effect di Garmin** (aerobico e anaerobico) come riquadro KPI, dal messaggio di sessione del FIT (`04` §16).
- **Power phase** (scheda di destra nella sezione *Pedalata*, una colonna alta: i due anelli in alto, il PCO sotto, poi i tempi seduto e in piedi e i valori dei pedali; la spiegazione del PCO è un tooltip «?»; sostituisce la vecchia scheda «dinamica di pedalata»): un anello per gamba, sinistra in verde e destra in azzurro, come nei diagrammi dei pedali di potenza: 0° al punto morto superiore, in senso orario, un arco chiaro per la fase di spinta da inizio a fine e un arco pieno per il suo picco, con la percentuale di equilibrio al centro; sotto, il Platform Center Offset (PCO, in mm) di ogni pedale e i tempi seduto e in piedi. Dati dal messaggio di sessione del FIT (`avg_left_power_phase`, `avg_left_power_phase_peak`, `avg_left_pco` e gli equivalenti di destra). Gli angoli possono essere salvati come testo, quindi interpretali; di' che il segno del PCO segue il diagramma di riferimento. Conserva anche l'efficacia di coppia e la fluidità di pedalata (sinistra e destra) che mostrava la vecchia scheda, e i tempi seduto e in piedi.
- **Percorso**: per impostazione predefinita una **mappa Leaflet su tessere OpenStreetMap** (polilinea blu da ≤ 600 punti GPS, marcatori di partenza/arrivo, zoom, attribuzione «© OpenStreetMap contributors» visibile, zoom con rotellina disattivato così la pagina continua a scorrere). Una barra compatta sopra la mappa contiene l'interruttore «Mappa / Offline» e, sulla stessa riga, il selettore «Colora per»; la scelta viene ricordata; la vista offline è una polilinea SVG (proiezione equirettangolare scalata per cos(latitudine)) con marcatori di partenza/arrivo e barra di scala, ed è usata in automatico se le tessere non si caricano. Un selettore di livelli (icona in alto a destra) offre quattro livelli di base e la scelta viene ricordata: OSM **Standard** (`https://tile.openstreetmap.org/{z}/{x}/{y}.png`), **CyclOSM** (ciclismo, `https://{s}.tile-cyclosm.openstreetmap.fr/cyclosm/{z}/{x}/{y}.png`), **OpenTopoMap** (rilievo, `https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png`, CC-BY-SA) e **Umanitaria** (`https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png`); ogni livello mostra la propria attribuzione. Non aggiungere livelli che richiedono una chiave API o che vietano questo uso (immagini satellitari commerciali, OpenCycleMap/Thunderforest senza la chiave dell'utente). Crea la mappa **una sola volta** e ridisegna solo la sovrapposizione quando cambiano i dati o il colore (i dati dell'uscita vengono ricaricati mentre gira una sincronizzazione: ricreare la mappa azzererebbe lo zoom); usa `detectRetina` per tessere nitide sugli schermi ad alta densità, zoom a mezzi passi e abilita lo zoom con la rotellina solo dopo un clic, così la pagina continua a scorrere. Un pulsante «⤢ Ingrandisci» accanto ai selettori apre una **finestra modale ampia** (circa il 96 % della larghezza e il 92 % dell'altezza) con la stessa mappa, il selettore «Colora per», la legenda e il selettore dei livelli; Esc o un clic fuori la chiude. Una nota sotto la mappa dice che il server delle tessere vede l'area guardata. **Tracciato colorato:** il selettore «Colora per» (tinta unita / velocità / pendenza / quota / potenza / frequenza cardiaca / cadenza; le voci senza dati sono nascoste e la scelta viene ricordata). L'API restituisce valori per punto: medie su ±10 s (quota ±3 s) e la pendenza su ±20 s, impostata a N/D se in quel tratto si sono percorsi meno di 40 m. Scala dei colori: sequenziale viola → giallo tra il quinto e il novantacinquesimo percentile dell'uscita (così i picchi non la appiattiscono); la pendenza usa una scala divergente (blu discesa, grigio piano, rosso salita) simmetrica attorno a 0. Un selettore «Gradiente» (ricordato) offre quattro palette: **Contrasto** (blu → viola → rosso → arancio, quella predefinita), **Caldo** (viola → arancio), **Blu** (da chiaro a scuro) e **Viridis**; le prime tre evitano i colori molto chiari (si perdono su strade e prati chiari) e il verde (si confonde con i boschi della mappa), e ognuna ha una variante divergente per la pendenza con punto centrale grigio scuro. Disegna sotto la linea colorata un bordo scuro semitrasparente, così ogni colore si stacca dallo sfondo, e mostra una legenda a gradiente con minimo e massimo e le unità.
- Grafico coppia-cadenza con le quote di quadrante dell'uscita e la cadenza/coppia mediana.
- Scheda del disaccoppiamento aerobico (o il motivo per cui non è disponibile); scheda della dinamica di pedalata (bilanciamento S/D, efficacia della coppia, fluidità, tempo in piedi).
- Tempo nelle zone di potenza e FC; migliori sforzi dell'uscita (5 s … 60 min) con la loro **posizione tra tutte le uscite** e il miglior valore di sempre; salite rilevate; tabella dei giri (km, tempo, velocità, potenza, NP, max, FC, cadenza, salita, discesa); selettore della bici con "Conferma".

### 2.14 Esplora le attività
Casella di ricerca e colonne ordinabili (le stesse colonne della tabella delle ultime 15), selettore della bici in linea, bordo rosso sulle righe incerte.

### 2.15 Uscite da abbinare, Pause, Parco Bici, Componenti
Vedi `05` e `04` §12. Ogni scheda bici mostra anche le quattro voci di valore di `05` §5 (listino, costo totale, valore con upgrade, rivendita stimata come intervallo), e la pagina del parco bici rilegge le bici a ogni apertura. La pagina Pause elenca le pause automatiche (date, giorni, ultima uscita → prossima uscita, un selettore del motivo, una nota, Salva) più un modulo per aggiungere una pausa manuale più breve, e spiega esattamente che cosa il motivo cambia e che cosa non cambia.

### 2.16 Profilo utente
Una finestra aperta da una **icona del profilo accanto al titolo dell'app** (non una pagina del menu) è l'unico posto per i dati personali di input, che stanno nel database e **non sono mai scritti nel codice** (`02`, tabella `athlete`): sesso, data di nascita (l'età si calcola da qui), peso, frequenza cardiaca massima e la massa di bici più equipaggiamento usata dalla stima del carico frenante. Un pulsante legge da Garmin, in sola lettura, la FC massima, le cinque soglie delle zone e la FC alla soglia del lattato dalle zone cardio; se cambi la FC massima a mano, le zone lette da Garmin vengono scartate. Un secondo pulsante ricalcola i dati derivati che dipendono dal profilo (tempo nelle zone cardio, carico frenante). Convalida ogni campo (peso 30–200 kg, FC massima 120–230, data di nascita non nel futuro) con un messaggio leggibile. Un valore mancante dà N/D dove serve, mai una stima inventata.

## 3. Sistema di design e regole dei grafici

- **Un asse per grafico.** Mai un grafico a doppio asse: due misure → due grafici, piccoli multipli o normalizzati.
- **Il colore ha un compito:** categorico (identità, ordine fisso, mai ciclato; un filtro non deve ricolorare le serie che restano), sequenziale (una tinta da chiaro a scuro, per grandezze ordinate come le zone), divergente (due tinte + punto medio grigio neutro), di stato (riservato; sempre con icona + etichetta). Il testo non porta mai il colore di una serie: usa i token dell'inchiostro; un segno colorato accanto al testo porta l'identità.
- **Segni:** barre ≤ 24 px con l'estremità dei dati arrotondata di 4 px e la linea di base squadrata; linee da 2 px; marcatori ≥ 8 px con un anello da 2 px del colore della superficie; aree a ~10–14 % di opacità; linee di griglia sottilissime e continue; uno spazio da 2 px del colore della superficie tra i segmenti impilati (nessun contorno).
- **Legende:** sempre per ≥ 2 serie; nessuna per una sola. Etichette dirette con parsimonia, mai su ogni punto.
- **Tooltip:** al passaggio del mouse **e** al focus da tastiera; un solo tooltip che elenca ogni serie sotto il cursore, prima il valore (in grassetto), poi il nome della serie, con una chiave a forma di linea; etichette inserite come testo (mai come HTML); aree sensibili ≥ 24 px per i punti.
- **Ogni grafico ha una didascalia di una frase e un interruttore di vista "Tabella"** (gli stessi dati come tabella ordinabile).
- **Disattiva le animazioni d'ingresso dei grafici** (rallentano le linee da 700 punti e rendono inaffidabili screenshot e test).
- **Chiaro/scuro:** definisci tutti i colori come variabili CSS, con l'insieme scuro sia sotto `prefers-color-scheme` sia sotto un attributo `data-theme` esplicito; l'interruttore memorizza la scelta. Le rampe sequenziali per le zone differiscono per modalità (in modalità scura maggiore intensità = più chiaro).
- **Accessibilità:** i pulsanti sono veri `<button>` con `aria-expanded`/`aria-pressed`/`aria-sort`; le finestre modali usano `role="dialog"`, `aria-modal`, portano il focus sulla finestra all'apertura, si chiudono con Esc/clic all'esterno/✕ e ripristinano il focus; le icone hanno alternative testuali; le tabelle hanno vere intestazioni; non affidarti al solo colore.
- **Finestre modali al posto di lunghi blocchi in linea** per le spiegazioni ("?"), i moduli di aggiunta/modifica e i glossari. **Nessuna barra di scorrimento interna** nelle schede quando la pagina scorre già (contrastano con lo scorrimento della pagina); usa lo scorrimento interno solo per tabelle davvero lunghe con intestazioni fisse.
- **Le schede che contengono SVG/grafici devono crescere con il loro contenuto**: un'altezza fissa una volta ha tagliato una matrice 9×9.
- **Gli stati vuoti** dicono sempre perché e cosa fare ("Nessuna uscita da abbinare: tutte le uscite hanno una bici confermata…").

## 4. Regole di ingegneria del frontend (trappole incontrate nella build di riferimento)

- Gli hook di React devono essere eseguiti nello stesso ordine a ogni render: metti tutti gli `useState` prima di qualsiasi `return` anticipato.
- Nel testo JSX, fai l'escape di `<` (scrivi `&lt;`); un `<30°` nudo rompe la build.
- Non definire componenti dentro altri componenti se hanno stato.
- `Recharts`: le aree di riferimento sull'asse x categorico richiedono valori x presenti nei dati (aggancia al giorno più vicino); `syncId` condivide il cursore tra grafici impilati; nascondi le etichette dell'asse x interne tranne che nell'ultimo grafico.
- Verifica ogni modifica con una **scansione dell'intera pagina** (apri ogni pagina per almeno due periodi, verifica l'assenza di errori in console, di `NaN`/`undefined`/`Infinity` nel testo, di "Caricamento" bloccato). Una pagina che mostra silenziosamente "Caricamento..." significa che l'endpoint ha restituito HTML (vedi `07`).
- Quando cambia un'etichetta visibile all'utente, cerca nel sorgente il vecchio testo, le docstring e le note di progetto.
- Salva le piccole preferenze dell'interfaccia in `localStorage` (tema, gruppi di menu chiusi, stato delle fisarmoniche, ordinamento delle tabelle, ultima uscita).
