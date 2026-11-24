# 05 · Bici, abbinamento automatico della bici, componenti e usura

## 1. Parco Bici

Una tabella `bikes` che l'utente modifica dall'interfaccia (non scrivere mai le bici nel codice). Campi: `name` (univoco), `kind` (`mtb`, `gravel`, `road`, `indoor`, `other`), `status` (`active`, `for sale`, `sold`), `bought_on`, `sold_on` (opzionale), `price` (acquisto), `list_price`, `sold_price`, `notes`, `color_slot`.

- **Il colore è un'identità:** a ogni bici viene assegnato uno slot di colore categorico (1–8) alla creazione e lo **mantiene per sempre**; aggiungere o vendere una bici non deve mai ricolorare le altre. Usa l'ordine categorico validato (vedi `09`).
- **Aggiungi / modifica / elimina:** un pulsante "+ Aggiungi bici" (Officina → Parco Bici e Componenti) apre una finestra modale. Una bici con uscite o componenti non può essere eliminata (il server risponde 400 con un messaggio chiaro): l'utente la segna invece come **venduta**. Rinominare una bici aggiorna le sue uscite e i suoi componenti.
- Una bici **venduta** è nascosta/compressa per impostazione predefinita nella pagina dei componenti, ma la sua storia resta.
- La pagina del parco bici mostra per ogni bici: uscite, km, ore, dislivello, uscite con una bici da confermare; un grafico a barre impilate mensile dei km per bici; pillole di stato.

## 2. Assegnare una bici a ogni uscita

La Garmin Personal API **non** espone su quale bici è stata registrata un'uscita (vedi `01`). Regole:

**Uscite storiche** (dall'export dei dati, `*_gear.json`): usa l'assegnazione di Garmin. Se l'utente ha una regola che non concorda con Garmin per alcune uscite (ha dimenticato di sincronizzare il cambio, ecc.), lascia **decidere all'utente**: segna quelle uscite come *incerte* e mostrale con un bordo rosso finché non vengono confermate.

**Nuove uscite** (ogni nuova uscita deve avere una bici; in automatico, in quest'ordine):

1. **Evidenza dai sensori (certa).** Costruisci, dalle uscite la cui bici è affidabile (quella di Garmin o confermata dall'utente), l'insieme dei sensori esterni (produttore + numero di serie; ignora la fascia cardio, indossata dal ciclista, e il ciclocomputer) e tieni quelli visti su **una sola bici** in **almeno 3 uscite**. Se una nuova uscita contiene tali sensori e puntano tutti a una bici → assegnala, `bike_uncertain = 0`, e scrivi il motivo in `bike_note` ("riconosciuta dal sensore X, visto solo su questa bici in N uscite"). Se la bici indicata è venduta o non ancora acquistata alla data dell'uscita, assegnala ma segnala come incerta. I sensori condivisi (un misuratore di potenza spostato tra bici) non portano informazione.
2. **Tipo di uscita e bici in uso (da confermare).** Tra le bici del tipo corrispondente (`gravel_cycling`→gravel, `mountain_biking`→mtb, `road_biking`/`cycling`→road, `indoor_cycling`/`virtual_ride`→indoor) già possedute alla data dell'uscita e non vendute, prendi quella acquistata più di recente. Segnala come incerta.
3. **Ripiego (da confermare):** la bici non venduta usata più di recente. Non lasciare mai un'uscita senza bici se esiste almeno una bici.

Un'uscita che l'utente assegna o conferma a mano imposta `bike_manual = 1` e **non viene mai più sovrascritta**. Rieseguire l'assegnazione tocca solo le uscite **senza** bici.

**Validazione della regola 1:** leave-one-out sulla storia (ricalcola l'evidenza dei sensori senza l'uscita in esame): nella build di riferimento 97 uscite su 97 con un sensore esclusivo di una bici sono state identificate correttamente.

## 3. Pagina "Uscite da abbinare"

Elenca le uscite con `bike_uncertain = 1` (o senza bici): data, titolo con link, tipo, km, **perché** serve una conferma, un selettore della bici e un pulsante "Conferma". Stato vuoto: un breve messaggio. La barra laterale mostra un badge con il conteggio. La conferma è immediata e persistente.

## 4. Componenti e usura

### 4.1 Cosa registra l'utente (per bici)

`kind` (catena, cassetta, corona, pastiglie freno anteriori/posteriori, dischi anteriori/posteriori, copertone anteriore/posteriore, lattice, liquido freni anteriore/posteriore, revisione dei foderi della forcella, revisione completa della forcella, revisione ad aria dell'ammortizzatore, revisione completa dell'ammortizzatore, più "altro" con un nome libero), `model`, `installed_on`, `price`, `vendor`, `notes`, fino a quattro soglie (`km`, `moving hours`, `climbing m`, `days`) e un `removed_on` opzionale. Salvali nel DB; mostra N/D per prezzo/venditore sconosciuti (non inventare mai).

**Anteriore/posteriore sono separati** per pastiglie, dischi, copertoni e liquido freni.

### 4.2 Registrare i componenti di una nuova bici dalla scheda tecnica

Quando l'utente indica il modello di una bici, cerca la **scheda tecnica del produttore** e precompila ogni componente alla **data di acquisto** (telaio, forcella, ammortizzatore, trasmissione, freni, rotori, ruote, copertoni, avantreno, reggisella, sella…), segnando gli elementi senza soglia di usura come "registrato senza soglia". Regole:

- Preferisci la pagina del produttore; **non aggirare le protezioni anti-bot**. Se è bloccata, dillo; chiedi all'utente di aprire la pagina nel suo browser (poi leggi quella scheda) oppure usa due rivenditori/database pubblici indipendenti e confrontali.
- Registra la `source` di ogni stringa di modello. Se il rivenditore dell'acquirente ha cambiato un pezzo (ad es. le ruote), **prevale la dichiarazione dell'utente** e la fonte lo indica.
- Non inserire mai un modello non verificato in una riga di componente; scrivi invece "non nella scheda tecnica".

### 4.3 Le soglie sono stime

Offri un modello di soglie **indicative** (dichiara che **non sono dati del produttore**) e lascia che l'utente le modifichi. Valori di riferimento usati dalla build di riferimento: catena 2 500 km, cassetta 7 500 km, corona 10 000 km, pastiglie anteriori 1 800 km, pastiglie posteriori 2 800 km, disco anteriore 8 000 km, disco posteriore 10 000 km, copertone anteriore 3 000 km, copertone posteriore 2 200 km (gomma XC leggera), lattice 90 giorni, liquido anteriore 540 giorni, liquido posteriore 730 giorni, revisione dei foderi della forcella **100 h o 365 giorni** (quello che arriva prima), revisione completa della forcella 730 giorni, revisione ad aria dell'ammortizzatore 100 h o 365 giorni, revisione completa dell'ammortizzatore 730 giorni. Cita da dove provengono gli intervalli e quanto è debole la fonte (ad es. una vecchia tabella del produttore più la pagina di un rivenditore); la prima versione della build di riferimento aveva ipotizzato 50 h per le sospensioni e generò un allarme dopo sei settimane: evita cifre prive di fondamento.

### 4.4 Due indicatori di usura

Per ogni componente, dalla data di installazione, usando solo le uscite di **quella bici**:

1. **Usura sulla soglia** (come impostata): il **massimo** tra le soglie definite di `used / threshold`, dove used = km, **ore in movimento** (non il tempo trascorso: le soste non muovono una sospensione), dislivello, giorni.
2. **Indice d'uso (sensibile al percorso):** lo stesso consumo pesato per ciò che il percorso ha effettivamente fatto al pezzo. La soglia è la vita del componente **al mix medio di uscite del ciclista**; un'uscita con il doppio di frenata per km consuma i freni due volte più in fretta.

```text
index = mean over drivers d of  Σ_since_install(driver_d) / ( ref_d · threshold )
ref_d = (Σ over the whole history of driver_d) / (Σ over the whole history of the threshold unit)
threshold unit priority: km, else moving hours, else days (days use the history span)
```

Fattori (dichiarali nell'interfaccia; sono scelte di modellazione):

| Componente | Fattore/i |
|---|---|
| catena, cassetta, corona | lavoro in pedalata (kJ) |
| pastiglie, dischi, liquido freni | carico frenante stimato (kJ, vedi `04` §7) |
| copertone anteriore | km e carico frenante |
| copertone posteriore | km, lavoro in pedalata e carico frenante |
| forcella, ammortizzatore | ore in movimento e ore trascorse in discesa |
| lattice, ruote, deragliatore, avantreno, ecc. | nessuno (solo tempo, oppure non monitorato) |

Il carico frenante **non è suddiviso tra anteriore e posteriore**; la differenza tra i pezzi anteriori e posteriori sta nelle loro soglie. I componenti senza un fattore dipendente dal percorso mostrano "non applicabile".

### 4.5 Avvisi

Un componente è in "attenzione" al **≥ 80 %** e "oltre" al **≥ 100 %** del **maggiore** dei due indicatori; indica quale ha fatto scattare l'avviso. Mostra: un indicatore con icona ed etichetta di testo (mai solo il colore: ✓ entro il limite, ▲ oltre l'80 %, ● oltre il 100 %), un banner in cima alla pagina e un badge nel menu.

### 4.6 Comportamento della pagina

- Una **sezione a fisarmonica per bici** (stato ricordato), con il nome della bici, un **pulsante "+" a destra del nome** che apre una finestra modale *Aggiungi componente* (tipo dall'elenco dei modelli, modello, data di installazione, prezzo, venditore, soglie, note), una pillola di stato e un pulsante "Modifica bici".
- Ogni tabella è **ordinabile per colonna**; l'**ordinamento predefinito è dal più usurato** (massimo decrescente dei due indicatori; i componenti senza valore per ultimi; quelli rimossi sempre in fondo; la scelta è ricordata, con un pulsante "ripristina").
- Una **finestra modale di aiuto "?"** (non un blocco in cima alla pagina) spiega i due indicatori, i fattori, le ipotesi (massa, Crr, CdA) e un breve glossario (revisione dei foderi, revisione del manicotto ad aria, revisione completa).
- I **componenti di serie** (flag `original`) non hanno prezzo né rivenditore: fanno parte del prezzo della bici e non sono mai contati come «prezzo mancante».
- **Rimuovere un componente chiede cosa fare** (pulsante «Rimuovi»): *sostituito* (resta nello storico di manutenzione con la sua data; non conta più per gli avvisi né per la rivendita, e il suo costo resta nel costo totale) oppure *elimina del tutto* (una prova o un errore: sparisce e il suo prezzo esce dal costo totale). Un componente sostituito offre solo «Rimonta» ed «Elimina».
- Una **finestra modale "Modifica"** per componente (modello, data di installazione, prezzo, venditore, casella «è un upgrade» e valore aggiunto (`is_upgrade`, `upgrade_value`), soglie, note, segna come rimosso, elimina con conferma).
- L'aggiornamento avviene automaticamente: l'usura è **calcolata al momento della richiesta** dalle uscite della bici dall'installazione, quindi una nuova uscita (o un'assegnazione della bici corretta) si riflette subito. **Test:** rimuovi un'uscita → i numeri calano; risincronizza → tornano esattamente come prima.

## 5. Valore della bici: quattro voci distinte

Ogni scheda bici nella pagina del parco bici mostra quattro voci (un pulsante **«?»** le spiega). Se mancano dati compare **N/D** con ciò che manca, mai un numero inventato.

| Voce | Definizione |
|---|---|
| Prezzo di listino | il prezzo a nuovo del modello base (`list_price`, inserito dall'utente) |
| Costo totale | prezzo di acquisto **più tutto ciò che hai speso sui componenti** (consumi, servizi, upgrade). Cresce sempre: è quanto è costata la bici nel tempo |
| Valore a nuovo con upgrade | listino più il valore aggiunto dagli **upgrade** montati e non rimossi. I consumi (pastiglie, copertoni, catena) **non** lo aumentano: nessuno vende una bici senza pastiglie |
| Rivendita stimata | listino × la quota di valore trattenuta all'età della bici, più gli upgrade, mostrata come **intervallo min–max** (il numero grande è il centro). Una bici venduta mostra il vero `sold_price` se inserito |

Un componente è un upgrade solo se l'utente spunta **«È un upgrade»** (`is_upgrade`); il suo `upgrade_value` facoltativo (vuoto = il suo prezzo) è il valore che aggiunge a nuovo.

**Modello di svalutazione: usane uno con fonte e dì da dove viene.** Fonte verificata: buycycle, «Quanto posso vendere la mia bici?» (`https://buycycle.com/blog/it/post/quanto-posso-vendere-la-mia-bici/`). Dà il valore trattenuto sul listino per fascia d'età e la regola «20–40 % perso nel primo anno, poi 10–15 % l'anno»:

| Età | Trattenuto |
|---|---|
| meno di 1 anno | 70–80 % |
| 1–2 anni | 60–70 % |
| 3–4 anni | 45–60 % |
| 5–6 anni | 35–50 % |
| 7 anni o più | 25–40 % |

Dichiara cosa è un'ipotesi **tua** e non della fonte: ogni fascia vale al centro del suo intervallo di età con una retta tra i punti; oltre 8 anni −12 % l'anno con un minimo del 15 %. La fonte **non** distingue MTB, gravel e strada e non dà **alcuna percentuale per gli upgrade**: un recupero come il 20–40 % del valore dell'upgrade è un'ipotesi non verificata, quindi etichettalo così nell'interfaccia, oppure chiedi all'utente. Non usare come dati cifre senza fonte prese dalla risposta di un assistente: apri la pagina e controlla. Ignora chilometri, condizioni, taglia e mercato locale, e dillo. Il vero listino dovrebbe essere lo street price al momento della vendita.

**I componenti da cambiare abbassano un po' la rivendita.** Un acquirente scala il costo per sostituire ciò che è consumato, anche se una catena nuova non farebbe salire il valore. Per ogni componente con prezzo e non rimosso detrai il suo prezzo in proporzione all'usura: nulla fino all'80 % di usura, poi in linea retta fino al prezzo pieno al 100 %. Questa regola è una tua ipotesi, non della fonte: dillo. I componenti di serie non hanno prezzo, quindi se sono consumati segnalalo soltanto e non detrarre nulla.
