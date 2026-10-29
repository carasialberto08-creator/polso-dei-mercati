# Polso dei Mercati – Coach

Web app in un unico file (`index.html`) per l'investitore privato italiano. Ti dice com'è il mercato oggi, ti propone azioni da studiare con criteri trasparenti e ti insegna i concetti per decidere da solo.

> **Strumento educativo, non consulenza finanziaria.** I segnali sono probabilistici: l'app dice "i dati indicano", mai "compra".

## Come si usa

1. Apri `index.html` nel browser, con doppio clic o da GitHub Pages. Non serve un server.
2. Senza chiavi vedi **dati demo**, segnalati con l'etichetta gialla "Demo".
3. Per i dati reali tocca ⚙️ **Impostazioni**, incolla le chiavi gratuite (istruzioni sotto) e salva.

---

## 1. Architettura

### Sezioni (barra in basso)

| Sezione | Contenuto |
|---|---|
| **Mercato** | Semaforo S&P 500 (punteggio 0–100 con il dettaglio del calcolo), scheda S&P 500, tessere degli indici (Nasdaq, FTSE MIB, Euro Stoxx 50, DAX), EUR/USD, oro, petrolio, VIX o volatilità, BTP e Treasury a 10 anni, movimenti dei titoli seguiti, notizie con sentiment, calendario a 30 giorni (BCE, Fed, inflazione, occupazione, trimestrali) |
| **Azioni** | Screener (settore, mercato IT/EU/USA, capitalizzazione, dividendo) su 32 titoli. Scheda titolo con fondamentali confrontati con il settore, tecnici (trend, RSI, medie 50/200, supporti e resistenze, volatilità), notizie degli ultimi 7 giorni, **Punteggio qualità** 0–100, **Momento di ingresso** (Caro / Nella media / Interessante), **Rischi principali**, "In sintesi" e "Spiegami questa decisione" |
| **Watchlist** | Titoli preferiti salvati in locale. Quando cambia la valutazione compare un avviso (pallino rosso sulla scheda) |
| **Accademia** | Percorso Base → Intermedio → Avanzato (9 lezioni da 3 minuti con quiz, progressi salvati), "Spiegami questa decisione", calcolatore PAC e interesse composto, glossario |
| **Simulatore** | Portafoglio virtuale da 10.000 €, storico delle operazioni, confronto con un ETF azionario globale (VT) |

Ogni indicatore ha un'icona **?** con una spiegazione semplice e un esempio pratico.

### Flusso dei dati

```
 Interfaccia (render* per ogni sezione)
        │
 Calcoli puri: sma, rsi, realizedVol, technicals, semaforo,
               qualityScore, entryMoment, risks, sentimentOf
        │
 Accesso ai dati: getQuote, getHistory, getPriceData, getMetrics,
                  getMarketNews, getCompanyNews, getEarnings, getTreasury
        │  ① cache localStorage (prefisso pdm_c_), 15 min per prezzi e notizie,
        │     6–12 h per fondamentali e storico
        │  ② coda per fornitore (limiter): massimo di chiamate parallele + pausa
        │  ③ fetchJSON: timeout 12 s, messaggi chiari per 401/403/429/CORS
        │  ④ se fallisce: dato in cache scaduto, altrimenti dati DEMO (etichettati)
        ▼
 Finnhub · Financial Modeling Prep · Alpha Vantage
 (dentro claude.ai: connettori Bigdata.com, Zacks, Twelve Data, Alpha Vantage)
```

- **Dati demo**: serie generate con un seme fisso per simbolo, uguali a ogni apertura e sempre etichettate "Demo".
- **Errori**: un riquadro rosso elenca cosa non è arrivato, raggruppando gli errori uguali ("Finnhub (XLY, XLP, …): limite di chiamate raggiunto").
- **Archiviazione locale** (sempre dentro try/catch): impostazioni e chiavi, watchlist, valutazioni precedenti, progressi delle lezioni, portafoglio del simulatore, cache.

### Il semaforo (sempre spiegato nell'app)

| Componente | Punti | Regola |
|---|---|---|
| Trend | 35 | Prezzo sopra la media a 200 giorni: +20. Media a 50 sopra la 200: +15 |
| Volatilità | 25 | VIX (o volatilità realizzata a 20 giorni) <15: 25 · <20: 19 · <25: 13 · <35: 6 · oltre: 0 |
| Ampiezza | 20 | Quota degli 11 ETF settoriali USA in rialzo × 20 |
| Sentiment notizie | 20 | (positive − negative) / totale, riportato su 0–20 |

Da 65 in su è **Favorevole**, da 40 a 64 **Neutro**, sotto 40 **Rischio alto**. Se una componente manca, il totale viene riportato su 100 e l'app lo dice.

### Profilo di rischio

In Impostazioni scegli Prudente, Equilibrato o Dinamico e l'orizzonte (breve, medio, lungo). Il profilo cambia i pesi del punteggio qualità: il prudente pesa di più la solidità, il dinamico la crescita. L'orizzonte cambia il momento di ingresso: con orizzonte lungo pesa di più la valutazione, con orizzonte breve i segnali tecnici. Ogni scheda indica quali pesi sta usando.

### API scelte e loro limiti (piani gratuiti)

| Fornitore | Uso nell'app | Limiti gratuiti | Note |
|---|---|---|---|
| **Finnhub** | Quotazioni, notizie generali e per titolo, fondamentali (`/stock/metric`), calendario trimestrali | 60 chiamate/minuto | Solo titoli quotati negli USA: gli indici sono rappresentati da ETF (SPY, QQQ, EWI per il FTSE MIB, FEZ, EWG per il DAX, FXE, GLD, USO). I titoli europei usano l'ADR USA quando esiste (Eni = E, ASML, SAP…); quelli solo a Milano, Parigi o Francoforte restano in demo |
| **Financial Modeling Prep** (consigliato per lo storico) | Storico giornaliero: grafici, medie mobili, RSI, variazioni | 250 chiamate/giorno | Storico in cache per 12 ore |
| **Alpha Vantage** (alternativa) | Storico giornaliero, Treasury 10 anni | 25 chiamate/giorno, 100 giorni di storico | Con 100 giorni la media a 200 giorni non si calcola: l'app lo segnala |
| BTP 10 anni | Inserito a mano in Impostazioni | Nessuna API gratuita affidabile | Con BTP e Treasury disponibili l'app mostra anche la differenza in punti base |

**CORS**: Finnhub, FMP e Alpha Vantage rispondono direttamente dal browser, quindi non serve un proxy. Se un'estensione o la rete bloccano la richiesta, l'app mostra "connessione non riuscita" e passa ai dati demo.

**Consumo tipico** all'apertura del cruscotto, a cache vuota: circa 30 chiamate in tutto tra Finnhub (quotazioni, 11 settori, notizie, calendario) e FMP (storico). Riaprendo l'app nei 15 minuti successivi le chiamate sono zero, perché arriva tutto dalla cache.

---

## 2. Il file

Tutto è in [`index.html`](index.html): HTML, CSS e JavaScript vanilla, commentati in italiano. Chart.js 4.4.1 arriva da cdnjs. Se il CDN non si carica l'app funziona lo stesso, senza il grafico grande. Il tema chiaro o scuro segue il sistema, oppure lo scegli in Impostazioni.

Per pubblicarlo: *Settings → Pages → Deploy from branch* sul ramo che contiene `index.html`.

---

## 3. Come ottenere le chiavi API (gratis, 2 minuti ciascuna)

### Finnhub (prezzi, notizie, fondamentali)
1. Vai su <https://finnhub.io/register> e crea un account con la tua email.
2. Dopo la conferma, la **API key** compare nella Dashboard (<https://finnhub.io/dashboard>).
3. Nell'app: ⚙️ → *Finnhub* → incolla la chiave → **Salva**.

### Financial Modeling Prep (storico prezzi, consigliato)
1. Registrati su <https://site.financialmodelingprep.com/developer/docs> (pulsante *Get my API Key*).
2. Copia la chiave dalla pagina *Dashboard*.
3. Nell'app: ⚙️ → *Storico prezzi* = "Financial Modeling Prep" → incolla in *Chiave per lo storico* → **Salva**.

### In alternativa: Alpha Vantage
1. Richiedi la chiave su <https://www.alphavantage.co/support/#api-key> (arriva subito a video).
2. Nell'app: ⚙️ → *Storico prezzi* = "Alpha Vantage" → incolla la chiave → **Salva**.

### BTP 10 anni
Copia il rendimento da un sito finanziario (per esempio Borsa Italiana o Il Sole 24 Ore) e inseriscilo in ⚙️ → *Rendimento BTP 10 anni*.

> Le chiavi restano **solo nel tuo browser** (localStorage) e partono solo verso il fornitore a cui appartengono. Non metterle mai nel codice che pubblichi su GitHub.

---

## 4. Miglioramenti futuri

1. **Briefing automatico ogni mattina**: un'attività pianificata (per esempio GitHub Actions alle 8:00) che calcola semaforo, movimenti della watchlist e calendario del giorno e li invia per email o Telegram.
2. **Notifiche push** (PWA con service worker) quando un titolo della watchlist cambia valutazione o si muove più del 5%.
3. **App installabile e offline** (manifest + service worker) con l'ultimo cruscotto salvato.
4. **Proxy leggero** (Cloudflare Worker) che tiene nascoste le chiavi, fa da cache condivisa e sblocca fonti senza CORS: VIX vero, indici reali, BTP e spread BTP-Bund automatici.
5. **Titoli di Borsa Italiana veri** (non ADR) con un fornitore che copre Milano (EOD Historical Data, Twelve Data).
6. **Sentiment migliore**: modello linguistico per riassunto e tono, al posto del dizionario di parole chiave.
7. **Medie di settore calcolate** dai dati reali invece che indicative.
8. **Backtest educativo**: "come sarebbe andato un PAC su questo ETF negli ultimi 10 anni", con le tasse italiane.
9. **Simulatore più realistico**: commissioni, tassazione al 26% sulle plusvalenze con compensazione delle minusvalenze, cambio EUR/USD storico.
10. **Sincronizzazione** di watchlist e progressi tra dispositivi (esportazione e importazione in JSON, oppure un account).
11. **Ribilanciamento**: un controllo dell'allocazione rispetto al profilo, con avviso quando ci si allontana troppo dal target.
12. **Accessibilità**: verifica WCAG completa e lettura vocale del briefing.
