# PROGETTO — PescaRiva: da PWA ad App native per Google Play e App Store

**Versione:** 1.0
**Data:** 2026-07-17
**Autore:** Marino Vincenzo (Marinovinc)
**Stato:** Bozza per approvazione — nessuna implementazione ancora avviata
**Repository app:** `github.com/Marinovinc/PescaRiva` — live: `https://marinovinc.github.io/PescaRiva/`

---

## 1. Sommario esecutivo

**Obiettivo:** pubblicare PescaRiva come **vera app scaricabile** su **Google Play** (Android) e **App Store** (iPhone/iPad), partendo dall'attuale PWA senza riscrivere il codice.

**Approccio consigliato:** **Capacitor** (di Ionic) come guscio nativo unico per entrambe le piattaforme. Riusa gli **stessi file web** (`m.html`, `engine.js`, ecc.), aggiunge plugin nativi (geolocalizzazione, splash, status bar, filesystem offline) e produce un progetto **Android Studio** e uno **Xcode** che si caricano sugli store. Un solo codice, due app.

**Costi vivi (obbligatori):**
- Google Play Console: **25 USD una tantum** (account sviluppatore).
- Apple Developer Program: **99 USD/anno** (obbligatorio anche solo per pubblicare).
- Per compilare/firmare l'app iOS serve un **Mac** (o un servizio cloud-Mac). Android si compila anche su Windows.

**Tempo stimato:** 2-4 giorni di lavoro tecnico per avere i due pacchetti pronti; **1-2 settimane di calendario** includendo revisione Apple (1-3 giorni tipici) e Google (poche ore-giorni), più la preparazione di icone/screenshot/testi e privacy policy.

**Rischio principale (onesto):** Apple può **rifiutare** app che sono "solo un sito impacchettato" (linea guida **4.2 Minimum Functionality**). PescaRiva ha buone carte (mappe offline, batimetria satellitare, GPS, log catture locale) ma va presentata valorizzando le funzioni native e l'uso offline. Mitigazioni al §14.

---

## 2. Stato attuale dell'app (punto di partenza)

- **Tipo:** PWA client-only (nessun backend proprio), servita come file statici da GitHub Pages.
- **Stack:** HTML + CSS + JavaScript vanilla; **Leaflet** per la mappa; **geotiff.js** + **proj4** per la batimetria satellitare (SDB Sentinel-2).
- **Struttura:** `index.html` (versione desktop/classica), `m.html` (UI mobile stile Windy, dove i cellulari vengono reindirizzati), **`engine.js`** (motore condiviso, unica fonte), `sw.js` (service worker, cache offline v12), `manifest.webmanifest`, icone 192/512.
- **Funzioni:** profondità satellite (SDB 10 m) + isobate + canaloni, tocco mare → profondità/fondo, righello di lancio con profilo del fondale, prede probabili, GPS "sei qui", segna punto/cattura (salvato in locale), riconoscimento porti (OSM + zone manuali), salva zona offline, calibrazione profondità.
- **Dati esterni a runtime:** tile mappa (Esri/OSM), scene Sentinel-2 (AWS Open Data), EMODnet (batimetria/habitat), Overpass (porti). Tutti in HTTPS, senza credenziali.
- **Punti di forza per gli store:** funziona **offline** (mappe + ultima SDB in cache), usa **GPS**, salva dati **solo sul dispositivo** (nessuna raccolta dati → privacy label semplice), valore reale (cartografia del sottocosta non banale).
- **Debolezze da sanare per il packaging:** alcune librerie sono caricate da **CDN** (unpkg/jsdelivr) → per un'app nativa vanno **incluse in locale** (offline vero e nessuna dipendenza esterna in avvio); mancano **splash screen** e set completo di icone per gli store; manca una **privacy policy** pubblica (obbligatoria).

---

## 3. Obiettivo dettagliato

1. App **Android** pubblicata su Google Play, scaricabile e aggiornabile, con icona propria, splash, funzionamento offline e GPS.
2. App **iOS** pubblicata su App Store con gli stessi requisiti.
3. **Un solo codice sorgente** (gli attuali file web) per non dover mantenere versioni separate.
4. Le app devono restare **sincronizzate con la versione web**: una modifica al motore (`engine.js`) deve poter essere rilasciata anche nelle app con un semplice rebuild.

---

## 4. Approcci possibili (analisi)

### Opzione A — Capacitor (CONSIGLIATA)
Guscio nativo che carica i file web dell'app dentro una WebView di sistema, con accesso a plugin nativi.
- **Pro:** un solo progetto per **Android + iOS**; riusa gli stessi file web; plugin nativi ufficiali (Geolocation, SplashScreen, StatusBar, Filesystem, App); aggiornabile; rafforza il caso "non è solo un sito" per Apple; community ampia.
- **Contro:** su Android usa una WebView (non il TWA con motore Chrome pieno); richiede Node + Android Studio (+ Xcode su Mac per iOS).

### Opzione B — PWABuilder
Servizio che dalla PWA genera automaticamente un pacchetto **Android (TWA)** e uno **iOS (WKWebView)**.
- **Pro:** velocissimo, poca configurazione; l'Android TWA usa il motore Chrome (resa perfetta).
- **Contro:** meno controllo; il wrapper iOS è essenziale e **più esposto al rifiuto Apple 4.2**; personalizzazioni native limitate.

### Opzione C — TWA (Bubblewrap) per Android + Capacitor per iOS
- **Pro:** miglior resa/UX su Android (Chrome pieno, no limiti WebView); iOS robusto con Capacitor.
- **Contro:** **due toolchain** da mantenere; più complessità.

### Raccomandazione
**Opzione A (Capacitor)** per semplicità di manutenzione e coerenza tra le due app. Se in futuro la resa Android dovesse risultare limitante, si può passare l'Android a **TWA** (Opzione C) mantenendo iOS su Capacitor. Decisione finale al §17.

---

## 5. Architettura scelta (Capacitor)

- Si crea una **cartella app nativa** accanto al sito (es. un progetto Capacitor che punta ai file web come `webDir`).
- I file web (`m.html`/`index.html`/`engine.js`/assets) diventano il contenuto dell'app. Si valuta se l'app apra direttamente **`m.html`** (UI mobile) come pagina iniziale.
- **Vendoring librerie:** copiare in locale Leaflet (JS+CSS), geotiff.js, proj4 (oggi da CDN) così l'app parte **senza rete** e senza dipendenze esterne (requisito di robustezza e gradito agli store).
- **Plugin nativi previsti:**
  - *Geolocation* — GPS nativo (permessi gestiti dal sistema; migliore del solo `navigator.geolocation` in WebView).
  - *SplashScreen* — schermata di avvio.
  - *StatusBar* — colore barra di stato coerente col tema scuro.
  - *App / Filesystem* — gestione back button Android, eventuale salvataggio file (es. export catture) in futuro.
- **Aggiornamenti:** modifica ai file web → `npx cap copy` → rebuild → nuova versione sugli store. (Facoltativo in futuro: aggiornamenti "live" dei contenuti web senza ripassare dallo store, valutando le regole Apple.)
- **Nessuna modifica distruttiva al sito web esistente:** la PWA continua a vivere su GitHub Pages; l'app è un layer aggiuntivo.

---

## 6. Prerequisiti

**Account e costi:**
- **Google Play Console** — 25 USD una tantum.
- **Apple Developer Program** — 99 USD/anno.

**Strumenti (Android, anche su Windows):**
- Node.js LTS, Java JDK, **Android Studio** (SDK, emulatore, firma).

**Strumenti (iOS, richiedono macOS):**
- **Xcode**, account Apple, certificati/provisioning. In assenza di Mac fisico: servizio **cloud-Mac** (es. runner CI macOS o Mac a noleggio) — da decidere.

**Altro:**
- Dominio/URL pubblico per la **privacy policy** (si può usare una pagina su GitHub Pages, es. `.../PescaRiva/privacy.html`).
- Per Android TWA (solo se si sceglie l'Opzione C): file **`assetlinks.json`** pubblicato sul sito per la verifica del dominio.

---

## 7. Requisiti tecnici comuni (prima del packaging)

- [ ] **Vendoring** delle librerie CDN in locale (Leaflet, geotiff, proj4).
- [ ] **Manifest** completo e coerente (nome, short_name, theme/background color, display standalone, orientation, scope) — già in buona parte presente.
- [ ] **Set icone completo** per app native (Android adaptive icon: foreground+background; iOS: set 1024 e derivati).
- [ ] **Splash screen** (immagine + colore) per Android e iOS.
- [ ] **Permesso geolocalizzazione** con stringhe d'uso obbligatorie (iOS `NSLocationWhenInUseUsageDescription`; Android permessi `ACCESS_FINE/COARSE_LOCATION`).
- [ ] **Gestione offline** verificata dentro il guscio nativo (cache tile + SDB).
- [ ] **Versioning**: versionCode/versionName (Android), build/version (iOS).
- [ ] **Privacy policy** pubblica e linkata.
- [ ] **Test su dispositivo reale** (Android e, se possibile, iPhone) prima dell'invio.

---

## 8. Google Play — requisiti e passi

- [ ] Creare l'app nella **Play Console**, scegliere nome e categoria (proposta: *Sport* o *Meteo/Navigazione*).
- [ ] **Application ID** univoco (proposta: `it.pescariva.app` — da confermare, immutabile dopo la pubblicazione).
- [ ] **Chiave di firma** (Play App Signing consigliato) — custodire la keystore.
- [ ] **Target API level** aggiornato ai requisiti Play del 2026.
- [ ] **Data safety form**: dichiarare che l'app **non raccoglie né condivide dati** (i dati restano sul dispositivo); la posizione è usata solo localmente.
- [ ] **Content rating** (questionario) — presumibilmente PEGI 3 / Everyone.
- [ ] **Asset store**: icona 512×512, **feature graphic 1024×500**, screenshot telefono (min 2), descrizione breve/lunga.
- [ ] **Privacy policy URL** obbligatorio.
- [ ] (Solo se TWA) pubblicare **`assetlinks.json`** e verificare il dominio.
- [ ] Caricare **AAB** (Android App Bundle) firmato, track *internal testing* → *closed* → *production*.

---

## 9. Apple App Store — requisiti e passi

- [ ] Iscrizione **Apple Developer Program** (99 USD/anno).
- [ ] **App Store Connect**: creare l'app, **Bundle ID** (proposta: `it.pescariva.app`).
- [ ] **Firma** (certificati, provisioning) via Xcode.
- [ ] **Info.plist**: `NSLocationWhenInUseUsageDescription` con testo chiaro (perché serve il GPS); tema/orientamento.
- [ ] **Privacy Nutrition Labels**: dichiarare *nessuna raccolta dati* (Data Not Collected) — la posizione non lascia il dispositivo.
- [ ] **Screenshot obbligatori** nelle taglie richieste (iPhone 6.7"/6.5" e, se si supporta iPad, relative taglie).
- [ ] **Icona 1024×1024** senza trasparenza/angoli.
- [ ] **Rischio linea guida 4.2**: evitare che sembri "solo un sito". Mitigazioni al §14.
- [ ] Invio alla **review**; rispondere a eventuali rilievi (Resolution Center).

---

## 10. Asset da produrre

- [ ] **Icona app** master 1024×1024 (già esiste `icon-512`; va rifinita e derivata).
- [ ] **Adaptive icon Android** (foreground trasparente + background).
- [ ] **Splash screen** (logo su fondo `#06121f`).
- [ ] **Feature graphic Google Play** 1024×500.
- [ ] **Screenshot** (mappa con SDB, canaloni, righello, prede, zona porto) per entrambi gli store nelle taglie richieste.
- [ ] **Testi store**: nome, sottotitolo, descrizione breve e lunga, parole chiave (es. pesca, surfcasting, batimetria, fondale, spinning, mare, Sabaudia, Tirreno).

---

## 11. Privacy policy (contenuti da pubblicare)

Punti chiave (favorevoli, perché l'app è privacy-friendly):
- L'app **non ha un server proprio** e **non raccoglie dati personali**.
- I punti/catture, le località e le calibrazioni sono salvati **solo sul dispositivo** (localStorage/cache), mai inviati.
- La **posizione GPS** è usata solo per mostrare "sei qui" sulla mappa e **non lascia il dispositivo**.
- L'app contatta servizi cartografici di terze parti (Esri, OpenStreetMap, EMODnet, Copernicus/Sentinel via AWS, Overpass) **solo** per scaricare mappe/dati: valgono le loro informative; nessun dato personale dell'utente viene loro trasmesso oltre alla normale richiesta di tessere/aree.
- Contatti del titolare.
- Da pubblicare a un URL stabile (es. pagina dedicata sul sito) e linkare in entrambi gli store.

---

## 12. Piano di lavoro a fasi

**Fase 0 — Decisioni (utente):** confermare approccio (Capacitor), disponibilità Mac, Bundle ID, nome/categoria, apertura account. (§17)

**Fase 1 — Preparazione web:** vendoring librerie in locale; icone/splash; privacy policy; verifica offline nel guscio. Test.

**Fase 2 — Android (Capacitor):** init progetto, plugin, build AAB firmato, test su dispositivo/emulatore.

**Fase 3 — Store Android:** listing, asset, data safety, upload su *internal testing*, poi produzione.

**Fase 4 — iOS (Capacitor, su Mac):** progetto Xcode, permessi, splash, build, test su iPhone.

**Fase 5 — Store iOS:** App Store Connect, privacy labels, screenshot, invio review, gestione rilievi.

**Fase 6 — Post-lancio:** monitoraggio crash/recensioni; procedura di aggiornamento (modifica web → rebuild → nuova versione).

Ogni fase avrà una **checklist di verifica** e un test su dispositivo reale prima di procedere.

---

## 13. Manutenzione e aggiornamenti

- Il **motore** resta unico (`engine.js`): una correzione si riflette su web e, dopo rebuild, sulle app.
- Ogni release store richiede **incremento di versione** e nuovo pacchetto firmato.
- Valutare in seguito un meccanismo di **aggiornamento contenuti live** (caricare i file web da remoto) per ridurre i passaggi dallo store — compatibilmente con le regole Apple.

---

## 14. Rischi e mitigazioni

| Rischio | Impatto | Mitigazione |
|---|---|---|
| Apple 4.2 "solo un sito" | Rifiuto iOS | Valorizzare GPS nativo, mappe offline, log catture locale, splash/icone curate; descrivere il valore d'uso in barca/spiaggia; usare plugin nativi Capacitor |
| Nessun Mac per build iOS | Blocca iOS | Cloud-Mac / CI macOS a noleggio |
| Dipendenze CDN in avvio | App fragile offline | Vendoring librerie in locale (Fase 1) |
| Costi ricorrenti Apple | 99 USD/anno | Da mettere a budget; Android resta pubblicabile a 25 USD una tantum |
| Limiti WebView Android | Resa/perf | Se necessario passare Android a TWA (Opzione C) |
| Permessi GPS non chiari | Rifiuto/UX | Stringhe d'uso esplicite; richiesta solo al bisogno |

---

## 15. Costi e tempi (stima onesta)

- **Costi obbligatori:** 25 USD (Google, una tantum) + 99 USD/anno (Apple). Eventuale cloud-Mac se manca un Mac.
- **Tempo tecnico:** ~2-4 giornate di lavoro effettivo per i due pacchetti pronti (Fasi 1-2-4), più preparazione asset/testi/privacy.
- **Tempo di calendario:** ~1-2 settimane includendo le review degli store.

---

## 16. Naming e identità (proposta, da confermare)

- **Nome app store:** *PescaRiva* (eventuale sottotitolo: "Fondale e spot sotto costa").
- **Bundle ID / Application ID:** `it.pescariva.app` (unico e immutabile dopo la pubblicazione).
- **Categoria:** Sport (alt.: Navigazione / Meteo).
- **Icona:** dal logo pesce esistente, rifinita.

---

## 17. Decisioni aperte (da confermare prima di iniziare)

1. **Approccio:** confermare **Capacitor** (consigliato) oppure preferire PWABuilder (più veloce, meno controllo) o TWA+Capacitor.
2. **Mac per iOS:** ho a disposizione un Mac? In caso contrario, ok a un servizio cloud-Mac?
3. **Account:** procediamo con l'apertura di Google Play Console (25 USD) e Apple Developer (99 USD/anno)?
4. **Bundle ID e nome:** va bene `it.pescariva.app` e "PescaRiva"? Categoria?
5. **iPad:** vogliamo supportare anche iPad (screenshot e test in più) o solo iPhone?
6. **Privacy policy:** ok a pubblicarla come pagina sul sito attuale?

---

## 18. Architettura server-less: app autonoma + accesso a codice

> Requisito: **l'app deve funzionare senza un server di supporto**. L'app è **autonoma**: scarica mappe/dati dai **portali pubblici** e **calcola la batimetria sul telefono**, con caching automatico per l'offline. Un server (minimo, serverless) serve **solo** per **validare il codice di accesso**. Nessun dato di mappa o batimetria passa dal nostro server.

### 18.1 Principio: offline-first, server-opzionale

- **Il motore dell'app (`engine.js`) e i dati dell'utente restano sul dispositivo.** Catture, spot, calibrazioni, zone porto: solo in locale.
- **I dati "live" (mappe, batimetria, porti) NON passano dal nostro server:** l'app li prende — quando online — direttamente dai servizi pubblici di terze parti (Esri, OpenStreetMap, EMODnet, Copernicus/Sentinel via AWS, Overpass). Quindi l'app **non dipende** da un nostro backend per le funzioni.
- **Il nostro server NON è mai nel percorso d'uso delle funzioni.** Viene contattato solo all'**attivazione** e a un **ricontrollo periodico** della licenza. Mappe e batimetria **non passano dal nostro server**: l'app scarica dai portali e calcola sul telefono.
- Conseguenza: se il nostro server è irraggiungibile, l'app **continua a funzionare** (entro il periodo di grazia della licenza e con i pacchi già scaricati).

### 18.2 Ruolo A — Controllo accesso con codice di attivazione

**Flusso utente**
1. L'utente installa l'app dagli store.
2. Al primo avvio compare una schermata che chiede il **codice di attivazione**.
3. L'app invia al server `{ codice, deviceId }` (deviceId = UUID casuale generato dall'app, **non** un identificativo hardware).
4. Il server valida il codice e restituisce un **token di licenza firmato** (con scadenza, es. 30-90 giorni, legato al deviceId).
5. L'app salva il token in **storage sicuro** e lo **verifica in locale** (firma) ad ogni avvio → funziona **offline**.
6. Prima della scadenza (o ogni N giorni) l'app **rinnova** il token in background; se offline, un **periodo di grazia** la tiene attiva; solo dopo grazia+scadenza richiede di riconnettersi.

**Perché è sicuro e offline**
- Il token è firmato con la **chiave privata** del server (crittografia asimmetrica, es. Ed25519/RS256). L'app contiene **solo la chiave pubblica**: può **verificare** il token senza rete e **non può falsificarlo**.
- Il token è **legato al deviceId** (limita la condivisione); un codice può consentire *N* dispositivi.
- Solo **HTTPS**; endpoint con **rate-limit**; codici **hashed** a riposo; nessuna password, nessun dato personale.
- Il deviceId è un **UUID casuale** salvato in Keychain/Keystore: rispetta le regole degli store (iOS vieta certi identificativi hardware) e la linea "nessun dato raccolto".

**Modello dati (tabella codici)**
- `code_hash`, `piano/plan`, `max_attivazioni`, `attivazioni[] (deviceId, data)`, `scadenza`, `revocato`.

**Amministrazione**
- Un modo semplice per **generare** e **revocare** codici: uno script CLI o una mini-pagina admin protetta. I codici possono avere scadenza e numero massimo di attivazioni.

### 18.3 Funzionamento autonomo sul dispositivo (mappe + batimetria on-device)

> **Niente pacchi pre-elaborati, niente download di mappe dal nostro server.** L'app è autonoma come l'attuale PWA.

**Comportamento**
- Quando l'utente **cambia mappa o zona di pesca**, l'app scarica **da sola e in silenzio** ciò che serve **direttamente dai portali pubblici**: tessere mappa (Esri/OSM), scene **Sentinel-2** (AWS Open Data), EMODnet, porti (Overpass).
- La **batimetria SDB viene calcolata sul telefono** (logica Stumpf già in `engine.js`, con geotiff.js + proj4), **non** pre-elaborata su un PC.
- Ciò che l'utente visita finisce in **cache** (service worker + Filesystem): quella zona torna disponibile **offline** senza operazioni manuali (caching automatico e silenzioso).

**Conseguenze**
- Il **nostro server non ospita né serve** mappe o batimetria: nessun manifest, nessun pacco, nessuna pipeline di pre-elaborazione.
- La distribuzione dell'**app** avviene dagli **store** (ed eventualmente un APK diretto dal sito, se scelto). Piccoli aggiornamenti di **contenuti/config** (es. elenco località di default) possono restare dentro l'app o essere un file statico opzionale.
- Resta valido e invariato il **controllo accesso con codice** (§18.2): è l'**unica** funzione di backend necessaria.

**Prestazioni / attenzioni (on-device)**
- Il calcolo SDB sul telefono è già collaudato nella PWA; su device di fascia media va verificata la fluidità (dimensione griglia, numero scene del composito).
- La **cache** va gestita con un **limite di spazio** e una politica di **sfratto** delle zone meno recenti, per non riempire la memoria del telefono.

### 18.4 Stack consigliato e costi

- **License function:** una funzione serverless (es. **Cloudflare Workers**, oppure Supabase Edge Function / Firebase Functions / piccola Lambda). Database codici su **KV/D1** (Cloudflare) o Postgres (Supabase).
- **Nessuno storage di mappe:** i dati arrivano dai portali pubblici e sono calcolati sul telefono. Un piccolo file **config** statico opzionale può stare su qualsiasi hosting/CDN o nel repo.
- **Costi:** i **piani gratuiti** coprono ampiamente la fase iniziale (la sola license function fa pochissime richieste); **nessun server da mantenere**. Consigliato: **Cloudflare Workers + KV** (o Supabase se preferisci SQL + pannello admin pronto).

### 18.5 Impatti su privacy e store

- Coerente con "**nessun dato personale raccolto**": si trasmettono solo un **codice** e un **UUID casuale**; niente password, niente PII.
- Da citare nella **privacy policy**: uso del deviceId casuale per la licenza e download dei pacchi.
- Nessun impatto negativo sulle **privacy label**: resta "Data Not Collected" (o al più "Identificatori" non collegati all'utente, da valutare).

### 18.6 Schema

```
                DISPOSITIVO (autonomo, offline-first)
   ┌───────────────────────────────────────────────┐
   │  App + engine.js                               │
   │  scarica da solo dai portali (silenzioso)      │
   │  CALCOLA la batimetria SDB sul telefono        │
   │  cache automatica delle zone visitate          │
   │  token licenza (verificato IN LOCALE)          │
   │  dati utente: catture/spot (solo locale)       │
   └───────┬───────────────────────────┬────────────┘
           │ occasionale               │ live, quando online
           ▼                           ▼
   [ License Function ]         [ PORTALI PUBBLICI ]
    valida codice                Esri · OSM · EMODnet
    firma token                  Copernicus/Sentinel · Overpass
           │                     (NON il nostro server)
           ▼
    [ DB codici (KV/SQL) ]
```

### 18.7 Checklist di realizzazione

- [ ] Definire formato **token** (JWT firmato Ed25519/RS256) e scadenza + periodo di grazia.
- [ ] Generare coppia di chiavi; **chiave pubblica** dentro l'app, **privata** solo sul server.
- [ ] Implementare la **license function** (valida codice, firma token, rilascia URL firmati).
- [ ] Predisporre **DB codici** + strumento genera/revoca.
- [ ] Schermata **attivazione** nell'app + storage sicuro del token + verifica locale + ricontrollo/grazia.
- [ ] Verificare il **caching automatico** delle zone visitate nel guscio nativo (service worker + Filesystem) con **limite di spazio** e sfratto.
- [ ] Verificare le **prestazioni del calcolo SDB on-device** su telefoni di fascia media.
- [ ] (Opzionale) piccolo file **config** statico per contenuti aggiornabili senza passare dagli store.
- [ ] Aggiornare **privacy policy** (deviceId per la licenza).

---

## 19. Riferimenti

- Repo e sito: `github.com/Marinovinc/PescaRiva` — `https://marinovinc.github.io/PescaRiva/`
- Capacitor (guscio nativo cross-platform), PWABuilder (packaging PWA→store), Bubblewrap/TWA (Android).
- Requisiti store: Google Play Console (data safety, target API, asset), Apple App Store Connect (privacy labels, linee guida review, screenshot).

---

*Documento di progetto — nessuna modifica al codice è stata effettuata. Alla conferma delle decisioni al §17 si parte dalla Fase 1.*
