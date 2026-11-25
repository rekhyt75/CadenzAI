# 04 · Metriche, formule e validazione

Ogni formula qui sotto indica i propri **parametri**, se è una **misura o una stima** e un **controllo** che devi eseguire sui dati reali. Non rilasciare una metrica di cui non hai potuto eseguire il controllo: segnalala come non verificata.

## 1. Preparazione dei flussi

Costruisci una **griglia regolare a 1 Hz** sull'uscita (`n = last_t + 1` campioni): riempi ogni canale al suo timestamp e lascia `NaN` il resto. Una pausa è un dato mancante, non uno zero.

### 1.1 Rimozione dei picchi anomali di potenza (igiene della misura)

I misuratori di potenza a volte riportano un valore mantenuto per 1–2 s mentre la bici è ferma o in rotazione libera (ad es. 2 000–4 000 W con cadenza 0). Tratta come mancante un **gruppo** di campioni quando valgono **tutte** queste condizioni:

- campioni consecutivi con `power > 0` e `cadence == 0`;
- lunghezza del gruppo ≤ 5 s;
- il secondo prima del gruppo e il secondo dopo non hanno potenza (zero o mancante);
- il massimo del gruppo è ≥ **400 W** (i gruppi più bassi non possono alterare un picco, quindi lasciali).

Impostali a `NaN` (mai a 0). Registra in `power_quality` il conteggio e il massimo per uscita.

**Non** richiedere velocità = 0: lo stesso artefatto compare quando la bici viene spinta lentamente. **Non** scartare le vere partenze in piedi: hanno una potenza continua, non un gruppo isolato. **Controllo:** il miglior 1 s e il miglior 5 s su tutte le uscite devono formare una curva regolare (ad es. 1 s ≈ 900 W, 5 s ≈ 850, 15 s ≈ 790); un miglior 1 s sopra ~2 000 W significa che un picco anomalo è sopravvissuto. Ispeziona a mano i gruppi rimossi più grandi (stesso valore ripetuto per 2 s, zeri intorno).

### 1.2 Migliore potenza per durata

Per le durate `[1, 5, 10, 15, 30, 60, 120, 300, 600, 1200, 1800, 3600, 5400]` s calcola la media mobile massima sulla griglia, accettando una finestra solo se **≥ 95 %** dei suoi secondi ha potenza. Salva per uscita in `best_power`. **Controllo:** i migliori 20 min di un'uscita coincidono con la "potenza massima 20 min" di Garmin (`summaryDTO.maxPowerTwentyMinutes`) a meno di arrotondamenti.

### 1.3 Potenza normalizzata (NP)

Media mobile a 30 s (zeri per i mancanti), quarta potenza, media, radice quarta. Usa `normPower` di Garmin come valore salvato e la tua `NP()` solo per le suddivisioni interne all'uscita (metà, giri).

## 2. Carico e intensità

- **TSS (valore principale): `trainingStressScore` di Garmin.** Formula standard per il controllo: `tss_own = duration_s · (NP/FTP)² / 3600 · 100`, con l'FTP dell'uscita stessa. Mostra il tuo valore **solo se differisce di oltre il 2 %** da quello di Garmin (in tal caso mostra entrambi, prima quello di Garmin). Nella build di riferimento ~77 % delle uscite concordava entro il 2 % e il resto differiva fino a ~19 % per un motivo che non è stato possibile identificare: dillo, non scegliere un vincitore.
- **IF** = NP/FTP (`intensityFactor` di Garmin concordava nel 100 % delle uscite).
- **Variability Index** = NP / potenza media.
- **PMC** (solo uscite con potenza misurata, TSS di Garmin): `CTL += (TSS − CTL)/42`, `ATL += (TSS − ATL)/7`, `TSB = CTL(yesterday) − ATL(yesterday)`; un valore per giorno di calendario a partire da **0 alla prima uscita**. I primi 42 giorni sottostimano la CTL: mostra il rapporto ATL/CTL solo dopo il giorno 42 e scrivi l'avvertenza nella pagina. Il TSS giornaliero è la somma delle uscite del giorno.
- **Volume settimanale:** settimane ISO (inizio il lunedì): ore, km, dislivello, TSS, numero di uscite.
- **KPI per un periodo:** km, dislivello, ore, NP media, TSS totale e TSS a settimana (totale / settimane del periodo), IF medio, numero di uscite con potenza e con FC.

## 3. Zone

- **Zone di potenza (Coggan, % dell'FTP dell'uscita):** Z1 < 55 %, Z2 ≤ 75 %, Z3 ≤ 90 %, Z4 ≤ 105 %, Z5 ≤ 120 %, Z6 ≤ 150 %, Z7 > 150 %. **0 W è una categoria a parte** ("non pedala", importante in MTB). Conta i secondi validi.
- **Zone di frequenza cardiaca:** le soglie inferiori delle zone dell'utente prese dal dispositivo (profilo ciclismo) con la sua FC max; mantieni una categoria "sotto Z1". **Non ricalcolare la FC max**: se i picchi osservati superano il massimo configurato (artefatti o massimo impostato male), elenca quelle uscite in Qualità dei dati e lascia stare le zone.
- Aggrega le zone su un periodo come **tempo** e indica quante uscite hanno contribuito.

## 4. Efficienza

- **EF** = NP / FC media, uscite **> 90 min** con entrambi i canali. Mostra per uscita e le mediane mensili.
- **Disaccoppiamento aerobico (Pw:Hr), per uscita e andamento:** prendi **solo i campioni in movimento** (velocità > 1 m/s), dividili in due metà per numero, calcola `EF = NP(half) / mean HR(half)` per ciascuna, `decoupling % = (EF1 − EF2)/EF1 · 100`. Richiedi ≥ 3 600 campioni in movimento e FC presente in ≥ 80 % di essi. Positivo = la frequenza cardiaca è salita a parità di potenza erogata. **Non dividere per tempo orologio**: le soste lunghe producono valori assurdi (un caso a −97 % è scomparso passando ai campioni in movimento; intervallo di riferimento dopo: −13 … +23 %, mediana ≈ 10 %). Didascalia: indicativo; il terreno lo rende rumoroso; esiste una regola empirica "< 5 % è buono" per sforzi costanti, ma non è validata per la MTB.

## 5. Power Analytics

- **Curva potenza-durata** per anno solare, per il periodo selezionato e per due anni qualsiasi scelti dall'utente; asse delle durate in scala logaritmica; W/kg opzionale.
- **Stima dell'FTP** per trimestre = **95 % dei migliori 20 min** di quel trimestre (una stima, **non un test**); mostrala accanto all'FTP salvato da Garmin e dichiara che nessuno dei due è un test.
- **Potenza critica (CP) e W′**: regressione lineare lavoro-tempo sui migliori sforzi da **5, 10, 20 min** (`work = CP·t + W′`). Una stima: gli sforzi nelle uscite non sono test massimali, quindi CP e W′ sono probabilmente **sottostimati**. Mostrali per l'intera storia e per gli ultimi 12 mesi.
- Peso per i W/kg: dal profilo (predefinito: ultimo valore Garmin, indicando se è un inserimento manuale del profilo).

## 6. Salite (stima)

Leviga l'altitudine con una media mobile a 15 s. Percorri la serie; una salita inizia da un minimo locale e prosegue finché resta entro **12 m dal suo picco corrente**; tienila se **guadagno ≥ 40 m, lunghezza ≥ 400 m, pendenza media ≥ 3 %**. Salva inizio, durata, distanza, guadagno, pendenza, potenza media, FC media, VAM (`gain/duration·3600`), coordinate di inizio. **Controllo:** per uscita la somma dei guadagni delle salite non deve superare il dislivello totale dell'uscita (riferimento: max 95 %, media 65 %). È una stima con soglie fisse: dillo.

## 7. Carichi di usura (stime) — frenata, lavoro in pedalata, discesa

Bilancio energetico fisico al secondo, su una griglia a 1 Hz con altitudine levigata su 15 s e velocità su 3 s:

```text
P_brake = P_gravity + P_pedal − dKE/dt − P_rolling − P_aero      (clipped to 0 … 2000 W)
P_gravity = −m·g·d(altitude)/dt        P_pedal = cleaned power (0 if missing)
dKE/dt    = m·v·dv/dt                  P_rolling = Crr·m·g·v      P_aero = 0.5·ρ·CdA·v³
counted only when speed > 1.5 m/s and data are valid
```

Parametri (dichiarali come ipotesi nell'interfaccia): **m = ciclista + bici + equipaggiamento** (riferimento 95 kg), **Crr = 0.012**, **CdA = 0.5 m²**, **ρ = 1.2 kg/m³**, g = 9.81. Per uscita salva: `brake_kj = ΣP_brake/1000`, `descent_m` (somma dei passi di altitudine negativi della serie levigata), `descent_h` (ore con pendenza < −3 % e velocità > 1 m/s), `pedal_kj = Σpower/1000`, ore in movimento.

**Controlli (devono passare tutti e tre):** (1) discesa stimata / dislivello positivo di Garmin ≈ 1 sugli anelli chiusi (mediana di riferimento 0.99); (2) `brake_kj` / energia potenziale persa `m·g·descent` è tra 0 e 1 (mediana di riferimento 0.6); (3) le uscite gravel in piano mostrano un carico frenante minuscolo per km e le discese ripide uno enorme (rapporto di riferimento ~80×). Correlazione di rango tra `brake_kj` e `descent_m` > 0.9.

## 8. Coppia e cadenza (quadranti)

`torque (Nm) = power / (cadence · 2π/60)` per i campioni con potenza > 0, cadenza ≥ 20 rpm e coppia ≤ 100 Nm. Istogramma in secondi: classi di cadenza da 5 rpm (0–140), classi di coppia da 2.5 Nm (0–100); salva in forma sparsa. Rappresenta come mappa di calore con curve a potenza costante (100/200/300/400 W). **Linee di riferimento (una convenzione, dichiarala nell'interfaccia):** cadenza **80 rpm** e la coppia che dà l'**FTP dell'utente a 80 rpm** (`FTP·60/(2π·80)`). Quadranti: coppia alta/cadenza alta ("sforzo duro"), coppia alta/cadenza bassa ("forza"), bassa/bassa ("facile e lento"), coppia bassa/cadenza alta ("agile"). Riporta la quota di tempo per quadrante, per uscita, per periodo e per mese. Interpreta con cautela: in MTB è il terreno a dettare la cadenza.

## 9. Dinamica di pedalata

Dal messaggio di sessione quando si usa un misuratore a doppio lato: bilanciamento sinistra/destra (decodificalo come in `02`), efficacia della coppia e fluidità di pedalata per lato, tempo in piedi. Presenta come medie per uscita e come andamento; non sovrainterpretare piccole asimmetrie (uno scarto di ~1–2 punti è normale) e confronta i periodi prima/dopo un infortunio solo come osservazione.

## 10. Radar del profilo del ciclista (7 assi) — tutti indicatori indiretti

| Asse | Metrica (indicatore indiretto) | Punteggio per la vista "tipi" |
|---|---|---|
| VO₂max / capacità aerobica | miglior potenza su 5 min, W/kg | equivalente in percentile nella tabella di riferimento (`09`) |
| Soglia (FTP) | miglior potenza su 20 min, W/kg | stesso, colonna 20 min |
| Sprint / anaerobico | media del miglior 5 s e del miglior 1 min, W/kg | media dei due equivalenti in percentile |
| Resistenza | migliori 20 min **dopo ≥ 1 500 kJ di lavoro** nella stessa uscita, come % dei migliori 20 min da fresco | **soglia propria:** ritenzione 70 % → 0, 100 % → 100 |
| Salita | miglior potenza media su una salita rilevata ≥ 10 min, W/kg | equivalente in percentile, colonna 20 min (approssimazione) |
| Passo in piano / costante | migliori 20 min su un tratto quasi pianeggiante (variazione netta di altitudine ≤ 1.5 % della distanza, ≥ 3 km, pochi saliscendi), W/kg | stesso |
| Recupero | giorni per riportare l'HRV notturno al limite inferiore della fascia di riferimento dell'utente dopo un giorno con TSS ≥ 250 (entro 7 giorni, ≥ 1 rilevazione HRV utilizzabile) | **soglia propria:** 1 giorno → 100, 5 giorni → 0 |

Equivalente in percentile: interpolazione lineare tra i livelli pubblicati (30/55/75/90/98); **sotto il trentesimo percentile non ci sono dati, quindi traccia una retta da 0** e dì che è un'ipotesi. Mostra sempre le dimensioni dei campioni (uscite, salite, giorni duri) e avvisa quando sono piccole.

Due viste dietro un interruttore:

1. **Forma attuale:** ogni asse come % del **miglior valore personale dell'utente sull'intera storia** (100 %) per il periodo selezionato (per il recupero 100 % = la media storica dell'utente; più giorni = valore più basso). Anello al 100 %. Con il periodo "Tutti i tempi" tutto è al 100 %: mostra una nota.
2. **Confronto con i tipi:** i punteggi 0–100 dell'intera storia, più una sovrapposizione opzionale della **forma** di un tipo di ciclista portata al livello medio dell'utente. Le forme dei tipi sono **ipotesi qualitative** dell'autore, **non misure** (pesi per asse: velocista, puncheur, MTB / XC, scalatore, cronoman / passista, fondista / granfondo, completo) — dichiaralo in evidenza e lascia che l'utente le modifichi. Il **tipo più vicino** (distanza euclidea tra forme normalizzate sulla media) viene ricalcolato a ogni caricamento della pagina sull'**intera storia**, indipendentemente dal filtro del periodo; preselezionalo, mostra una classifica di somiglianza cliccabile (100 = il più vicino dell'insieme, 0 = il più lontano) e spiega che non è fisso.

## 11. Statistiche per confronti e correlazioni

- **Evoluzione (confronto tra due periodi):** metriche per uscita (NP, potenza media, IF, EF, TSS, km, dislivello, ore); mediana di ciascun periodo, variazione %, **Mann-Whitney U a due code** con almeno 5 uscite per periodo e correzione di **Bonferroni** per le 8 metriche. Dì che le uscite non sono indipendenti. Predefinito: periodo selezionato contro il periodo precedente di pari lunghezza; per "Tutti i tempi", prima metà contro seconda metà della storia. Esclusione opzionale delle pause per infortunio/malattia + i 14 giorni successivi (vedi sotto).
- **Correlazioni (Spearman):** serie giornaliere di TSS del giorno precedente, TSS dei 7 giorni precedenti, CTL, TSB, FC a riposo, HRV notturno, punteggio del sonno, massimo giornaliero di Body Battery, stress. Salta le coppie con < 30 giorni in comune. **Escludi le coppie tra le metriche di carico stesse** (derivano dalla stessa serie). Opzionalmente escludi i giorni all'interno delle pause per infortunio/malattia. Mostra sempre ρ, p e n; avvisa che non è causalità.

## 12. Pause

Una **pausa** = almeno *N* giorni consecutivi (predefinito 10) tra due uscite (anche una in corso dopo l'ultima uscita). Le pause sono calcolate dalle uscite; si salvano solo il **motivo** (infortunio, malattia, viaggio, lavoro, riposo programmato, meteo/manutenzione, altro) e una nota. Il motivo **non cambia mai alcun valore calcolato** (TSS, CTL…): dopo uno stop la CTL deve scendere comunque. I motivi *infortunio* e *malattia* possono opzionalmente escludere la pausa più i **14 giorni** successivi dai confronti tra periodi e i giorni di pausa dalle correlazioni. Disegna le pause come bande grigie nei grafici di carico e recupero e come celle tratteggiate nel calendario.

## 13. Coach (basato su regole)

Una manciata di schede, ciascuna con **Osservazione / Ipotesi / Raccomandazione** scritte a partire da numeri calcolati: carico e freschezza (CTL 6 settimane fa contro ora, TSB), distribuzione dell'intensità (quota in Z1–Z2 / Z3 / Z4+), andamento dell'EF con il suo valore p, pause e ritorni (con i loro motivi), recupero (HRV e FC a riposo a 7 giorni contro 60 giorni), qualità dei dati delle uscite recenti. Nessun consiglio medico. Nessun modello di IA è coinvolto, a meno che l'utente non ne aggiunga uno.

## 14. Pagina Qualità dei dati

Conteggi ed elenchi di: uscite senza potenza / senza FC (per mese), campioni di potenza scartati, uscite in cui il TSS calcolato da te differisce > 2 %, picchi di frequenza cardiaca sopra il massimo configurato, bici da confermare, pause senza motivo, giorni senza sonno/HRV/FC a riposo (calcolati sull'intervallo effettivo dei dati dell'orologio), più l'elenco dei **limiti noti** (il peso è un valore di profilo; la stima dell'FTP non è un test; le salite sono un algoritmo; il riscaldamento della CTL; l'API non espone il cambio).

## 15. Indicatori fisiologici a gauge e tabelle di riferimento

Quattro gauge a semicerchio (in stile contachilometri) e un insieme di barre. Ognuno dice con che cosa è confrontato e non dà mai un verdetto.

| Indicatore | Fonte e scala |
|---|---|
| VO₂max (Garmin) | la stima dell'orologio (tabella `vo2max`). Livello (scarso, sufficiente, buono, eccellente, superiore) per sesso e fascia d'età dalla tabella di Garmin, basata sulle classificazioni del Cooper Institute, a `https://www.garmin.com/it-IT/blog/fitness/vo2max/`: leggi la pagina, citala, non inventare mai le soglie e non copiare la tabella nel repository. Età dalla data di nascita esatta. La lancetta dentro una fascia è indicativa |
| HRV notturna (Garmin) | ultima notte, media di 7 notti e **la fascia di equilibrio personale** (`balanced_low`, `balanced_upper`): tre zone (sotto, dentro, sopra) e lo stato di Garmin così com'è. La finestra dell'andamento disegna la fascia come ombra, con la scelta del periodo |
| FTP (Garmin) | l'FTP dell'ultima uscita (`activities.ftp`), la sua variazione dall'inizio del periodo e i W/kg; il valore non dipende dal periodo |
| FC a riposo, Body Battery, punteggio del sonno | nessuna scala esterna: confronto con il solito dell'utente, la media ± 1 deviazione standard dei 60 giorni precedenti (servono almeno 14 giorni), tre zone. Dì che non è una soglia clinica |

**Potenza per età e sesso.** Tabella `ref_power_age`: W/kg a 5 s, 1 min, 5 min e 20 min, con le colonne «10 % inferiore», «media» e «10 % superiore» per sei fasce d'età, da 20–29 a 70–79. Vengono dal video di GCN Italia «Quanto sei in forma per la tua età?» (`https://www.youtube.com/watch?v=B3uTsYNu8V0`). Non scaricare il video: chiedi all'utente gli screenshot delle tabelle e trascrivili. Registra il video come fonte, di' che lo studio originale non è indicato e che il significato di «10 %» è la dicitura del video, **non copiare i numeri nel repository**, e controlla la trascrizione (inferiore < media < superiore, valori in calo con l'età). Sulla carta d'identità disegna una barra per durata dal 10 % inferiore al 10 % superiore con il miglior valore di sempre come segno, e la posizione a parole.

## 16. Controlli incrociati con i valori di Garmin

Prima di tenere un algoritmo tuo, confrontalo con ciò che Garmin già salva. Risultati della build di riferimento, su un'uscita (controlla la tua, non fidarti di questi):

- **Tempo nelle zone di frequenza cardiaca:** il file FIT ha messaggi `time_in_zone` (numero globale 216, sconosciuto a `fitparse`: leggi i campi numerici). Il record a livello di sessione ha `reference_mesg` 18; il campo 2 è il tempo per zona cardio in millisecondi e il campo 6 i confini delle zone. Il nostro conteggio coincideva entro l'1 %, quindi tieni il calcolo tuo e usa quello di Garmin come prova.
- **Tempo nelle zone di potenza:** i confini di Garmin (campo 9) li decide il dispositivo e sono diversi da quelli di Coggan; i due non sono confrontabili. Tieni le zone standard e dì quali sono.
- **Salite:** i messaggi `climb_pro` (numero 317) vengono da un percorso caricato sul dispositivo, non dal riconoscimento delle salite nell'uscita, quindi non sono un controllo alla pari. Tieni l'algoritmo tuo e non considerare verificata la decodifica di quei campi.
- **Training Effect:** aerobico e anaerobico, scala 0–5, stanno nel messaggio di sessione del FIT (`total_training_effect`, `total_anaerobic_training_effect`) e coincidono con l'API; mostrali come riquadro KPI del dettaglio uscita.
- **Potenza normalizzata, TSS e FTP** vengono da Garmin; NP e TSS calcolati da te servono solo come controllo (segnala una differenza oltre il 2 %).
