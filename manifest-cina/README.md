# Manifest Cina — Dashboard spedizioni fornitori

Dashboard interna, pubblicata come Artifact Claude, che sostituisce il flusso
manuale del file `Quotazioni Spedizioni Fornitori.xlsx` per la gestione delle
spedizioni fornitori dalla Cina (aereo / nave / treno / camion).

**Pagina live:** https://claude.ai/code/artifact/de78184c-aac1-41dc-a55f-7b3fcf38c3c3
(privata, visibile solo a chi ha il link — condividila con chi in Redmade deve usarla).

## Perché un Artifact e non Power Apps / n8n

Questo repo contiene già un workflow n8n (`fedex_customs_workflow.json`) basato
su SharePoint + Outlook + Graph API per lo sdoganamento FedEx. Per questa
dashboard è stata scelta una strada diversa, discussa e concordata con l'utente:

- **Nessuna licenza aggiuntiva** (Power Apps/Power Automate premium, AI Builder)
  né configurazione OAuth: la pagina gira subito, autenticata con l'account
  Claude di chi la apre.
- **Database e AI integrati nella pagina stessa** (capability `db` e `sample`
  della piattaforma Artifact): niente liste SharePoint da creare a mano, niente
  Azure Function per l'estrazione.
- **Uso interno vero e proprio**: la pagina è persistente, condivisa da tutti
  i viewer, con drag&drop reale nel browser — non una demo.

Il limite attuale: la pagina **non può inviare o leggere email in autonomia**
(nessun accesso SMTP/Graph da dentro l'Artifact). Per questo l'invio delle
richieste di offerta e la lettura delle risposte sono assistiti ma non
automatici — vedi "Cosa manca per l'automazione completa" sotto.

## Analisi del file originale

`Quotazioni Spedizioni Fornitori.xlsx`, foglio `Quotazioni`: 756 spedizioni
compilate su un'unica tabella A1:AC1427, più il foglio `CheckFatture` che
confronta la miglior quotazione con l'importo fatturato dallo spedizioniere.

Problemi strutturali che la nuova dashboard risolve:

- **Una colonna per spedizioniere** (`AVION`, `VENTANA`, `LOGWIN`, `FEDEX`,
  `SOGEDIM`, `LEONARDI`, `SIFTE BERTI`, `TRENO`, `TNT`, `SITTAM`, ripartiti in
  gruppi Aerea / Nave-Treno / Camion) invece di una riga per offerta →
  nella dashboard le offerte sono una sottocollezione `quotes` per spedizione.
- **Fornitore e modalità nello stesso campo** (es. `UPOWERTEK SEA`, `JHD TRAIN`,
  `LEDLINK AIR`) → separati in `fornitore` e `modalita`.
- **Tracking N° generico**, con dentro booking, ETA, ETD, riferimenti nave
  mescolati (es. `MX/2021/400355`, `CSCL SATURN 069W`) → campi distinti
  `booking`, `awbBl`, `trackingNumber`, `etd`, `eta`.
- **Nessuna data spedizione strutturata** → ogni spedizione ha `createdAt`/`updatedAt`.
- **Nessuno stato spedizione** → workflow a 8 stati (Bozza → Quotazioni richieste
  → Offerte ricevute → Da confermare → Booking → In transito → Consegnata → Chiusa).
- **Nessun campo Packing List dedicato** → `packingListNumber` proprio.

La logica di confronto quotazione/fattura del foglio `CheckFatture` resta da
reintrodurre in una fase successiva (vedi Roadmap).

## Modello dati (Artifact `db`)

| Collezione | Contenuto |
|---|---|
| `shipments/{id}` | Una spedizione: `code` (`RM-AAAA-NNNN`), `stato`, `modalita`, `fornitore`, `invoiceNumber`, `packingListNumber`, `incoterm`, `localitaRitiro`, `pesoKg`, `volumeCbm`, `numColli`, `spedizioniereScelto`, `booking`, `awbBl`, `trackingNumber`, `etd`, `eta`, `quotesCount`, `history[]`, `isExample` |
| `shipments/{id}/quotes/{id}` | Un'offerta ricevuta: `spedizioniere`, `prezzo`, `valuta`, `transitTime`, `note`, `ricevutaIl` |
| `forwarders/{id}` | Anagrafica spedizionieri: `nome`, `modalita[]`, `email`, `attivo` |
| `suppliers/{id}` | Anagrafica fornitori: `nome`, `storicoCount`, `akaNames[]` (varianti di grafia viste nel file storico), `note` |
| `meta/counters` | Contatore progressivo per il codice spedizione |

Dati seed reali importati dal file (non inventati):

- **10 spedizionieri**, con modalità dedotte dai gruppi di colonne del file
  (Aerea / Nave-Treno / Camion). **`TRENO` è un nome segnaposto**: nel file
  storico la colonna treno non riporta un nome azienda — va corretto in
  "Spedizionieri & Fornitori" nella dashboard.
- **30 fornitori**, nomi ripuliti dal suffisso modalità (AIR/SEA/TRAIN) e da
  varianti minori (es. `UNIHOME`→`UNI-HOME`, `YINGJAO`→`YINGJIAO`), con
  `akaNames` per audit. Le 756 spedizioni storiche **non** sono state
  migrate come righe `shipments` (scelta esplicita per l'MVP): il vecchio
  Excel resta l'archivio storico.
- 2 spedizioni di esempio (`isExample:true`, badge "Esempio" in dashboard),
  rimovibili con un clic da "Spedizionieri & Fornitori" quando i dati reali
  iniziano ad arrivare.

## Cosa fa oggi la dashboard

1. **Nuova spedizione** → drag&drop CI/PL (PDF anche scansionati, o foto) +
   scelta aereo/nave/treno/camion → Claude estrae fornitore, invoice, packing
   list, incoterm, peso, volume, colli, con affidabilità per campo — sempre
   editabile prima di salvare.
2. **Richiesta quotazioni** → la pagina genera un'email pronta (oggetto con
   codice spedizione `RM-AAAA-NNNN`) per ogni spedizioniere idoneo alla
   modalità; l'invio resta dal client di posta dell'utente (link `mailto:`
   o testo da copiare).
3. **Offerte** → si incolla il testo della risposta email, Claude estrae
   prezzo/valuta/transit time; confronto automatico con evidenza del prezzo
   migliore.
4. **Conferma** → un clic sull'offerta scelta imposta lo stato Booking.
5. **Tracking** → si incolla la mail di booking dello spedizioniere, Claude
   estrae booking/AWB-BL/tracking/ETD/ETA; stato passa a In transito.

## Cosa manca per l'automazione completa

L'invio delle RFQ e la lettura delle risposte sono **assistiti da AI ma non
automatici**, perché l'Artifact non ha accesso diretto alla casella email.
Per chiudere il ciclo (invio automatico, lettura automatica delle risposte in
arrivo, scrittura diretta nel database della dashboard) serve collegare una
casella email reale. Opzioni, da discutere con l'utente prima di procedere:

- **Connettore Microsoft 365** (coerente con SharePoint/Outlook già in uso):
  richiede che l'utente autorizzi il connettore `ms365` nelle impostazioni
  claude.ai — non completabile da una sessione non interattiva.
- **n8n** (stesso motore del workflow dogana FedEx già nel repo): richiede
  analogamente l'autorizzazione del connettore `n8n`.

Una volta autorizzato uno dei due, il passo successivo è un flusso che:
monitora la casella `spedizioni@redmade.it` (o quella scelta), fa il match
delle risposte tramite il codice `RM-AAAA-NNNN` nell'oggetto, e scrive le
offerte/il tracking direttamente nel database della dashboard (stessa API
`write_db` usata per il seed di questo documento).

## Roadmap

- **Fatto (MVP)**: dashboard, estrazione documenti, RFQ assistite, confronto
  offerte, conferma, tracking assistito.
- **Prossimo**: collegare la casella email per l'invio/lettura automatica.
- **Dopo**: reintrodurre il confronto quotazione/fattura finale (come
  `CheckFatture` nel vecchio file) e un export Excel di compatibilità.
- **Eventuale, su richiesta**: migrazione delle 756 spedizioni storiche in
  `shipments`/`quotes` (colonne spedizioniere → righe), tenuta fuori
  dall'MVP su scelta esplicita dell'utente.
