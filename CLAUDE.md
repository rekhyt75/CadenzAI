# CLAUDE.md — CadenzAI plan / plan di CadenzAI

## English

You are about to build **CadenzAI**, a local, single-user cycling-analytics dashboard fed by the user's own Garmin data, from the specification in `plan/en/` (or `plan/it/` if the user writes in Italian — same file names).

1. **Read** `plan/<lang>/00-overview.md` first, then `01` … `09` before writing code (`10` is optional: read it only if the user wants the local AI summary).
2. **Ask before building**: list contradictions or gaps in the plan and ask the customisation questions of `00-overview.md` §4 in **one short round**; use the defaults for the rest and say so.
3. **Build phase by phase** (§6 of the overview). After each phase run its acceptance checks on the user's **real data** and report honestly what you verified, what you could not verify and what failed.
4. **Secrets and privacy:** never print, log, commit or transmit the Garmin API key (it lives in `~/.config/garmin/connect-personal-api-key`, mode 600). Never commit the database, FIT files, Garmin export, `.venv` or `node_modules`. Map tiles: only OpenStreetMap, with attribution and an offline fallback. Treat downloaded files as untrusted. Do not bypass bot protections.
5. **Honesty rules:** never invent a missing value (show N/D); label estimates; separate observation, hypothesis and recommendation; do not confuse correlation with causation.
6. Answer in the user's language. Before destructive or outward-facing actions (deleting data, pushing to a remote, installing third-party tools that receive the key), ask.
7. Record the user's decisions that the code cannot show (see `07-operations-and-quality.md`, "Project memory").
8. **No personal data in the code.** Weight, maximum heart rate, sex, birth date and bike mass go in the profile table, edited from a "User profile" page; read from Garmin what the API gives (heart-rate zones). A missing value is N/D, never a guess.

## Italiano

Stai per costruire **CadenzAI**, una dashboard ciclistica locale a utente singolo alimentata dai dati Garmin dell'utente, a partire dalla specifica in `plan/it/` (o `plan/en/` se l'utente scrive in inglese: i file hanno gli stessi nomi).

1. **Leggi** prima `plan/<lingua>/00-overview.md`, poi da `01` a `09`, prima di scrivere codice (`10` è facoltativo: leggilo solo se l'utente vuole il riassunto IA locale).
2. **Chiedi prima di costruire**: elenca contraddizioni o lacune del plan e fai le domande di personalizzazione di `00-overview.md` §4 in **un solo breve giro**; per il resto usa i valori predefiniti e dillo.
3. **Costruisci per fasi** (§6 della panoramica). Dopo ogni fase esegui i controlli di accettazione sui **dati reali** dell'utente e riferisci con onestà cosa hai verificato, cosa non hai potuto verificare e cosa è fallito.
4. **Segreti e privacy:** non stampare, registrare nei log, includere in un commit né trasmettere la chiave API di Garmin (sta in `~/.config/garmin/connect-personal-api-key`, permessi 600). Non includere mai in un commit database, file FIT, export Garmin, `.venv` o `node_modules`. Mappe: solo tessere OpenStreetMap, con attribuzione e ripiego offline. Tratta i file scaricati come non fidati. Non aggirare le protezioni anti-bot.
5. **Regole di onestà:** non inventare mai un valore mancante (mostra N/D); etichetta le stime; separa osservazione, ipotesi e raccomandazione; non confondere correlazione e causa.
6. Rispondi nella lingua dell'utente. Prima di azioni distruttive o verso l'esterno (cancellare dati, fare push su un remoto, installare strumenti di terzi che ricevono la chiave) chiedi.
7. Registra le decisioni dell'utente che il codice non mostra (vedi `07-operations-and-quality.md`, «Project memory»).
8. **Nessun dato personale nel codice.** Peso, frequenza cardiaca massima, sesso, data di nascita e massa della bici vanno nella tabella del profilo, modificata da una pagina «Profilo utente»; leggi da Garmin ciò che l'API dà (zone cardio). Un valore mancante è N/D, mai una stima inventata.
