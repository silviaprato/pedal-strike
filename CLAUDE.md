# Pedal Strike: note per Claude

Gioco arcade per allenarsi sui rulli (alternativa a Zwift), usato su iPad con il browser **Bluefy** (Safari e Chrome su iOS non supportano Web Bluetooth). Le utenti parlano italiano: interfaccia, messaggi e risposte in italiano.

## Struttura
- L'app è `index.html` (HTML + CSS + JS, senza build) più `three.min.js` (three.js r128, licenza MIT, copiato dal pacchetto npm `three@0.128.0` perché i CDN non sono raggiungibili dall'ambiente di sviluppo). three.js viene caricato solo quando si sceglie un percorso 3D.
- Pubblicazione: **GitHub Pages dal branch `main`, cartella root**. Un push su `main` va online in 1-2 minuti.
- Versione visibile sotto il titolo nella home: `<span style="opacity:.6">· vX.Y</span>`. **Incrementala a ogni modifica pubblicata**, così le utenti possono verificare di avere l'ultima versione (ricaricando la pagina in Bluefy).
- Messaggi di commit in italiano, con il prefisso della versione (es. `v3.1: ...`).

## Hardware e Bluetooth
- Rulli **Van Rysel D100** (Decathlon, wheel-on): FTMS (`0x1826`, Indoor Bike Data `0x2AD2`, Control Point `0x2AD9`) con ripiego su Cycling Power (`0x1818`).
  - **ERG** (campagna): opcode `0x05` con i watt target.
  - **SIM** (giro libero): opcode `0x11` con la pendenza × "Salite %" (default 50%).
  - Il D100 *stima* la potenza: la precisione dipende da gomma e taratura.
- Cardio: Heart Rate Service `0x180D` (fascia Garmin o orologio con "Trasmetti FC"). L'Apple Watch richiede un'app terza che ritrasmetta il battito.
- Comandi Zwift (Ride, Click v2, Play), sperimentali:
  - handshake `RideOn` sul servizio `0xFC82` / `00000001-19ca-...`; i messaggi noti sono `0x23` (bitmask), `0x07` e `0x37`;
  - i formati sconosciuti si assegnano "a campione" (id `zr<slot>:<hex>`);
  - **due slot separati**: Comando 1 (sparo A-D + intensità +) e Comando 2 (intensità − + pausa), con id `z1:`/`z2:`.
- La mappatura dei tasti è in `SET.map` / `SET.custom` (localStorage `ps_settings`), condivisa da tutti i profili. Le scelte dell'utente vanno conservate nelle modifiche future.
- Sono gestiti anche tastiera (`k:`) e gamepad (`p:`).

## Dati e profili
- Tutto resta in localStorage sul dispositivo:
  - `ps_profiles` contiene `{cur, list:[profilo...]}`;
  - `ps_settings` contiene audio e mappatura dei tasti.
- **Non rompere i dati salvati**: aggiungi i campi nuovi con un default in `newProf()` e scrivi una migrazione se cambia un formato.
- Profili multipli (di solito 2 persone sullo stesso iPad). Ogni profilo ha:
  - FTP, peso, età, sesso e altezza (facoltativa);
  - personaggio e colore della maglia;
  - XP, stelle, badge e storico;
  - monete, oggetti posseduti e indossati, mondo scelto;
  - missioni, record di GPM e sprint, fantasma del giro libero.
- Kcal, in ordine di priorità:
  1. quelle inserite a mano dall'orologio (`kcalW`);
  2. la stima dal battito con la formula di Keytel (`kcalHr`, serve età e sesso);
  3. i kJ dai watt.
- **Le uscite in DEMO** (rulli non collegati, potenza simulata) **non contano** per classifica, monete, missioni, record e serie. XP e livello invece sì.

## Funzioni di gioco (per orientarsi nel codice)
- **Campagna:** 3 mondi di allenamenti (`WORKOUTS`, segmenti `S(durata, %FTP, %FTP finale, testo)`), più il Ramp Test FTP. Stelle in base al tempo nei watt richiesti (±10%).
- **Gameplay:** droni con lettere A/B/C/D da colpire, boss negli intervalli ≥100% FTP, combo, Overdrive.
- **Fantasma:**
  - in campagna pedala ai watt esatti dell'allenamento;
  - nel giro libero rifà il miglior giro dell'utente.
- **Giro libero:** percorso fisso (`buildCourse(true)`, deterministico) con 2 GPM e 2 sprint cronometrati (`findSegs`).
- **Eventi a sorpresa:** attacco (solo nel giro libero), sciame di droni, drone d'oro.
- **Mondi** (`THEMES`, si sbloccano con il livello), **negozio** (`SHOP`: accessori, ruote, scie), **missioni settimanali** (`MPOOL`) e **serie di giorni**.
- **Personaggi** (`AVATARS`), disegnati **di spalle** con il canvas in `drawRider()`: niente volti né zip.
- **Musica** chiptune generata con WebAudio (`MUSIC`); il ritmo segue l'intensità.
- **Altre schermate:** classifica tra profili, storico, export TCX, pannello 📊 "Verifica dati".

## Percorsi realistici 3D (`ROUTES`, `R3`)
- Percorsi: `arcade` (vecchio motore 2D pseudo-3D con i mondi `THEMES`), `lanzarote`, `hawaii` (motore WebGL `R3`). La scelta si fa al Giro Libero (`pickRoute`) e nella scheda dell'allenamento; resta salvata in `SET.route`.
- Un percorso 3D è definito da dati: `km`, profilo altimetrico `prof` [[km, quota]], curve `bends` [[km, gradi, lunghezza km]], `zones` (nome, descrizione, lato del mare `sea` -1/1/0, distanza della costa, ampiezza colline, colori, densità oggetti per km), `cones` (vulcani), `turbines`, `far` (montagne lontane), `sky`. I profili sono *ispirati* ai luoghi reali, non presi da dati GPS.
- `R3.buildCourse` genera sia il `course` di gioco (segmenti da 5 m con pendenza) sia la geometria. Il terreno è fatto a strisce ai lati della strada (colonne `COLS` fino a 1250 m), il mare è un piano con shader a y=0, il cielo è una sfera con shader. Gli oggetti sono `InstancedMesh` raggruppati ogni 1,5 km per il frustum culling.
- Droni, fantasma, ciclista, spari ed effetti sono disegnati sul canvas 2D sopra il WebGL (`render3D` → `overlayTail`), proiettati con `R3.project`.
- Fantasmi (`profile.ghosts[route]`) e record dei tratti (`segRec`, con id `route_kom1` per i percorsi 3D) sono separati per percorso.
- Il tempo di gioco segue il tempo reale (`G.rdt`) anche se i fotogrammi sono pochi. C'è una riduzione automatica della risoluzione se l'iPad va lento.
- In Chromium headless il WebGL è emulato via software (circa 1 fotogramma/s): per provare la logica disattiva il disegno con `R3.load().then(()=>{R3.init();R3.ren.render=()=>{}})` prima di partire; per gli screenshot metti `G.paused=true` e aspetta qualche secondo.

## Come provare le modifiche
Non c'è hardware nell'ambiente di sviluppo: si prova in **modalità DEMO** con Playwright e Chromium headless.
1. Apri `file://.../index.html` e controlla che non ci siano `pageerror`.
2. Avvia un allenamento o il giro libero, usa i tasti A/B/C/D e chiama `finish()`.
3. Per simulare i rulli:
   - imposta `BLE.trainerOK=true`;
   - passa a `onIBD()` dei DataView FTMS finti;
   - in alternativa, sostituisci `readSensors`.
4. Fai uno screenshot per controllare la grafica.

Prima di pubblicare non lasciare dati di prova nel localStorage del browser di test.
