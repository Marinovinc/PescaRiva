# PROGETTO — PescaRiva: da PWA ad App native per Google Play e App Store

**Versione:** 1.0
**Data:** 2026-07-17
**Autore:** Marino Vincenzo (Marinovinc)
**Stato:** Bozza per approvazione — nessuna implementazione ancora avviata
**Repository app:** `github.com/Marinovinc/PescaRiva` — live: `https://marinovinc.github.io/PescaRiva/`

---

## 1. Sommario esecutivo

**Obiettivo:** pubblicare PescaRiva come **vera app scaricabile** su **Google Play** (Android) e **App Store** (iPhone/iPad), partendo dall'attuale PWA senza riscrivere il codice.

> **⭐ STRATEGIA DI LANCIO (aggiornata):** al lancio **NON** si passa dagli store. Si vende **diretto** (prezzo basso, **niente commissioni 15-30%**) tramite la **PWA installabile** (funziona su iOS e Android) + **codici licenza** venduti sul proprio sito/Facebook. Gli store (Capacitor, §§4-9) restano un'**opzione successiva** per la diffusione di massa. Dettaglio operativo: **vedi §25**.

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

> **Stato: bozza PRONTA** → file `privacy.html` (in repo, pubblicabile a `https://marinovinc.github.io/PescaRiva/privacy.html`). Include: sintesi, dati solo-su-dispositivo, GPS locale, attivazione/licenza (codice + UUID casuale), servizi terze parti, cosa NON facciamo, permessi, minori, diritti GDPR, contatti, riepilogo in inglese. **Da completare** i campi tra `[…]`: titolare, indirizzo, email di contatto. La versione inglese completa seguirà con la localizzazione.

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

> **Prototipo funzionante già sviluppato:** `access.html` (in repo → `https://marinovinc.github.io/PescaRiva/access.html`). Contiene le 3 viste navigabili (acquisto / attivazione codice / sbloccato), il **riscatto codice** client-side con abilitazione salvata in `localStorage` (`pr_license`, scadenza 6 mesi) e la simulazione di sblocco. **Da sostituire** con: chiamata reale alla *license function* (validazione + token firmato) e **IAP** dello store per l'acquisto. Il **gate** che mostra questa schermata all'avvio dell'app **non è ancora attivato** (per non bloccare l'app live): si aggancia quando il backend è pronto.

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

## 19. Monetizzazione e prezzo dei codici

### 19.1 Modello scelto

- **Licenza stagionale a 6 mesi** (fascia economica), con incasso **ibrido**:
  - **Acquisto in-app (IAP)** tramite i pagamenti dello store per l'utente normale → **conforme** ad Apple e Google.
  - **Codici del nostro server** (validi 6 mesi) per **regali, promo, closed beta e vendita diretta** → fuori dagli store, **nessuna commissione**.
- L'app è **gratuita da scaricare**; l'accesso alle funzioni si sblocca con l'acquisto **oppure** con un codice.

### 19.2 Prezzo (offerta di lancio + abbonamento + funzioni premium)

**Offerta di lancio (introduttiva)**
- **Primi 6 mesi a €3,99**, poi il servizio prosegue alla tariffa di abbonamento scelta. Sugli store si realizza come **offerta introduttiva** sull'abbonamento auto-rinnovabile.

**Abbonamento a regime (auto-rinnovabile), due durate**
- **Mensile — €1,00/mese**
- **Annuale — €10,00/anno** (invece di €12 → ~2 mesi gratis, incentivo all'annuale)

**Funzioni premium (crescono nel tempo)**
- L'abbonamento base sblocca l'app. **Alcune funzioni** (attuali o nuove) possono essere **premium**, a pagamento aggiuntivo: modello a **livelli** (Base / Premium) oppure **add-on** à la carte.
- Politica di prodotto: **le nuove funzioni si aggiungono a pagamento**, così il valore percepito (e il ricavo) cresce nel tempo senza svendere l'esistente.

**Note**
- Prezzi *psicologici*, tarabili dopo il lancio (promo stagionali).
- I **codici** del nostro server (§18.2) possono concedere accesso **Base** o **Premium**, per regali/promo/beta/vendita diretta.

### 19.3 Come si incassa (ibrido) — dettaglio

**Canale A — Acquisto in-app (utente normale)**
- Prodotto IAP **abbonamento auto-rinnovabile a 6 mesi** (consigliato: si rinnova da solo, l'utente può disdire; ricavo ricorrente). Alternativa: **non-rinnovabile** (l'utente ri-acquista a mano; più semplice ma più attrito).
- Lo store gestisce pagamento, IVA e rinnovi. L'app riceve la **ricevuta/entitlement** e sblocca.

**Canale B — Codici (regali / promo / beta / diretta)**
- Il nostro server (license function, §18.2) genera codici a 6 mesi. L'utente li **riscatta** nell'app (campo "Hai un codice?").
- Venduti su canali **esterni** (tuo sito con Stripe/PayPal, o consegnati a mano): nessuna commissione store, ma **gestisci tu IVA/fattura**.

### 19.4 Come l'app decide se è sbloccata (entitlement da 2 fonti)

- L'app considera l'accesso valido se **almeno una** di queste è attiva:
  1. **Ricevuta IAP** dello store (abbonamento in corso), verificata via StoreKit / Play Billing;
  2. **Token licenza** da codice riscattato (firmato dal nostro server, verificato in locale — §18.2).
- **Offline:** entrambe le fonti sono in **cache** (la ricevuta dallo store, il token in locale) con **periodo di grazia**; l'app resta usabile senza rete.
- Alla scadenza di entrambe → schermata di rinnovo/riscatto.

### 19.5 Conformità agli store (importante)

- L'**acquisto** avviene **solo** con l'IAP dello store → nessun problema con la regola Apple **3.1.1**.
- Il **campo "riscatta codice"** è **ammesso**, ma in-app **non si vende né si pubblicizza** l'acquisto del codice esterno (niente link/prezzi verso il sito): questo tiene l'app conforme. La vendita dei codici avviene fuori dall'app.
- In alternativa/aggiunta si possono usare i sistemi nativi: **Apple Offer Codes** e **Google Play promo codes** (per sconti/promo gestiti dagli store).
- **Small Business Program**: iscriversi su **entrambi** gli store → commissione **15%** (anziché 30%) finché il fatturato è sotto la soglia (≈ 1 M$/anno).

### 19.6 Economia (stima onesta, indicativa)

Ipotesi: prezzo **€3,99** IVA inclusa (IT 22%), **Small Business Program (15%)**.
- Netto ex-IVA ≈ €3,27; **proventi ≈ €2,78** per vendita IAP (con 30% sarebbero ≈ €2,29).
- Vendita **diretta** con Stripe: ≈ €3,99 − commissioni (~1,5% + €0,25) − IVA da versare ≈ **€2,9–3,0** netti, ma con gestione fiscale a tuo carico.
- **Break-even costo Apple** (99 $/anno ≈ €90): servono ~**33-36 attivazioni** (prezzo lancio) via IAP con SBP. Google è una tantum (25 $), già coperto.
- **A regime (dopo i 6 mesi intro):** €10/anno → netto ≈ **€6,9** con SBP; €1/mese → ≈ **€0,70 netti/mese** ricorrenti. L'**abbonamento ricorrente** rende il modello sostenibile e crescente nel tempo; il prezzo di lancio €3,99 è soprattutto **acquisizione clienti**.
- Conclusione: bastano poche decine di attivazioni per coprire i costi; l'abbonamento e le funzioni premium fanno il margine.

### 19.7 Checklist monetizzazione

- [ ] Configurare il prodotto IAP **abbonamento 6 mesi** su App Store Connect e Google Play (prezzo €3,99).
- [ ] *(Opzionale)* SKU **12 mesi €5,99**.
- [ ] Iscrizione **Small Business Program** su entrambi gli store (commissione 15%).
- [ ] Integrare **StoreKit / Play Billing** (o un plugin Capacitor per gli acquisti) + verifica ricevuta.
- [ ] Logica **entitlement a 2 fonti** (IAP OR codice) con cache offline e grazia.
- [ ] Schermata **paywall**: "Sblocca 6 mesi €3,99" + "Hai un codice?" (senza vendere il codice in-app).
- [ ] Canale di **vendita diretta codici** (sito + Stripe) con gestione IVA/fattura.
- [ ] *(Opzionale)* Apple Offer Codes / Google promo codes per le promozioni.

> **Mockup grafici pronti:** `MOCKUP_REGISTRAZIONE_ACQUISTO_PESCARIVA_20260717.html` — 3 schermate (1. acquisto/paywall, 2. attivazione con codice, 3. sbloccato) nello stile dell'app, con flusso e note di conformità store. Testi/valori da rifinire.

---

## 20. Suite di app (brand ed espansione)

> Questa app è la **prima di una suite** di app per la pesca. Vantaggio enorme: il **motore condiviso** (`engine.js`) e l'infrastruttura (Capacitor, license function, sistema codici, account store) sono **gli stessi** per tutte → si riusa il grosso del lavoro.

### 20.1 Visione

- **App 1 (questa) — Mare / sotto costa:** PescaRiva (pesca da riva, batimetria del sottocosta).
- **App 2 — Acque interne:** laghi e fiumi.
- **App 3+ — Discipline in barca:** drifting, **traina d'altura**, e altre tecniche.
- In prospettiva: una app per **disciplina/ambiente**, tutte con lo stesso stile e la stessa qualità.

### 20.2 Brand e naming — DECISO: FishCast

- **Brand ombrello (suite): FishCast** (internazionale: "fishing" + "cast"). **Sostituisce** *PescaRiva*.
- **Nomi app:**
  - Mare / sotto costa → **FishCast** (o **FishCast Coast**) — questa app.
  - Laghi/fiumi → **FishCast Lakes**.
  - Barca (drifting, traina d'altura) → **FishCast Offshore**.
- Icone e UI coerenti (famiglia FishCast), colore/tema per ambiente.
- **Da verificare prima di depositare/pubblicare:** disponibilità **marchio**, **dominio** (fishcast.app / .com), **handle social**, **nome sugli store**.
- **Rinomina tecnica** (repo, URL Pages, stringhe in-app, manifest, service worker, documenti): passo dedicato — cambia l'URL live in `marinovinc.github.io/FishCast`.

### 20.3 Riuso tecnico tra le app

- **Motore comune** `engine.js` (mappa, GPS, righello, catture, offline, licenza) → libreria condivisa.
- Ogni app aggiunge i **dati e le funzioni specifiche** dell'ambiente (es. batimetria mare vs profili di lago/fiume; specie e tecniche diverse).
- **Un unico account** sviluppatore Google/Apple per tutta la suite; **una sola** license function/DB codici (multi-app).

### 20.4 Monetizzazione della suite

- Ogni app ha il suo abbonamento (§19).
- In futuro: **bundle suite** (accesso a più app a prezzo scontato) e/o **codici multi-app**.
- Le **funzioni premium** possono essere trasversali (es. sync catture tra le app della suite).

---

## 21. Marketing (sito web + pagina Facebook)

### 21.1 Sito web / landing

- **Scopo:** presentare la suite e le singole app, con pulsanti verso **Google Play** e **App Store**, la **guida**, la **privacy policy** e la **vendita diretta dei codici** (canale esterno, non pubblicizzato in-app).
- **Contenuti:** cosa fa l'app, screenshot/video, funzioni, prezzi, FAQ, blog/novità.
- **Hosting:** una pagina dedicata (es. su GitHub Pages come il resto, o un **dominio** proprio tipo `pescariva.it` / un dominio della suite). SEO su parole chiave (pesca, surfcasting, batimetria, spinning, mare, laghi, traina…).
- **Coerenza store:** la vendita dei codici vive **qui** (sito/FB), **non** dentro l'app.

### 21.2 Pagina Facebook

- **Pagina brand** della suite per: annunci di lancio, aggiornamenti, promo stagionali, community (catture, spot, consigli), assistenza informale.
- Collegamento reciproco sito ↔ Facebook ↔ schede store.
- Possibile gruppo/community per fidelizzare i pescatori e raccogliere feedback per nuove funzioni.

### 21.3 Materiali comuni

- Kit grafico coerente per tutte le app (logo suite + logo per app), template screenshot store, brevi **video demo**.

### 21.4 Struttura del sito (mappa pagine)

- **Home** (suite) · **Pagina per app** · **Download** (link store) · **Guida** · **Privacy** · **Acquista codice** (Stripe) · **Blog/News** · **FAQ** · **Contatti**.
- **Flusso vendita codice:** checkout Stripe → codice via email → **riscatto nell'app**. (La vendita vive sul sito, non in-app.)
- Multilingua **IT/EN**, analytics **privacy-friendly**, collegamento a Facebook. Hosting statico (GitHub Pages o dominio proprio).

---

## 22. Multilingua (autoconfigurazione della lingua)

> Requisito: le app devono **auto-configurarsi** in base alla lingua del dispositivo.

### 22.1 Come funziona

- All'avvio l'app **rileva la lingua del dispositivo** e imposta l'interfaccia di conseguenza, con **fallback** (es. Inglese) se la lingua non è disponibile.
- **Selettore manuale** opzionale (per chi vuole forzare una lingua).

### 22.2 Struttura tecnica (i18n)

- Estrarre le stringhe dell'interfaccia (oggi **hardcoded in italiano** in `engine.js`/`m.html`) in **dizionari** per lingua (file JSON: `it`, `en`, …) e usare una funzione `t('chiave')`.
- **Lingue iniziali proposte (da confermare):** **IT** + **EN** (base), poi **DE**, **FR**, **ES** (Mediterraneo + turismo). Ampliabili.
- **Dati localizzati specifici della pesca:** i **nomi delle specie** e delle tecniche/esche cambiano per lingua/paese → vanno tradotti nei dizionari (importante per un'app di pesca).
- **Numeri/unità:** metri sempre, ma formattazione locale (virgola/punto) coerente con la lingua.

### 22.3 Store e marketing multilingua

- **Schede store** (titolo, descrizione, screenshot) tradotte nelle lingue supportate.
- Sito web e materiali marketing almeno in **IT/EN**.

### 22.4 Lavoro da pianificare

- L'app attuale è **tutta in italiano**: la prima localizzazione richiede di **estrarre le stringhe** in un dizionario e tradurle. È un lavoro reale ma una tantum, poi riusato da tutta la suite.

---

## 23. Roadmap e MVP

### 23.1 MVP — cosa entra nella v1.0 (store)

- L'**app mare attuale** (tutte le funzioni odierne) impacchettata con **Capacitor**.
- **Paywall:** abbonamento IAP (lancio 6 mesi €3,99, poi €1/mese o €10/anno) + **riscatto codice**.
- **Multilingua base:** IT + EN, autoconfigurazione.
- **Librerie in locale** (vendoring), **icone**, **splash**, **privacy policy**, **schede store**.
- Rilascio **Android per primo**, poi **iOS**.
- *Fuori dall'MVP (dopo):* funzioni premium, altre lingue, gestione cache avanzata, app sorelle.

### 23.2 Incrementi post-lancio (proposta)

- **v1.1** — primo blocco **funzioni premium** + lingue **DE/FR/ES**.
- **v1.2** — gestione **cache/offline avanzata**, esportazione catture, statistiche.
- **v1.3** — **sync catture** (cloud opzionale) tra dispositivi.
- **v2.0** — **App 2: Laghi e fiumi**.
- **v2.x** — **App 3: Barca** (drifting, traina d'altura).
- **v3.0** — **bundle suite** + codici multi-app.

### 23.3 Criterio

- Rilasciare **presto** un MVP solido (mare), poi **iterare** con funzioni premium e nuove app **riusando il motore** condiviso.

---

## 24. Funzioni premium (dettaglio)

### 24.1 Base (incluse nell'abbonamento)

Mappa satellitare/OSM, **profondità SDB + isobate**, **canaloni**, tocco profondità/fondo, **GPS**, **righello + profilo**, **prede probabili**, **catture/spot** locali, **riconoscimento porti**, **uso offline** (cache automatica), calibrazione base.

### 24.2 Premium (a pagamento aggiuntivo, nel tempo)

Candidate (da confermare e prioritizzare):
- **Meteo-marino integrato** (vento, onde, marea), stile Windy.
- **Solunar** / finestre di attività dei pesci.
- **Layer avanzati:** correnti, SST, **clorofilla** (integrazione con le altre app/progetti della suite).
- **Esportazione catture** GPX/CSV + **statistiche** avanzate.
- **Sync cloud** catture/spot tra dispositivi e tra app della suite.
- **Cache offline estesa** / più zone.
- **Composito multi-data avanzato**, calibrazione avanzata.

> Principio: il Premium deve essere **valore aggiunto reale**, senza tagliare le funzioni base attuali.

### 24.3 Come si vende il Premium

- **Tier superiore** dell'abbonamento, oppure **add-on IAP** (sblocco non-consumabile).
- I **codici** possono concedere accesso **Base** o **Premium**.

---

## 25. Vendita diretta senza store (PWA + codici) — strategia di lancio

> Obiettivo dell'utente: vendere **direttamente**, a **prezzi bassi**, **senza pagare le commissioni** di Apple/Google e senza passare dalle loro regole, almeno al lancio.

### 25.1 Il vincolo onesto (perché la via è la PWA)

- **iOS:** fuori dall'App Store **non** è possibile distribuire app native al pubblico generico (le eccezioni UE post-DMA — marketplace alternativi / web distribution — hanno requisiti onerosi: iscrizione, notarizzazione, soglie). Quindi, per non passare da Apple, la strada è la **PWA**.
- **Android:** è possibile distribuire l'**APK** direttamente (sideload dal proprio sito) **e/o** la PWA. Nessuna commissione.
- **Buona notizia:** PescaRiva è **già una PWA**. Su **iOS e Android** si "installa" dal browser (**Aggiungi a schermata Home**): icona propria, schermo intero, funziona **offline**. Zero store, zero commissioni.

### 25.2 Come funziona (flusso di vendita diretta)

1. L'utente arriva dal **sito** o da **Facebook**.
2. **Paga** direttamente (prezzo basso) → riceve un **codice licenza** (via email).
3. **Installa la PWA** (Aggiungi a Home) e **riscatta il codice** nella schermata `access.html` (già pronta) → sblocco, poi funziona **offline** con ricontrollo periodico.

### 25.3 Incasso diretto — SCELTA: Merchant of Record

> **Deciso: A) Merchant of Record.** Piattaforma consigliata: **Lemon Squeezy** (pensata per chiavi di licenza software, con **License API** validabile dal client; gestisce IVA e fatture; ~5% + commissioni di pagamento). Alternative equivalenti: **Paddle** (anch'esso MoR + licenze), **Gumroad** (più semplice, commissione più alta).

- **A) Merchant of Record (SCELTO):** vende **chiavi di licenza**, gestisce **IVA e fatture al posto tuo**, ~**5%**, con **API di licenza** per validare/attivare. **Vantaggio chiave: con il MoR NON serve un nostro server** per i codici a pagamento — la piattaforma È il backend di licenza (validazione + scadenza + limite dispositivi). La nostra *license function* (§18.2) resta **opzionale**, solo per codici **regalo/beta** emessi da noi.
- **B) Stripe / PayPal diretto** *(non scelto):* commissione più bassa (~**3% + €0,25**), ma IVA (OSS UE) e fatture a carico nostro.

### 25.4 Prezzo

- Potendo evitare le commissioni store, si può andare **molto bassi** (es. **€1-3**) tenendo ~**95-97%** (Stripe) o ~**95%** (MoR). L'utente paga poco, tu incassi quasi tutto.

### 25.5 Codici e validazione (con Merchant of Record)

- I **codici a pagamento** sono le **chiavi di licenza** generate da **Lemon Squeezy** (o Paddle). Il **riscatto** avviene nella PWA (`access.html`); la **validazione/attivazione** usa la **License API** del MoR — chiamabile **dal client** con la sola chiave (endpoint `validate` / `activate`), con **limite di attivazioni** per N dispositivi.
- I **codici regalo/beta** emessi da noi usano (opzionalmente) la nostra **license function** (§18.2).
- In entrambi i casi l'abilitazione è salvata **in locale** (con scadenza) e **periodo di grazia** → offline; ricontrollo periodico online.
- Confronto completo delle commissioni e convenienza: vedi il documento dedicato **CONFRONTO_METODI_PAGAMENTO_PESCARIVA_20260717.html/.md**.

### 25.6 Limiti onesti della PWA su iOS

- Nessun listing sullo store (la **scoperta** avviene via sito/Facebook/passaparola).
- Alcune limitazioni delle API web (ma su **iOS 16.4+** ci sono le notifiche web; installazione e offline funzionano bene).
- Per un pubblico di **nicchia** che acquista direttamente, è più che adeguato.

### 25.7 Dopo (opzionale): gli store per la diffusione

- Quando si vuole **diffusione di massa**, si può comunque pubblicare la versione **Capacitor** sugli store (§§4-9), **mantenendo** la vendita diretta per chi arriva dal sito. Le due cose **convivono**: chi compra diretto usa il codice, chi arriva dallo store usa l'IAP.

### 25.8 Checklist (vendita diretta)

- [ ] Scegliere il metodo d'incasso (MoR vs Stripe/PayPal) — §25.3.
- [ ] Impostare la **pagina di vendita** (sito) con il prodotto e il prezzo.
- [ ] Collegare **pagamento → generazione codice → email** (automatico o manuale).
- [ ] Collegare `access.html` alla **validazione reale** del codice (API pagamento o license function).
- [ ] Guida "**come installare la PWA**" (iOS: Aggiungi a Home da Safari; Android: Aggiungi a Home / installa).
- [ ] *(Opzionale)* build **APK** Android per download diretto dal sito.

---

## 26. Riferimenti

- Repo e sito: `github.com/Marinovinc/PescaRiva` — `https://marinovinc.github.io/PescaRiva/`
- Capacitor (guscio nativo cross-platform), PWABuilder (packaging PWA→store), Bubblewrap/TWA (Android).
- Requisiti store: Google Play Console (data safety, target API, asset), Apple App Store Connect (privacy labels, linee guida review, screenshot).

---

*Documento di progetto — nessuna modifica al codice è stata effettuata. Alla conferma delle decisioni al §17 si parte dalla Fase 1.*
