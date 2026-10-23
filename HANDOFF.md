# Bilancio — app finanze personali (riepilogo per riprendere il lavoro)

## Obiettivo
App per tracciare tutte le mie spese e finanze, stessa filosofia dell'"app del gommone" (Giornale di Bordo): **un unico file HTML**, mobile-first (iPhone), hosting su **GitHub Pages**, login **Google** con **Firebase** per sincronizzare i dati. Stile diverso dal gommone: grafite scuro + verde smeraldo (Manrope + JetBrains Mono) invece di blu notte/ottone.

## Stato attuale
- File: `index.html` (un solo file, nessuna dipendenza oltre a Google Fonts e, solo in modalità cloud, l'SDK Firebase compat 10.14.1 caricato da gstatic).
- **Nessun dato dentro il file**: i dati si caricano da CSV (ingranaggio in alto a destra → "Importa da CSV").
- **Due modalità**, decise da `FIREBASE_CONFIG` in cima allo script:
  - `apiKey` vuota → solo locale, `localStorage` (chiave `bilancio:v1`), come la prima bozza.
  - `apiKey` compilata → login Google + passphrase, dati su Firestore **cifrati end-to-end**.
- Il resto dell'app usa solo `store.get()` / `store.set()` (`localStore` o `cloud`).

## Cloud: come funziona
- Gate prima dell'app: **Accedi con Google** (popup, con fallback a redirect) → **Crea cassaforte** la prima volta / **Sblocca** le volte dopo.
- Cifratura: AES-GCM 256, chiave da passphrase con PBKDF2-SHA256 600.000 iterazioni, sale casuale. Ogni sezione è cifrata a parte con la sezione come dato associato. Su Firebase non arriva mai nulla in chiaro.
- Firestore:
  - `bilancio/{uid}`: `salt`, `iter`, `check` (la parola `bilancio-ok` cifrata, serve a verificare la passphrase)
  - `bilancio/{uid}/sez/{tx|conti|rate|inv|trading}`: `kid` (= sale attuale), `iv`, `ct`
- Salvataggio ottimistico: la UI cambia subito, l'invio parte dopo 400 ms con le sole sezioni modificate, in un batch; se fallisce riprova con attese crescenti (2 s → 60 s). Puntino in alto: verde sincronizzato, giallo salvataggio in corso, rosso errore. Quando l'app va in background invia subito; quando torna in primo piano ricarica le modifiche fatte da altri dispositivi. Cache offline di Firestore attiva (anche lì solo dati cifrati).
- "Ricorda su questo dispositivo": salva la chiave AES **non esportabile** in IndexedDB (mai la passphrase). "Blocca" la dimentica e chiede di nuovo la passphrase.
- **Cambia passphrase** (Impostazioni): nuovo sale, ricifra tutto in un'unica scrittura atomica. Le regole rifiutano le scritture con `kid` vecchio: un altro dispositivo con la passphrase vecchia viene bloccato e la chiede di nuovo.
- Primo accesso da un browser con dati locali e cloud vuoto: i dati locali vengono caricati nel cloud e poi tolti da `localStorage`.
- Se perdo la passphrase i dati non si recuperano: esportare ogni tanto il backup `.json` (che è in chiaro, da tenere fuori dal repo).

## Configurare Firebase (da fare una volta)
1. Console Firebase → progetto (nuovo, oppure quello del gommone: la collezione `bilancio` è separata).
2. Authentication → Metodo di accesso → attiva **Google**. Impostazioni → Domini autorizzati → aggiungi `<utente>.github.io`.
3. Firestore Database → crea (modalità produzione). Regole → incolla `firestore.rules` **dopo aver messo la tua email** al posto di `TUA_EMAIL@gmail.com`. Se il progetto è condiviso, aggiungi solo il blocco `bilancio` alle regole esistenti.
4. Impostazioni progetto → Le tue app → App web → copia `apiKey`, `authDomain`, `projectId`, `appId` dentro `FIREBASE_CONFIG` in `index.html`.
5. (Consigliato) Google Cloud Console → Credenziali → limita la chiave API ai referrer `https://<utente>.github.io/*`.
6. GitHub → Settings → Pages → deploy dal branch principale. Apri il sito, accedi, crea la passphrase, importa i CSV.

## Schermate (barra in basso)
1. **Panoramica**: saldo mese/anno, liquidità e patrimonio, ciambella uscite per categoria, barre entrate vs uscite, top voci.
2. **Movimenti**: lista con filtri (tipo, mese/anno, ricerca), pulsante + per aggiungere, tocca per modificare/eliminare.
3. **Mensile**: tabella categorie × mesi (replica la tabella "Contabilità" del mio file Numbers), uscite/entrate.
4. **Patrimonio**: conti divisi in Banche / Cash / Investimenti, saldo modificabile.
5. **Altro**: Rate (con "Rata pagata" che scala le residue), Investimenti (investito, attuale, rendimento % ricalcolato), Trading (P&L giornaliero per mese).

## Modello dati (stato `S`)
- `tx`: `{id, d:'YYYY-MM-DD', t:'u'|'e', cat, sub, amt, note}`: uscite con macro-categoria + sottovoce (Mezzi › Gommone, Casa › Spesa, Lavoro › Trading…); entrate per fonte + voce (Porto › CostaMed, Locali › Morgan…)
- `conti`: `{id, g:'Banche'|'Cash'|'Investimenti', n, v}`
- `rate`: `{id, tipo, prod, rata, res}`
- `inv`: `{n, lvl:0|1|2, inv, eff}` (i totali usano solo `lvl 0`)
- `trading`: `{id, d, v}`

## Come lavoro (dal mio file Numbers 2026)
Fogli: CONTABILITA (uscite per macro-categoria/sottovoce × 12 mesi, entrate, situazione finanziaria con Liquidità e Patrimonio), Rate (PayPal), RIEPILOGO INVESTIMENTI, P&L (trading giornaliero, vuoto), SPESE CARTA DI CREDITO (vuoto). Ho 239 movimenti mensili storici da importare: le celle mensili sono state convertite in movimenti datati il giorno 1 di ogni mese.

## Import CSV (già implementato)
Separatore `;` o `,`, decimali con virgola, date `YYYY-MM-DD` o `DD/MM/YYYY`.
- Movimenti: `data;tipo;categoria;voce;importo;nota` (tipo = uscita|entrata)
- Conti: `gruppo;nome;saldo`
- Rate: `metodo;prodotto;rata;residue`
- Investimenti: `voce;livello;investito;attuale`
- Trading: `data;importo`
Opzione "Sostituisci" per cancellare i dati di quel tipo prima di importare.

## Privacy / sicurezza (decisioni e piano)
- I CSV con i dati reali **non vanno mai nel repo**: aggiungere `*.csv` e i backup `.json` al `.gitignore`.
- Il sito su GitHub Pages è pubblico anche con repo privato: i dati devono stare solo nel database, mai nell'HTML.
- Firestore: salvare sotto `users/{uid}/...` con regole che permettono accesso solo al proprietario (`request.auth.uid == uid`), eventualmente ristretto alla mia email Google.
- Valutare **cifratura end-to-end** nel browser: AES-GCM (Web Crypto API) con chiave derivata da passphrase, cifrando prima di salvare e decifrando dopo la lettura, tutto dentro `store.get/set`. Se perdo la passphrase i dati non si recuperano. Possibile sblocco con Face ID/WebAuthn.

## Prossimi passi
1. ~~Repo con `index.html`, `.gitignore`, riepilogo~~ fatto (`.gitignore` esclude `*.csv`, `*.json`, `*.numbers`, `*.xlsx`).
2. ~~Firebase + login Google~~ fatto nel codice: manca la configurazione (sezione sopra).
3. ~~Cifratura con passphrase~~ fatto. Possibile in futuro: sblocco con Face ID/WebAuthn.
4. Pubblicare su GitHub Pages, importare i CSV dall'app.
5. Poi rifinire: categorie modificabili/riordinabili, budget mensili, grafici per voce, import dal CSV della carta/banca, eventuale PWA (icona sulla home iPhone).
6. Limite da tenere a mente: ogni sezione è un documento Firestore (max 1 MiB): i movimenti ci stanno per molti anni (~5–6 mila movimenti), oltre andranno divisi per anno.

## Note sui dati del foglio (da verificare)
- Nel foglio Investimenti la riga Criptovalute mostra -29% con 4.898 € investiti e 8.181 € attuali: la % del foglio non coincide, nell'app è ricalcolata.
- Bondora risulta 4.148 € in Investimenti e 4.290 € in Situazione Finanziaria.
- Nov/Dic 2026 contengono già 76,76 € di ricorrenti inseriti nel foglio.
