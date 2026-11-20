# 08 · Confronto con GoldenCheetah, roadmap e cosa non si fa

Questa sezione documenta perché l'app di riferimento contiene ciò che contiene. La confronta con **GoldenCheetah** (software desktop open source per l'analisi del ciclismo, GPL-3.0; la v3.8 è stata rilasciata il 20 Sep 2026). Il confronto si basa sul suo README, sulle note di rilascio e sulle pagine della guida utente, **non su un uso diretto**; il glossario completo delle metriche non è stato letto. Ricontrolla prima di farci affidamento.

## 1. Dove ciascuno è più forte

| Area | GoldenCheetah | Questa app |
|---|---|---|
| Dati in ingresso | importazione di file e servizi cloud (Strava, Tredict, Withings, TodaysPlan…); nessun collegamento diretto a Garmin Connect trovato nelle fonti lette | sincronizzazione automatica e incrementale dalla Garmin Personal API a ogni avvio; file FIT originali conservati |
| Carico | PMC con diverse metriche di stress (BikeStress, TRIMP, RPE), modello di Banister, "PMC atteso" dagli allenamenti pianificati | CTL/ATL/TSB sul TSS di Garmin, rapporto ATL/CTL, fasce di pausa |
| Curva di potenza | mean-max con modelli di potenza critica, bilancio W′ | curve per anno/periodo, CP a 3 punti, stima dell'FTP |
| Singola uscita | grafico completo dell'attività, mappa, istogrammi, intervalli, quadranti forza/velocità, FC vs potenza, stima del CdA | pagina di dettaglio dell'uscita (grafici sincronizzati, percorso, giri, zone, migliori sforzi, quadranti di coppia, disaccoppiamento) — costruita dopo il confronto |
| Dinamica di pedalata | analisi a quadranti | memorizzata e mostrata per uscita e come medie |
| Pulizia dei dati | ~20 processori (picchi anomali di potenza/FC, GPS, buchi, duplicati da ruota libera, altitudine…) | filtro dei picchi anomali di potenza, pagina della qualità; picchi anomali di FC e correzioni GPS **non** costruiti |
| Recupero | HRV e peso tramite servizi esterni | sonno, HRV, FC a riposo, Body Battery da Garmin, correlati al carico; asse del recupero nel radar |
| Attrezzatura | traccia l'uso dell'attrezzatura | componenti per bici con soglie **e** un indice d'uso che tiene conto del percorso, avvisi, abbinamento delle bici basato sui sensori |
| Statistiche | scripting in R/Python | confronto tra periodi integrato, con test, e correlazioni |
| Pianificazione | calendario, piani, allenamenti, controllo del trainer in modalità ERG (vista Plan della v3.8) | nessuna |
| Estensibilità | metriche definite dall'utente, R, Python | il codice stesso |

Il processore "freewheeling duplicate" di GoldenCheetah rimuove lo stesso artefatto del filtro dei picchi anomali di potenza in `04` §1.1: una conferma indipendente dell'approccio.

## 2. Roadmap (candidati, in ordine di valore)

1. **Bilancio W′** con un modello CP migliore — rilevante per gli scatti ripetuti in XC/MTB; affidabile solo se CP/W′ derivano da veri sforzi massimali (offri un test guidato o l'importazione di un ramp test), altrimenti indicalo chiaramente come stima.
2. **Pulizia dei picchi anomali di frequenza cardiaca e del GPS**, più un'azione "correggi e ricalcola" in Qualità dei dati.
3. **Esportazione** delle tabelle in CSV e un backup con un solo comando del database e della cartella dei FIT.
4. **Filtri avanzati** in Esplora le attività (bici, tipo, intervallo di date, potenza/FC presenti).
5. **Effetto di meteo/temperatura** sull'EF (la temperatura è nel flusso) per spiegare i cali dell'EF.
6. **Pianificazione** (calendario delle uscite pianificate e del carico atteso) — solo se l'utente la chiede.
7. **Importazione dello storico precedente al misuratore di potenza** (uscite più vecchie con metriche di potenza N/D) — solo se richiesta.
8. **Internazionalizzazione:** un dizionario di stringhe e un selettore della lingua.

## 3. Cosa deliberatamente non si fa

Controllo del trainer/ERG ed editor di allenamenti, analisi di corsa e multisport, editor di formule per l'utente, runtime R/Python, molti formati di importazione, caricamento su Strava o altri servizi, stima aerodinamica del CdA (poco valore alle velocità della MTB — non testata), qualsiasi pretesa di "AI" per il Coach (è basato su regole; l'unica IA è il riassunto locale facoltativo di `10`, chiaramente etichettato).

## 4. Limiti onesti da ripetere all'utente

- I riferimenti percentili sono una tabella di popolazione, non corretta per età o disciplina; i migliori sforzi nelle uscite non sono test, quindi soglia, VO₂max e potenza critica sono probabilmente sottostimati.
- Le forme dei tipi nel radar e le soglie di usura sono ipotesi dell'autore.
- La Garmin Personal API non espone le bici; l'abbinamento delle bici è un'inferenza.
- L'indice d'uso e il carico frenante derivano da un modello fisico con ipotesi dichiarate.
- La dimensione dei campioni nei periodi brevi può essere minuscola (ad es. una dozzina di uscite lunghe, una manciata di giorni duri); mostrala.
