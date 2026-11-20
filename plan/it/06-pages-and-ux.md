# 06 · Pagine, navigazione e regole di design

## 1. Navigazione

Una barra laterale a sinistra con **quattro aree comprimibili** (stato ricordato; l'area della pagina corrente è sempre visibile), routing tramite hash (`#page` o `#page/param`, ad es. `#ride/<id>`; un listener `hashchange` deve aggiornare la vista) e una barra superiore.

| Area | Voce | Usa il filtro del periodo |
|---|---|---|
| **Statistiche generali** | Fisiologia · Profilo del ciclista · Efficienza cardiovascolare · Power Analytics · Carico di allenamento · Cadenza e coppia · Salite · Evoluzione · Correlazioni · Recupero | sì |
| | Coach · Qualità dei dati | no (finestre fisse) |
| **Calendario** | Calendario · Uscite da abbinare · Pause e motivi | no |
| **Singola uscita** | Dettaglio uscita · Esplora le attività | no / sì |
| **Officina** | Parco Bici · Componenti e usura | parco sì / componenti no |

**Barra superiore:** nelle pagine che usano il periodo, la riga di filtro `[ Tutti i tempi | Ultimi 2 anni | Ultimo anno | Ultimi 6 mesi | Ultimi 3 mesi | Ultimo mese ]` più "dal <data>"; nelle altre, una briciola di pane "Area › Pagina". A destra: testo di stato della sincronizzazione + pulsante "Aggiorna" e un **interruttore del tema sole/luna**. **Apri su "Ultimi 6 mesi".** "Tutti i tempi" significa tutto ciò che è salvato (nessun limite di età, vedi `01` §5). La riga di filtro delimita **ogni numero, grafico e tabella della pagina**; durante il ricaricamento mantieni il rendering precedente attenuato (nessuno scheletro, nessun salto del layout).

Badge della barra laterale: componenti oltre l'80 %, uscite da abbinare, pause senza motivo. "Dettaglio uscita" nel menu riapre l'ultima uscita vista. Ogni titolo di uscita in ogni tabella e nel calendario rimanda a `#ride/<id>`.

## 2. Specifiche delle pagine

### 2.1 Fisiologia
Riquadri KPI (km, dislivello, ore, NP media, TSS totale, TSS/settimana, IF medio); una "carta d'identità" con i migliori valori di sempre a 5 s, 1 min, 5 min, 20 min, 60 min in W e W/kg e una breve osservazione/ipotesi; ore settimanali (con interruttore per i km), TSS settimanale, rapporto ATL/CTL; **picchi nel periodo (barre) contro il miglior valore di sempre (linea con marcatori)**; EF delle uscite > 90 min nel tempo; tempo per zona di potenza e per zona di frequenza cardiaca (mostra a parte le ore a 0 W); tabella delle **ultime 15 uscite** (data, titolo con link, km, dislivello, potenza media, NP, FC media, EF, TSS — quello di Garmin, con il valore proprio tra parentesi solo se distante > 2 % — e bici con una pillola rossa "incerta").

### 2.2 Profilo del ciclista
Due schede: il radar a 7 assi (interruttore di vista *Forma attuale* / *Confronto con i tipi*, una finestra modale "?" con fonti e limiti, un menu dei tipi con predefinito **"più simile a te"**) e una tabella "le sette qualità, una per una" (definizione, metrica, valore, punteggio, dimensione del campione) più la classifica di somiglianza cliccabile. Apri su **Confronto con i tipi**. Vedi `04` §10.

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
HRV (notturno e media settimanale), FC a riposo (+ media a 7 giorni), punteggio del sonno (+ media a 7 giorni), Body Battery (massimo/minimo giornaliero), con bande delle pause. I giorni mancanti restano vuoti, mai riempiti.

### 2.11 Coach e Qualità dei dati
Vedi `04` §13–14.

### 2.12 Calendario
Griglia mensile (si parte dal lunedì) con navigazione (precedente/successivo/oggi) e un riepilogo del mese (uscite, km, ore, TSS). Ogni uscita è un'etichetta colorata (colore = bici) che mostra km · ore e TSS (o "senza potenza"), con link al suo dettaglio. I giorni di pausa sono tratteggiati, con "Pausa: <motivo>" sul primo giorno. Una **colonna del totale settimanale** (km, ore, TSS). Legenda dei colori delle bici. Nascondi la colonna della settimana sugli schermi stretti.

### 2.13 Dettaglio uscita
Intestazione con pulsanti precedente/successiva e un selettore dell'uscita; titolo, data/ora, tipo, pillola della bici (+ nota di incertezza); dieci riquadri (distanza e dislivello, tempo e tempo in movimento, velocità media, potenza media e NP, IF · TSS e FTP dell'uscita, VI, FC media/max, EF, cadenza, lavoro e temperatura).

- **Grafici sincronizzati** (cursore condiviso tramite `syncId`): altitudine (area), potenza (con una linea di riferimento dell'FTP), frequenza cardiaca, cadenza, velocità — **un asse per grafico** — con un interruttore Tempo/Distanza per l'asse x; ogni punto è la media di N secondi (indica N). Salta il grafico della FC e stampa una nota quando l'uscita non ha FC.
- **Percorso**: una polilinea SVG da ≤ 600 punti GPS (proiezione equirettangolare scalata per cos(latitudine)), marcatori di partenza/arrivo, una barra di scala, **nessuna tessera di mappa esterna**.
- Grafico coppia-cadenza con le quote di quadrante dell'uscita e la cadenza/coppia mediana.
- Scheda del disaccoppiamento aerobico (o il motivo per cui non è disponibile); scheda della dinamica di pedalata (bilanciamento S/D, efficacia della coppia, fluidità, tempo in piedi).
- Tempo nelle zone di potenza e FC; migliori sforzi dell'uscita (5 s … 60 min) con la loro **posizione tra tutte le uscite** e il miglior valore di sempre; salite rilevate; tabella dei giri (km, tempo, velocità, potenza, NP, max, FC, cadenza, salita, discesa); selettore della bici con "Conferma".

### 2.14 Esplora le attività
Casella di ricerca e colonne ordinabili (le stesse colonne della tabella delle ultime 15), selettore della bici in linea, bordo rosso sulle righe incerte.

### 2.15 Uscite da abbinare, Pause, Parco Bici, Componenti
Vedi `05` e `04` §12. La pagina Pause elenca le pause automatiche (date, giorni, ultima uscita → prossima uscita, un selettore del motivo, una nota, Salva) più un modulo per aggiungere una pausa manuale più breve, e spiega esattamente che cosa il motivo cambia e che cosa non cambia.

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
