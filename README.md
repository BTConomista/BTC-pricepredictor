# BTC Price Predictor

Progetto per costruire un modello di forecast del prezzo di Bitcoin (BTC) combinando dati **macro**, **cross-asset**, **micro (market microstructure)**, **on-chain**, **sentiment** e **analisi tecnica**.

> **Stato:** fase di brainstorming. Questo documento elenca *cosa* vogliamo monitorare e *perché*.
> Le fonti dei dati (API, provider, scraping) verranno decise in una fase successiva.

---

## Indice

1. [Principi metodologici](#1-principi-metodologici)
2. [Macro](#2-macro)
3. [Altre asset class (cross-asset)](#3-altre-asset-class-cross-asset)
4. [Micro: struttura del mercato BTC](#4-micro-struttura-del-mercato-btc)
5. [On-chain](#5-on-chain)
6. [Sentiment e dati alternativi](#6-sentiment-e-dati-alternativi)
7. [Analisi tecnica](#7-analisi-tecnica)
8. [Priorità proposta](#8-priorità-proposta)
9. [Domande aperte e prossimi passi](#9-domande-aperte-e-prossimi-passi)

---

## 1. Principi metodologici

Prima ancora della lista dei dati, alcune regole che influenzano *quali* dati ha senso raccogliere e *come*.

- **L'orizzonte del forecast decide quali dati contano.** Su un orizzonte intraday (minuti/ore) domina la microstructure: order book, funding, liquidazioni. Su un orizzonte di settimane/mesi dominano liquidità globale, tassi reali, dollaro e cicli on-chain. Un dato macro mensile è quasi inutile per prevedere i prossimi 15 minuti, e viceversa.
- **Data di pubblicazione ≠ periodo di riferimento.** Il CPI di ottobre esce a metà novembre: nel dataset deve "esistere" solo dal giorno di pubblicazione, altrimenti il modello usa informazioni che nel passato non erano disponibili (*look-ahead bias*).
- **Revisioni dei dati.** GDP, payrolls e altri dati vengono rivisti più volte. Per un backtest onesto servono i *vintage* (il valore così come era noto in quella data), non solo il valore finale.
- **Frequenze miste.** Si va da tick/minuto (order book) a trimestrale (GDP). Serve una strategia di allineamento (forward-fill fino alla nuova release, modelli mixed-frequency, ecc.).
- **BTC è 24/7, le borse no.** Weekend e festività creano buchi nei dati tradizionali; i fusi orari (sessione Asia / Europa / USA) vanno gestiti esplicitamente.
- **Stazionarietà.** Lavorare su rendimenti, variazioni e spread più che su livelli assoluti.
- **Ridondanza e overfitting.** Molte variabili sono fortemente correlate tra loro (es. DAX e CAC 40). Più feature non significa modello migliore: servirà una fase di feature selection e validazione out-of-sample rigorosa.

---

## 2. Macro

### 2.1 Perimetro geografico

- **USA** come area principale: il dollaro è la valuta di riferimento di BTC, la Fed guida la liquidità globale e i flussi degli ETF spot sono quotati negli USA.
- **Estensione proposta:** G7 + area euro + Cina.
  - Nota: il G8 non esiste più dal 2014 (esclusione della Russia); oggi si parla di **G7** (USA, Canada, UK, Francia, Germania, Italia, Giappone).
  - La **Cina** va inclusa anche se non è nel G7: è la seconda economia mondiale e la PBoC è uno dei principali motori della liquidità globale.
  - Il **Giappone** è importante per un motivo specifico: lo yen è la valuta di finanziamento del *carry trade*, e un suo unwind (es. agosto 2024) colpisce tutti gli asset rischiosi, BTC compreso.
- Per i paesi non-USA probabilmente basta un set ridotto per paese: tasso di policy, CPI, PMI, rendimento 10y, valuta, bilancio della banca centrale.

### 2.2 Politica monetaria e liquidità (probabilmente il blocco macro più importante per BTC)

- **Fed funds rate** (target range e effective rate).
- **Aspettative sui tassi**: probabilità implicite nei Fed funds futures, dot plot del FOMC.
- **Bilancio della Fed** (total assets): fasi di QE / QT.
- **Treasury General Account (TGA)**: il "conto corrente" del Tesoro presso la Fed; quando sale drena liquidità dal sistema.
- **Overnight Reverse Repo (RRP)**: liquidità parcheggiata presso la Fed.
- **Net liquidity** = bilancio Fed − TGA − RRP: indicatore molto seguito nel mondo crypto.
- **Riserve bancarie** presso la Fed.
- **Money supply M2** USA e **M2 globale** (somma delle principali economie convertita in USD).
- **Tassi di mercato monetario**: SOFR, spread SOFR − Fed funds (segnale di tensione sul funding).
- **Banche centrali estere**: tassi e bilanci di BCE, BoJ, BoE, PBoC (incluse operazioni come RRR cuts e iniezioni di liquidità della PBoC).
- **Calendario**: date delle riunioni FOMC/BCE/BoJ, verbali (minutes), discorsi del presidente Fed, Jackson Hole.

### 2.3 Inflazione

- **CPI** headline e core (MoM e YoY).
- **PCE** headline e core (la misura preferita dalla Fed).
- **PPI** (inflazione alla produzione, anticipa parzialmente il CPI).
- **Aspettative di inflazione**:
  - di mercato: breakeven inflation 5y e 10y, 5y5y forward;
  - da survey: University of Michigan (1y e 5–10y).
- **Inflation nowcast** (es. Cleveland Fed).
- **Salari**: Average Hourly Earnings, Employment Cost Index (ECI).
- **Sorpresa rispetto al consensus**: più del dato in sé, il mercato reagisce allo scarto *actual − forecast*. Conviene salvare anche il consensus.

### 2.4 Crescita e mercato del lavoro

- **GDP** (advance, second, third estimate) e **GDPNow** (nowcast Atlanta Fed).
- **Lavoro**: Non-Farm Payrolls, unemployment rate, initial e continuing jobless claims (settimanali, quindi più tempestive), JOLTS (job openings).
- **PMI / survey**: ISM manufacturing e services, S&P Global PMI; equivalenti per area euro, Cina (Caixin e NBS), Giappone, UK.
- **Consumi**: retail sales, personal income & spending.
- **Produzione**: industrial production.
- **Fiducia**: Conference Board Consumer Confidence, University of Michigan Sentiment.
- **Immobiliare**: housing starts, building permits, Case-Shiller, tassi sui mutui.
- **Leading indicators**: Conference Board LEI.
- **Recession probability** (es. modello NY Fed basato sulla curva).

### 2.5 Politica fiscale e Tesoro USA

- **Deficit e debito federale.**
- **Aste dei Treasury**: bid-to-cover, *tail*, quota di acquirenti esteri (indirect bidders).
- **Quarterly Refunding Announcement (QRA)**: mix di emissioni tra T-bill e coupon; influisce su liquidità e term premium.
- **Debt ceiling**: durante gli scontri sul tetto il TGA si svuota (liquidità in ingresso), dopo la risoluzione si riempie (liquidità drenata).

### 2.6 Credito e stress finanziario

- **Spread creditizi**: High Yield OAS, Investment Grade OAS.
- **Indici di condizioni finanziarie**: Chicago Fed NFCI, St. Louis Fed Financial Stress Index.
- **MOVE index**: volatilità implicita dei Treasury (il "VIX dei bond").
- **Credito bancario**: Senior Loan Officer Survey (SLOOS), crescita dei prestiti.
- **Utilizzo delle facility di emergenza Fed** (es. discount window), utile in episodi come la crisi bancaria del marzo 2023.

### 2.7 Eventi e regolamentazione (variabili dummy / calendario)

- Date di release dei dati macro principali (CPI, NFP, FOMC).
- Elezioni USA e decisioni politiche rilevanti per il settore crypto.
- Regolamentazione: SEC/CFTC, approvazioni di ETF, MiCA in Europa, decisioni su stablecoin.
- Eventi specifici crypto: fallimenti di exchange, hack, *halving* (prossimo previsto intorno ad aprile 2028).

---

## 3. Altre asset class (cross-asset)

### 3.1 Curva dei Treasury USA

- **Scadenze**: 1m, 3m, 6m, 1y, 2y, 3y, 5y, 7y, 10y, 20y, 30y.
  - Sì, ha senso andare oltre i 2–4 anni: il **10y** è il benchmark globale per il costo del capitale e il **30y** riflette i timori fiscali e il term premium.
  - Nota: non esiste un Treasury benchmark a 4 anni; le scadenze standard sono 2, 3, 5, 7 anni.
- **Spread della curva**: 2s10s e 3m10y (inversione / disinversione come segnale di ciclo), 5s30s.
- **Rendimenti reali**: TIPS 5y e 10y. Storicamente il tasso reale 10y è uno dei driver più chiari dell'oro e, in parte, di BTC (correlazione negativa).
- **Term premium** (es. stima ACM della NY Fed).
- **Curve estere**: Bund 2y/10y, BTP 10y e spread BTP-Bund, JGB 10y (soprattutto per la politica di yield curve control della BoJ), Gilt 10y, Cina 10y.

### 3.2 ETF obbligazionari

- **ETF a scadenza fissa (iShares iBonds)**: negli USA esistono dal 2010 (Treasury iBonds dal 2020); in Europa le versioni UCITS sono arrivate nel 2023. Utili perché si comportano come un singolo bond che invecchia, quindi il loro rendimento è facilmente interpretabile.
- **ETF a duration costante**: SHY (1–3y), IEF (7–10y), TLT (20y+), BIL (T-bill).
- **ETF creditizi**: HYG / JNK (high yield), LQD (investment grade).
- Cosa monitorare: prezzo, ma soprattutto **flussi** (creazioni/rimborsi) come misura del risk appetite.
- Osservazione: per il livello dei tassi il rendimento del Treasury è un dato più pulito del prezzo dell'ETF. Gli ETF servono di più per i flussi e per avere prezzi intraday.

### 3.3 Metalli

- **Oro**: spot, flussi negli ETF (GLD, IAU), acquisti delle banche centrali. È il confronto naturale per la narrativa "digital gold".
- **Argento** e **rapporto oro/argento**.
- **Rame** e **rapporto rame/oro**: proxy della crescita globale, storicamente correlato con il rendimento 10y.
- **Platino / palladio**: probabilmente marginali, da valutare.

### 3.4 Energia e materie prime

- **Petrolio** WTI e Brent: impatto su inflazione e quindi su aspettative di politica monetaria.
- **Gas naturale** (USA Henry Hub, TTF europeo): legato anche al costo energetico del mining.
- **Indice materie prime** (es. Bloomberg Commodity Index) come sintesi.

### 3.5 Azionario USA

- **S&P 500** e **Nasdaq 100**: la correlazione BTC–equity è variabile nel tempo, quindi conviene monitorare la *rolling correlation* oltre agli indici.
- **Russell 2000**: le small cap sono molto sensibili a liquidità e tassi, spesso si muovono in modo simile a BTC.
- **Settori**: semiconduttori (SOX), tech growth.
- **Volatilità**: VIX, VVIX, term structure del VIX (contango/backwardation).
- **Azioni legate a BTC**: Strategy (MSTR), Coinbase (COIN), miner quotati (MARA, RIOT, CLSK, ecc.). Il loro premio rispetto al BTC detenuto (mNAV) misura la domanda "leveraged" di esposizione a BTC.
- **Put/call ratio** sull'azionario.

### 3.6 Indici esteri

- **Europa**: Euro Stoxx 50, DAX (Germania), CAC 40 (Francia), FTSE MIB (Italia), FTSE 100 (Londra), IBEX 35 (Spagna), SMI (Svizzera).
  - Sono molto correlati tra loro: probabilmente basta Euro Stoxx 50 + FTSE 100, con gli altri come riserva.
- **Asia**: Nikkei 225 / TOPIX (Giappone), CSI 300 / Shanghai Composite (Cina continentale), **Hang Seng** (Hong Kong, dove esistono ETF spot su BTC), **KOSPI** (Corea del Sud, mercato retail crypto molto attivo), Nifty 50 (India), ASX 200 (Australia).
- **Altri**: Bovespa (Brasile), TSX (Canada).
- **Aggregati**: MSCI World, MSCI Emerging Markets.
- Utilità specifica: BTC scambia anche quando Wall Street è chiusa. Le sessioni asiatiche ed europee danno informazioni durante la notte USA.

### 3.7 Valute

- **DXY** (indice del dollaro): storicamente correlazione negativa con BTC.
- **EUR/USD**, **USD/JPY** (carry trade), **USD/CNH** (yuan offshore, segnale di stress / svalutazione cinese), GBP/USD, USD/CHF.
- **Valute emergenti** in paesi con alta adozione di BTC/stablecoin (ARS, TRY, NGN): svalutazioni locali possono spingere la domanda.

### 3.8 Altre crypto e stablecoin

- **ETH** e rapporto **ETH/BTC**, SOL, capitalizzazione totale del mercato crypto.
- **BTC dominance**: quota di BTC sulla capitalizzazione totale (rotazione verso/da altcoin).
- **Stablecoin**: capitalizzazione di USDT e USDC, emissioni (mint) e rimborsi (burn). Nuove emissioni = "polvere da sparo" in ingresso nel mercato crypto.

---

## 4. Micro: struttura del mercato BTC

### 4.1 Prezzi e volumi spot

- Prezzo e volume **per exchange** (Coinbase, Binance, Kraken, Bitstamp, OKX, Bybit, ecc.), non solo aggregato.
- **Taker buy/sell volume** e **Cumulative Volume Delta (CVD)**: chi aggredisce il book, compratori o venditori.
- **Coinbase premium**: differenza di prezzo Coinbase (USD) vs Binance (USDT), proxy della domanda USA / istituzionale.
- **Kimchi premium**: premio sugli exchange coreani, proxy della domanda retail asiatica.
- **Stagionalità intraday e settimanale**: comportamento per sessione (Asia/Europa/USA) e nel weekend.

### 4.2 Order book e grandi ordini

- **Profondità** del book entro ±0,5% / ±1% / ±2% dal mid price.
- **Bid-ask spread.**
- **Order book imbalance**: rapporto tra liquidità in bid e in ask.
- **"Muri" di liquidità**: grandi ordini limite a livelli specifici, con monitoraggio di quanto durano (gli ordini che appaiono e spariscono possono essere *spoofing*).
- **Large trades / whale trades**: eseguiti sopra una soglia (es. > 1 M$), con direzione.
- **Iceberg orders** (ordini grandi spezzati in tranche) e **block trade OTC** dove osservabili.

### 4.3 Derivati

- **Futures perpetui**: open interest, **funding rate** (positivo = long pagano gli short, eccesso di leva rialzista), long/short ratio.
- **Liquidazioni** long e short: le cascate di liquidazioni amplificano i movimenti.
- **Basis** dei futures a scadenza (incluso CME) rispetto allo spot, annualizzato: misura la domanda di leva e il *cash-and-carry*.
- **CME**: open interest e report **COT della CFTC** (posizioni di leveraged funds vs asset manager).
- **Opzioni** (Deribit, CME, opzioni su IBIT):
  - volatilità implicita (es. indice DVOL) e sua term structure;
  - **skew 25-delta** (quanto il mercato paga per protezione al ribasso vs al rialzo);
  - put/call ratio, open interest per strike, *max pain*;
  - date di scadenza mensili/trimestrali e stima della **gamma exposure** dei market maker.

### 4.4 Flussi istituzionali

- **ETF spot BTC USA** (IBIT, FBTC, GBTC, ecc.): flussi netti giornalieri, AUM, volumi, premio/sconto sul NAV. Dal 2024 è uno dei dati più seguiti.
- **ETP europei e asiatici** su BTC.
- **Tesorerie aziendali**: acquisti di BTC da parte di società quotate (Strategy e simili).
- **Holdings governative** (es. vendite di BTC sequestrati, riserve strategiche).

### 4.5 Volatilità realizzata

- Volatilità realizzata su più finestre (1d, 7d, 30d).
- Confronto **volatilità implicita vs realizzata** (volatility risk premium).

---

## 5. On-chain

Dati unici di BTC: la blockchain è pubblica, quindi si possono osservare direttamente comportamenti di miner, exchange e holder.

### 5.1 Mining e sicurezza della rete

- **Hashrate** e **difficulty**.
- **Hash price** (ricavo per unità di hashrate) e **miner revenue** (block reward + fee).
- **Miner reserves e flussi in uscita**: vendite dei miner.
- **Puell Multiple** e **Hash Ribbons** (capitulation dei miner).

### 5.2 Exchange

- **Riserve di BTC sugli exchange** e **netflow** (afflusso = potenziale pressione in vendita, deflusso = accumulo in self-custody).
- **Stablecoin sugli exchange** (potere d'acquisto pronto all'uso).

### 5.3 Attività della rete

- Indirizzi attivi, numero di transazioni, volume trasferito.
- **Fee** e dimensione della **mempool** (congestione).

### 5.4 Comportamento degli holder

- **Long-Term Holder vs Short-Term Holder supply** e loro *realized price*.
- **HODL waves** (distribuzione dei BTC per età dell'ultimo movimento).
- **Coin Days Destroyed**: movimento di coin vecchi (vendite di holder storici).
- **Coorti di wallet** per dimensione (es. indirizzi con > 1.000 BTC).

### 5.5 Metriche di valutazione

- **Realized cap** e **realized price** (prezzo medio a cui i BTC si sono mossi l'ultima volta).
- **MVRV** e **MVRV Z-score**.
- **NUPL** (net unrealized profit/loss).
- **SOPR** (spent output profit ratio).
- **NVT** (network value to transactions).
- Modelli di ciclo (stock-to-flow, ciclo dell'halving): da trattare con molta cautela, ma da conoscere perché molti partecipanti li seguono.

---

## 6. Sentiment e dati alternativi

- **Crypto Fear & Greed Index.**
- **Google Trends** ("bitcoin", "buy bitcoin").
- **Social media**: volume e sentiment su X, Reddit, Telegram (NLP).
- **News sentiment**: analisi testuale dei titoli finanziari e crypto.
- **Ranking delle app** (Coinbase, Binance) negli store: proxy dell'ingresso di retail nuovo.
- **Wikipedia pageviews** della voce Bitcoin.
- **Mercati predittivi** (es. Polymarket) su eventi macro e politici.
- **Discorsi e testi delle banche centrali**: tono *hawkish/dovish* misurato con NLP.

---

## 7. Analisi tecnica

Osservazione importante: gli indicatori tecnici sono trasformazioni del prezzo e del volume stessi, quindi non aggiungono informazione *nuova*. Possono però aiutare il modello a catturare relazioni non lineari, e contano anche perché molti trader li usano (effetto autorealizzante su livelli molto seguiti). Vanno testati out-of-sample come qualsiasi altra feature.

- **Trend**: medie mobili SMA/EMA (20, 50, 200 giorni; 200 settimane), golden/death cross, ADX.
- **Momentum**: RSI, MACD, stocastico, rate of change.
- **Volatilità**: Bollinger Bands, ATR, Keltner Channels.
- **Volume**: OBV, VWAP e anchored VWAP, volume profile (livelli con più scambi).
- **Livelli**: supporti/resistenze, massimi storici (ATH) e distanza dall'ATH, numeri tondi, ritracciamenti di Fibonacci, pivot point.
- **Multi-timeframe**: lo stesso indicatore su 1h, 4h, 1d, 1w.

---

## 8. Priorità proposta

Per non partire con centinaia di variabili, una prima classificazione (da discutere).

- **Tier 1, da avere subito**: prezzo/volume BTC; net liquidity Fed; M2 globale; DXY; rendimenti Treasury 2y e 10y e tasso reale 10y; S&P 500 e Nasdaq 100; VIX; oro; flussi ETF spot BTC; funding rate e open interest; date FOMC/CPI/NFP con relative sorprese; capitalizzazione stablecoin.
- **Tier 2, da aggiungere poi**: resto della curva e spread; credit spread e MOVE; USD/JPY e USD/CNH; indici esteri principali; opzioni (IV, skew); liquidazioni; Coinbase premium; metriche on-chain principali (exchange netflow, MVRV, LTH/STH).
- **Tier 3, esplorativo**: order book a livello di tick; sentiment social/news via NLP; macro dettagliata dei singoli paesi non-USA; metriche on-chain di nicchia.

---

## 9. Domande aperte e prossimi passi

- **Orizzonte del forecast**: intraday, giornaliero, settimanale o mensile? È la scelta che più condiziona il resto.
- **Target**: prezzo, rendimento, direzione (su/giù) o volatilità?
- **Profondità storica**: da quando? (ETF spot solo dal 2024, molte metriche di derivati solo dal 2019–2020.)
- **Fonti dei dati**: prossimo step, mappare per ogni variabile una fonte gratuita o a pagamento, frequenza, storico disponibile e ritardo di pubblicazione.
- **Storage e pipeline**: formato, frequenza di aggiornamento, gestione dei vintage.
