# Park&Speech — contesto progetto

App per l'analisi e l'esercizio della voce, sviluppata per una logopedista che
segue pazienti con **malattia di Parkinson** (disartria ipocinetica).
Autore: Sergio Moro. Non è un dispositivo medico diagnostico: è uno strumento di
monitoraggio ed esercizio.

## Due app, due repository

| App | Repository | Indirizzo | Destinatario |
|---|---|---|---|
| Clinica | `parkandspeech` | https://smacc8.github.io/parkandspeech/ | logopedista |
| Esercizi | `parkandspeechpat` | https://smacc8.github.io/parkandspeechpat/ | paziente, in autonomia |

Entrambe: **PWA vanilla**, un unico `index.html` autosufficiente (HTML+CSS+JS
inline, nessun framework, nessun build), più `sw.js`, `manifest.json`,
`icon-192.png`, `icon-512.png` nella radice del repo. Pubblicate con GitHub Pages
(branch `main`, root).

Le due app condividono lo stesso "motore" audio duplicato nei due file:
**ogni modifica al motore va riportata su entrambi.**

## Differenze fra le due

**Clinica**: menu con Test singolo · Test in serie · Cronometro · Storico ·
Dettato · QR paziente · Impostazioni e info. Storico su `localStorage`, export
CSV, invio via email, anagrafica paziente chiesta **al salvataggio** (non prima).

**Paziente**: solo 3 esercizi (Intensità massima, Forte e lungo, Cambio di tono)
+ Cronometro + Dettato + check distanza. **Niente storico, niente salvataggi,
niente impostazioni.** Esclusa "Normale e lungo" perché puramente valutativa.

## I test

1. **Intensità massima** — voce fortissima, anche liste di parole (contare 1-10).
   Registrazione **continua**.
2. **Forte e lungo** — /a/ forte tenuta in un fiato. MPT, intensità, tenuta.
3. **Normale e lungo (MPT)** — solo app clinica.
4. **Cambio di tono** — il paziente deve **mantenere** la voce dentro fasce di
   frequenza per un tempo dato. Fasce configurabili (fino a 5, con colore
   verde/azzurro/rosa), grafico alto con scala Y adattiva al min/max impostato.

Metriche: MPT, intensità media/picco, trend di intensità (regressione lineare
→ "calante" è il decay tipico del Parkinson), F0 e sua deviazione standard,
jitter/shimmer/HNR (stime frame-level, non ciclo-per-ciclo come Praat: valide
come trend, non come valori clinici assoluti).

## Vincoli tecnici — NON REGREDIRE SU QUESTI

Sono tutti nati da bug reali costati settimane. Prima di toccarli, capire perché ci sono.

1. **AGC off.** `getUserMedia` con `autoGainControl:false, echoCancellation:false,
   noiseSuppression:false`. Senza, qualsiasi misura di intensità è inutilizzabile.

2. **Rilascio del microfono (`releaseMic`).** Lo stream audio va **chiuso** prima
   di avviare il riconoscimento vocale: su Android il microfono non si condivide.
   Tenerlo aperto causava: dettatura che non parte, che funziona solo con la bocca
   attaccata al telefono, e che ripete le prime parole. È stato il bug più grave.

3. **Registrazione continua** su "Intensità massima" e "Cambio di tono"
   (`CUR_CONT`): non si ferma alle pause e prosegue **finché l'utente preme
   "Ferma ora"** (tetto di sicurezza 5 min). Serve per liste di parole e vocalizzi
   in serie. Non reintrodurre timer che la interrompono.

4. **Rilevamento del tono permissivo.** Soglie basse (rms 0.0025, clarity 0.30) e
   linea che "scavalca" i buchi brevi. Scelta deliberata: per l'esercizio conta la
   fluidità del tracciato più della precisione. L'analisi pass/fail delle fasce
   resta invece filtrata.

5. **Dettato Android-safe.** Su Android: `continuous=false` (lì è mal supportato e
   ri-emette i risultati), risultati **indicizzati** (`finals[i]=...`, una
   ri-emissione sovrascrive invece di duplicare), riavvio ritardato, e
   de-duplicazione per sovrapposizione che ignora punteggiatura e maiuscole.

6. **Intensità mostrata come "Livello 0–100"** se non c'è calibrazione: è
   derivata dal dBFS e non è confrontabile con i 65/75 dB attesi dalla
   logopedista. Con la **calibrazione SPL** attiva (vedi sotto) l'app mostra
   invece **dB SPL** ovunque. Nel CSV restano i dBFS reali.

7. **CSV per Excel italiano**: separatore `;`, virgola decimale, BOM UTF-8, CRLF.

8. **Service worker network-first**, con versione cache da incrementare ad ogni
   rilascio (`voce-vN` e `voce-paz-vN`), altrimenti i telefoni servono la versione
   vecchia. Se una PWA è già installata, spesso serve disinstallare + hard refresh.

## Impostazioni (solo app clinica)

Temi (5), visualizzazione dB/Livello, orientamento comandi (paziente / medico
capovolto), soglie di avvio voce normale e forte **legate fra loro** (la normale
non può superare la forte), gestore stanze (il nome stanza finisce nel CSV; utile
per tracciare anche il setup, es. "Studio 1 · VM10"), schema del Cambio di tono,
schema del Test in serie (attivazione, ordine, ripetizioni), email del medico,
cancella cronologia (doppia conferma).

## Preferenze dell'autore

Risposte brevi e diritte, franche anche quando critiche. Sergio **non è un
programmatore**: spiegare codice e informatica in modo semplice ma corretto.
Chiedere prima di scelte importanti; sulle cose di routine procedere.
Testa su Mac desktop, Android (telefono e tablet), laptop Linux Mint; va comunque
verificata la compatibilità iOS.

## Calibrazione SPL

Fatta (app paziente, 18/09/2026). Taratura **a due punti** con fonometro esterno:
si misura 3 s a voce normale e 3 s a voce forte, si digitano i dB letti sul
fonometro, l'app calcola la retta `SPL = a·dBFS + b` (la pendenza corregge anche
il guadagno del microfono, non solo l'offset) e la salva in `SET.cal`.

- Accesso **nascosto**: pressione lunga ~1,8 s sul logo → schermata `vCal`.
  Il gesto apre la calibrazione al **rilascio** del dito, non allo scadere del
  timer: `getUserMedia` vuole un gesto utente fresco.
- La media delle misure è una **media di potenza** sui frame sopra la soglia di
  voce, non una media di dB (che falserebbe il risultato).
- Controlli di plausibilità: i due punti devono distare ≥ 6 dBFS, il fonometro
  deve leggerli fra 30 e 130 dB e in ordine crescente, la pendenza deve stare
  fra 0,4 e 2,5. Fuori da lì la calibrazione viene rifiutata con il motivo.
- Con `SET.cal.on` l'app mostra **dB SPL** al posto del Livello 0–100 ovunque
  (live, risultati, riepilogo, referto). Senza, tutto torna come prima.
  **Nel CSV restano sempre i dBFS.**
- Vale per quel telefono a quella distanza: cambiando l'uno o l'altra va rifatta.
- **Da portare sull'app clinica** (`parkandspeech`): stessa cosa, ma lì la
  schermata può stare dentro Impostazioni invece che dietro la pressione lunga.

## Aperto / da fare

- Valutare se il Dettato regge sul campo; l'alternativa affidabile sarebbe una STT
  cloud (a pagamento) o un'app nativa con riconoscimento Android nativo.
- Eventuale guida fase-per-fase in tempo reale nel Cambio di tono (oggi la
  valutazione è a posteriori su tutta la registrazione).

## Storia dei feedback clinici (sintesi)

La logopedista usa l'app in ambulatorio. Ha chiesto e ottenuto: registrazione che
non si ferma alle pause, grafico del tono più facile da tracciare (confronto con
"Voice Tools"), rimozione di "Normale e lungo" dalla versione paziente, e il
Dettato (ispirato a Voice Notebook). Ha segnalato il problema dei permessi
microfono: il permesso va dato **al browser per quel sito**, non all'app Google —
ora le istruzioni sono dentro l'app.
