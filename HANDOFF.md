# Bilancio — app finanze personali (riepilogo per riprendere il lavoro)

## Obiettivo
App per tracciare tutte le mie spese e finanze, stessa filosofia dell'"app del gommone" (Giornale di Bordo): **un unico file HTML**, mobile-first (iPhone), hosting su **GitHub Pages**, login **email/password** con **Firebase** per sincronizzare i dati. Stile diverso dal gommone: grafite scuro + verde smeraldo (Manrope + JetBrains Mono) invece di blu notte/ottone.

## Stato attuale
- File: `index.html` (un solo file, nessuna dipendenza oltre a Google Fonts e, solo in modalità cloud, l'SDK Firebase compat 10.14.1 caricato da gstatic).
- **Nessun dato dentro il file**: i dati si caricano da CSV (ingranaggio in alto a destra → "Importa da CSV").
- **Due modalità**, decise da `FIREBASE_CONFIG` in cima allo script:
  - `apiKey` vuota → solo locale, `localStorage` (chiave `bilancio:v1`), come la prima bozza.
  - `apiKey` compilata (progetto `contabilit-1a500`) → login email/password + passphrase, dati su Firestore **cifrati end-to-end**.
- Il resto dell'app usa solo `store.get()` / `store.set()` (`localStore` o `cloud`).

## Cloud: come funziona
- Gate prima dell'app: **email + password** (account creato a mano dalla console, nessuna registrazione nell'app; c'è "Password dimenticata?") → **Crea cassaforte** la prima volta / **Sblocca** le volte dopo.
- Password dell'account e passphrase sono due cose diverse e devono avere valori diversi: la password passa da Google, la passphrase non esce mai dal browser.
- Cifratura: AES-GCM 256, chiave da passphrase con PBKDF2-SHA256 600.000 iterazioni, sale casuale. Ogni sezione è cifrata a parte con la sezione come dato associato. Su Firebase non arriva mai nulla in chiaro.
- Firestore:
  - `bilancio/{uid}`: `salt`, `iter`, `check` (la parola `bilancio-ok` cifrata, serve a verificare la passphrase)
  - `bilancio/{uid}/sez/{tx|conti|rate|inv|trading}`: `kid` (= sale attuale), `iv`, `ct`
- Salvataggio ottimistico: la UI cambia subito, l'invio parte dopo 400 ms con le sole sezioni modificate, in un batch; se fallisce riprova con attese crescenti (2 s → 60 s). Puntino in alto: verde sincronizzato, giallo salvataggio in corso, rosso errore. Quando l'app va in background invia subito; quando torna in primo piano ricarica le modifiche fatte da altri dispositivi. Cache offline di Firestore attiva (anche lì solo dati cifrati).
- "Ricorda su questo dispositivo": salva la chiave AES **non esportabile** in IndexedDB (mai la passphrase). "Blocca" la dimentica e chiede di nuovo la passphrase.
- **Cambia passphrase** (Impostazioni): nuovo sale, ricifra tutto in un'unica scrittura atomica. Le regole rifiutano le scritture con `kid` vecchio: un altro dispositivo con la passphrase vecchia viene bloccato e la chiede di nuovo.
- Primo accesso da un browser con dati locali e cloud vuoto: i dati locali vengono caricati nel cloud e poi tolti da `localStorage`.
- Se perdo la passphrase i dati non si recuperano: esportare ogni tanto il backup `.json` (che è in chiaro, da tenere fuori dal repo).

## Configurare Firebase (da fare una volta) — progetto `contabilit-1a500`
1. Authentication → Inizia → Metodo di accesso → attiva **Email/password** (non il link via email).
2. Authentication → Utenti → **Aggiungi utente**: la tua email e una password. Copia l'**UID utente** che compare nella lista.
3. (Consigliato) Authentication → Impostazioni → Azioni utente → disattiva **Abilita la creazione (registrazione)**, così nessun altro può crearsi un account.
4. Firestore Database → crea (modalità produzione, località europea). Regole → incolla `firestore.rules` mettendo il tuo UID al posto di `IL_TUO_UID` → Pubblica.
5. La config web è già in `FIREBASE_CONFIG` dentro `index.html`.
6. (Consigliato) Google Cloud Console → API e servizi → Credenziali → limita la chiave API ai referrer `https://gasbarrigiordano-max.github.io/*`.
7. GitHub → Settings → Pages → deploy dal branch. Sito: https://gasbarrigiordano-max.github.io/contabilit-/ → accedi, crea la passphrase, importa i CSV.

## Schermate (barra in basso)
1. **Panoramica**: saldo mese/anno, liquidità e patrimonio, ciambella uscite per categoria, barre entrate vs uscite, top voci.
2. **Movimenti**: lista con filtri (tipo, mese/anno, ricerca), pulsante + per aggiungere, tocca per modificare/eliminare.
3. **Mensile**: tabella categorie × mesi (replica la tabella "Contabilità" del mio file Numbers), uscite/entrate.
4. **Patrimonio**: riepilogo con composizione (Banche / Cash / Investimenti), conti con logo e quota del patrimonio, saldo modificabile; **Andamento** (grafico del totale con tooltip, 12 mesi / 2 anni / tutto, variazione ultimo mese e da inizio anno) e **Storico mensile** (tocca un mese per il dettaglio dei conti, correggerlo o eliminarlo; "+ Mese passato" per inserire i mesi precedenti).
5. **Altro**: Rate, Investimenti (investito, attuale, rendimento % ricalcolato), Trading (P&L giornaliero per mese). Su iPad/Mac Rate e Investimenti hanno una voce nel menu laterale.

### Carte di credito
- Sezione Carte (in Altro su iPhone, voce nel menu su iPad/Mac), una scheda per carta: ING Oro (plafond 1.500 €, addebito il 10 del mese dopo su ING, PagoFlex attivo) e BBVA (1.000 €, addebito a fine mese). Tutto modificabile, si possono aggiungere carte.
- Spese registrate con "+ Spesa con …" (o "Pagato con" nel movimento): sono uscite normali alla loro data; il giorno dell'addebito il totale del mese (spese + rate PagoFlex) viene tolto dal conto della carta.
- **Disponibile = plafond − spese non ancora addebitate − capitale PagoFlex ancora da restituire.**
- PagoFlex (ING): su spese ≥ 200 € (minimo modificabile), 3/6/9/12 rate (o altro numero), commissione mensile 0,5% dell'importo (modificabile; da verificare sul contratto). La commissione diventa un'uscita "Carte › Commissioni PagoFlex" a ogni addebito; il capitale no, perché la spesa è già tra le uscite.
- Rate con "Pagata da" una carta: alla scadenza diventano spese della carta e lasciano il conto con l'estratto.

### Rate
- "+ Nuova rata" e "Servizi" in alto. In Da pagare le rate sono raggruppate per servizio; i servizi senza rate in corso restano nascosti e ricompaiono con una nuova rata.
- Due schede: **Da pagare** (piani in corso, con prossima scadenza e avanzamento) e **Pagate** (piani conclusi + storico dei pagamenti registrati).
- Nuova rata: metodo, prodotto, data acquisto, importo totale, numero rate, rate da pagare, data prossima rata, conto da cui viene pagata.
- **Pagamento automatico**: all'apertura dell'app (e ogni ora se resta aperta) ogni rata scaduta, oggi compreso, diventa un'uscita `Rate › <metodo>` con la data di scadenza, viene tolta dal saldo del conto scelto e segnata pagata. Il movimento ha id fisso `rt_<rata>_<n>` per non duplicarlo tra dispositivi.

## Modello dati (stato `S`)
- `tx`: `{id, d:'YYYY-MM-DD', t:'u'|'e', cat, sub, amt, note, carta?, pag?, conto?}`: `carta` = carta di credito (va nell'estratto conto); `pag` = 'carta' (debito/prepagata) | 'conto' (bonifico, bollettino) | 'contanti' con `conto`: l'importo esce subito dal saldo del conto e torna indietro se la spesa viene modificata o eliminata: uscite con macro-categoria + sottovoce (Mezzi › Gommone, Casa › Spesa, Lavoro › Trading…); entrate per fonte + voce (Porto › CostaMed, Locali › Morgan…)
- `conti`: `{id, g:'Banche'|'Cash'|'Investimenti', n, v}`
- `rate`: `{id, tipo, prod, acq, tot, n, rata, pre, done, first, fi, conto}`: piano di n rate mensili (`rata` = tot/n, l'ultima assorbe gli arrotondamenti). `pre` = già pagate prima di inserirla, `done` = pagate dall'app; la rata i scade `first` + (i − `fi`) mesi. `conto` = id del conto da cui escono i soldi. Il vecchio formato `{tipo, prod, rata, res}` viene convertito da solo (senza data: va completato con Modifica).
- `inv`: `{n, lvl:0|1|2, inv, eff}` (i totali usano solo `lvl 0`)
- `carte`: `{id, n, plafond, tipo:'giorno'|'fine', giorno, conto, pf, pfMin, pfComm, dal, flex:[], es:[]}`. Le spese con carta sono movimenti con `carta`. Ciclo = mese della spesa; addebito il `giorno` del mese dopo (ING Oro, 10) o a fine mese (BBVA). `dal` = primo mese gestito (prima si considera già addebitato). `flex` = PagoFlex `{id, tx, desc, imp, n, comm, m0, done}`; `es` = estratti addebitati `{m, d, spese, pf, tot, conto}`.
- `opz`: impostazioni/flag (`contiBase`: conti ING, BBVA, N26, Buddy Bank, Wise creati una volta se mancavano; `storicoDal`: primo mese seguito dallo storico automatico)
- `storico`: una foto del patrimonio per mese `{m, d, tot, g:{Banche,Cash,Investimenti}, conti:[{n,g,v}], tipo}`; tipo `fine` (ultimo giorno del mese, aggiornata a ogni controllo di quel giorno), `tardi` (prima apertura dopo, saldi riportati a fine mese togliendo i movimenti registrati dall'app nel frattempo), `ora` (salvata a mano, poi sostituita da quella di fine mese), `inserito` (mese passato a mano o da CSV, mai sovrascritto)
- `servizi`: elenco dei servizi delle rate (PayPal, Klarna, Scalapay, Findomestic, Compass…), modificabile.
- `trading`: `{id, d, v}`

## Come lavoro (dal mio file Numbers 2026)
Fogli: CONTABILITA (uscite per macro-categoria/sottovoce × 12 mesi, entrate, situazione finanziaria con Liquidità e Patrimonio), Rate (PayPal), RIEPILOGO INVESTIMENTI, P&L (trading giornaliero, vuoto), SPESE CARTA DI CREDITO (vuoto). Ho 239 movimenti mensili storici da importare: le celle mensili sono state convertite in movimenti datati il giorno 1 di ogni mese.

## Import CSV (già implementato)
Separatore `;` o `,`, decimali con virgola, date `YYYY-MM-DD` o `DD/MM/YYYY`.
- Movimenti: `data;tipo;categoria;voce;importo;nota` (tipo = uscita|entrata)
- Conti: `gruppo;nome;saldo`
- Rate: `metodo;prodotto;rata;residue;prossima rata;conto` (ultime due facoltative; conto = nome del conto in Patrimonio)
- Investimenti: `voce;livello;investito;attuale`
- Storico patrimonio: `mese;banche;cash;investimenti` (mese `2026-03`, `03/2026` o `01/03/2026`)
- Trading: `data;importo`
Opzione "Sostituisci" per cancellare i dati di quel tipo prima di importare.

## Privacy / sicurezza (decisioni e piano)
- I CSV con i dati reali **non vanno mai nel repo**: aggiungere `*.csv` e i backup `.json` al `.gitignore`.
- Il sito su GitHub Pages è pubblico anche con repo privato: i dati devono stare solo nel database, mai nell'HTML.
- Firestore: salvare sotto `users/{uid}/...` con regole che permettono accesso solo al proprietario (`request.auth.uid == uid`), eventualmente ristretto alla mia email Google.
- Valutare **cifratura end-to-end** nel browser: AES-GCM (Web Crypto API) con chiave derivata da passphrase, cifrando prima di salvare e decifrando dopo la lettura, tutto dentro `store.get/set`. Se perdo la passphrase i dati non si recuperano. Possibile sblocco con Face ID/WebAuthn.

## Prossimi passi
1. ~~Repo con `index.html`, `.gitignore`, riepilogo~~ fatto (`.gitignore` esclude `*.csv`, `*.json`, `*.numbers`, `*.xlsx`).
2. ~~Firebase + login~~ fatto (email/password): manca la configurazione nella console (sezione sopra).
3. ~~Cifratura con passphrase~~ fatto. Possibile in futuro: sblocco con Face ID/WebAuthn.
4. Pubblicare su GitHub Pages, importare i CSV dall'app.
5. Poi rifinire: categorie modificabili/riordinabili, budget mensili, grafici per voce, import dal CSV della carta/banca, eventuale PWA (icona sulla home iPhone).
6. Limite da tenere a mente: ogni sezione è un documento Firestore (max 1 MiB): i movimenti ci stanno per molti anni (~5–6 mila movimenti), oltre andranno divisi per anno.

## Note sui dati del foglio (da verificare)
- Nel foglio Investimenti la riga Criptovalute mostra -29% con 4.898 € investiti e 8.181 € attuali: la % del foglio non coincide, nell'app è ricalcolata.
- Bondora risulta 4.148 € in Investimenti e 4.290 € in Situazione Finanziaria.
- Nov/Dic 2026 contengono già 76,76 € di ricorrenti inseriti nel foglio.
