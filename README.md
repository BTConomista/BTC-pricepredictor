# BTC Price Predictor

Progetto per costruire un modello di forecast del prezzo di Bitcoin (BTC) combinando dati **macro**, **cross-asset**, **micro (market microstructure)**, **fondamentali di Bitcoin (mining, energia, on-chain)**, **sentiment** e **analisi tecnica**, partendo da uno studio storico di come ciascuno di questi fattori ha influenzato BTC in passato.

> **Stato:** fase di brainstorming. Questo documento elenca *cosa* vogliamo monitorare e studiare, e *perché*.
> Le fonti dei dati (API, provider, scraping) verranno decise in una fase successiva.

---

## Indice

1. [Principi metodologici](#1-principi-metodologici)
2. [Orizzonti temporali e ciclo di Bitcoin](#2-orizzonti-temporali-e-ciclo-di-bitcoin)
3. [Macro](#3-macro)
4. [Altre asset class (cross-asset)](#4-altre-asset-class-cross-asset)
5. [Micro: struttura del mercato BTC](#5-micro-struttura-del-mercato-btc)
6. [Fondamentali di Bitcoin: mining, energia, on-chain](#6-fondamentali-di-bitcoin-mining-energia-on-chain)
7. [Sentiment e dati alternativi](#7-sentiment-e-dati-alternativi)
8. [Analisi tecnica](#8-analisi-tecnica)
9. [Studio storico: cosa ha mosso BTC in passato](#9-studio-storico-cosa-ha-mosso-btc-in-passato)
10. [Priorità proposta](#10-priorità-proposta)
11. [Domande aperte e prossimi passi](#11-domande-aperte-e-prossimi-passi)

---

## 1. Principi metodologici

Prima ancora della lista dei dati, alcune regole che influenzano *quali* dati ha senso raccogliere e *come*.

- **L'orizzonte del forecast decide quali dati contano.** Su un orizzonte intraday (minuti/ore) domina la microstructure: order book, funding, liquidazioni. Su un orizzonte di settimane/mesi dominano liquidità globale, tassi reali, dollaro e cicli on-chain. Un dato macro mensile è quasi inutile per prevedere le prossime 4 ore, e viceversa.
- **Data di pubblicazione ≠ periodo di riferimento.** Il CPI di ottobre esce a metà novembre: nel dataset deve "esistere" solo dal giorno di pubblicazione, altrimenti il modello usa informazioni che nel passato non erano disponibili (*look-ahead bias*).
- **Revisioni dei dati.** GDP, payrolls e altri dati vengono rivisti più volte. Per un backtest onesto servono i *vintage* (il valore così come era noto in quella data), non solo il valore finale.
- **Frequenze miste.** Si va da tick/minuto (order book) a trimestrale (GDP). Serve una strategia di allineamento (forward-fill fino alla nuova release, modelli mixed-frequency, ecc.).
- **BTC è 24/7, le borse no.** Weekend e festività creano buchi nei dati tradizionali; i fusi orari (sessione Asia / Europa / USA) vanno gestiti esplicitamente.
- **Stazionarietà.** Lavorare su rendimenti, variazioni e spread più che su livelli assoluti: due serie che salgono entrambe nel tempo risultano "correlate" anche senza alcun legame (*spurious correlation*).
- **Ridondanza e overfitting.** Molte variabili sono fortemente correlate tra loro (es. DAX e CAC 40). Più feature non significa modello migliore: servirà una fase di feature selection e validazione out-of-sample rigorosa.

---

## 2. Orizzonti temporali e ciclo di Bitcoin

### 2.1 Struttura a tre livelli

L'idea è non avere un unico modello, ma tre livelli che si parlano: il livello lungo non prevede un prezzo, ma dice **in che fase del ciclo siamo**; questa informazione diventa poi un input (o un filtro) per i modelli di medio e breve periodo. Un segnale rialzista a 4h ha un significato diverso a 6 mesi dall'halving rispetto a 6 mesi dopo il top del ciclo.

- **Lungo periodo (il ciclo):** l'orizzonte è la fine del ciclo in corso. Obiettivo: classificare la fase (accumulazione, espansione post-halving, euforia/top, bear market) e stimare quanto manca, in tempo e in prezzo, ai punti di svolta tipici. Non ha senso guardare oltre il ciclo corrente.
- **Medio periodo:** 1 settimana, e in prospettiva 1 mese.
- **Breve periodo:** 1 giorno e 4 ore.

### 2.2 Orizzonti da testare per primi

Distinzione utile: l'**orizzonte** è quanto avanti prevediamo; il **timeframe** è la durata della candela con cui guardiamo i dati. Si può prevedere la prossima settimana usando dati giornalieri.

- **1 day: il livello di riferimento con cui partire.** È il miglior compromesso: quasi tutte le fonti (macro, ETF, on-chain, derivati) hanno almeno frequenza giornaliera, il rumore è minore che a 4h e lo storico è ampio (dal 2014 circa 4.700 osservazioni).
- **1 week:** è l'orizzonte in cui macro, liquidità e flussi ETF dovrebbero contare di più. Limite: dal 2014 ci sono solo circa 670 candele settimanali, quindi il rischio di overfitting è alto e il modello deve restare semplice.
- **4 h:** tantissime osservazioni (circa 28.000 dal 2014) ma molto più rumore; qui contano soprattutto derivati e microstructure, il cui storico affidabile parte solo dal 2019–2020. Da affrontare dopo che il daily funziona.
- **Aggiunta proposta, 1 month:** fa da ponte tra il medio periodo e il ciclo, ed è il naturale livello per studiare la stagionalità mensile.
- **Più avanti, eventualmente, 1 h:** solo se i dati micro dimostrano di avere potere predittivo.

Dettagli pratici da fissare:

- **Chiusura delle candele:** daily alle 00:00 UTC (convenzione crypto), weekly domenica alle 00:00 UTC, 4h su 00/04/08/12/16/20 UTC. I dati tradizionali chiudono in orari diversi (es. 16:00 New York): va deciso come allinearli.
- **Gap del CME:** i futures CME non scambiano nel weekend, e i "gap" tra chiusura del venerdì e riapertura sono seguiti da molti trader.
- **Cosa prevedere ai vari orizzonti:** probabilmente rendimento o direzione per 1d/1w, ma anche **volatilità**, che è storicamente molto più prevedibile del rendimento.

### 2.3 Il ciclo: cosa dicono i dati

Le date principali dei cicli passati (date e prezzi indicativi):

- **Halving:** 28/11/2012, 09/07/2016, 11/05/2020, 20/04/2024. Il prossimo è atteso intorno a primavera 2028 (blocco 1.050.000).
- **Top di ciclo:** dicembre 2013 (~1.150 $), dicembre 2017 (~19.700 $), novembre 2021 (~69.000 $), ottobre 2025 (~126.000 $). Distanza dall'halving: circa 371, 526, 548, 534 giorni.
- **Minimi del bear market:** gennaio 2015 (~−86%), dicembre 2018 (~−84%), novembre 2022 (~−77%). Distanza dal top: circa 406, 363, 376 giorni.
- **Presidenziali USA:** novembre 2012, 2016, 2020, 2024.

Osservazioni critiche, da tenere presenti prima di costruirci sopra:

- **Campione minuscolo.** Abbiamo 3 cicli completi e uno in corso. Qualsiasi regolarità su 3–4 osservazioni può essere caso: è un'ipotesi da monitorare, non una legge.
- **Halving e presidenziali sono confusi statisticamente.** Finora ogni halving è caduto nello stesso anno delle presidenziali USA, quindi con i dati non si può distinguere quale dei due faccia cosa. L'ipotesi "l'halving dà l'impulso, le elezioni lo alimentano" è plausibile, ma non verificabile separatamente. L'intervallo tra halving si sta allungando verso 4 anni pieni (3,6 → 3,8 → 3,9 anni), quindi i due eventi resteranno allineati ancora a lungo (anche nel 2028).
- **Spiegazioni alternative da testare:**
  - **ciclo della liquidità globale** (espansioni e contrazioni dei bilanci delle banche centrali e del credito, con una periodicità non lontana dai 4 anni);
  - **ciclo dei tassi della Fed**;
  - **profezia autorealizzante**: molti partecipanti credono nel ciclo e si comportano di conseguenza.
- **L'impulso dell'halving si sta riducendo.** Dopo il 2024 l'emissione è di 450 BTC al giorno (3,125 BTC × circa 144 blocchi). In molti giorni gli ETF spot comprano più BTC di quanti ne vengano emessi: il peso relativo dell'offerta nuova è sempre più piccolo, e la domanda istituzionale pesa sempre di più.
- **Il ciclo 2024 ha già mostrato un'anomalia:** per la prima volta BTC ha fatto un nuovo massimo storico *prima* dell'halving (marzo 2024), spinto dai flussi ETF.
- **Rendimenti decrescenti:** ogni ciclo ha avuto un moltiplicatore dal minimo al massimo più basso del precedente, e drawdown più contenuti.
- **Test in corso:** se la regolarità storica si ripetesse, il minimo del bear market cadrebbe intorno a 12–13 mesi dopo il top di ottobre 2025, cioè circa fine 2026. Siamo esattamente nella finestra in cui l'ipotesi del ciclo viene messa alla prova.

### 2.4 Indicatori della fase del ciclo

Feature costruibili per dire al modello "dove siamo":

- giorni dall'ultimo halving e giorni al prossimo;
- giorni dall'ultimo massimo storico e drawdown dal massimo;
- rapporto tra prezzo e medie di lungo periodo (200 settimane, realized price);
- metriche on-chain di ciclo (MVRV, Puell Multiple, quota di supply dei Long-Term Holder; vedi sezione 6);
- **ciclo sovrapposto:** prezzo normalizzato e allineato al giorno dell'halving (o al minimo del bear market), per confrontare il ciclo attuale con quelli passati.

### 2.5 Stagionalità

- **Mensile:** rendimento medio e mediano per mese, e percentuale di mesi positivi (es. il cosiddetto "Uptober" e la debolezza di settembre).
- **Per anno del ciclo:** anno dell'halving, anno successivo, anno del bear, anno di accumulazione. Confronto con il *ciclo presidenziale* noto nell'azionario USA (dove storicamente il terzo anno del mandato è il più forte).
- **Settimanale:** giorno della settimana, weekend vs giorni lavorativi.
- **Intraday:** ora del giorno, sessione Asia / Europa / USA, apertura di Wall Street.
- **Attorno agli eventi:** scadenze delle opzioni, FOMC, CPI, fine trimestre (ribilanciamenti).
- **Attenzione:** con 12 mesi × 12 anni ogni media mensile si basa su circa 12 osservazioni; testando molte stagionalità, qualcuna risulterà "significativa" per caso. Serve guardare anche la mediana e la dispersione, non solo la media.

### 2.6 Dati storici di BTC

- **Prezzo giornaliero** dal 2010 (i dati 2010–2013 vengono quasi solo da Mt. Gox e sono poco affidabili).
- **Prezzo intraday (minuto/ora)** dal 2013–2014 circa sui principali exchange (Bitstamp, Coinbase, Kraken).
- **Problema del wash trading:** molti exchange hanno gonfiato i volumi in passato; meglio usare un insieme ristretto di exchange affidabili.
- **Dati on-chain completi dal 2009:** sono tutti nella blockchain. Gestendo un **nostro nodo Bitcoin** potremmo calcolare in autonomia hashrate, difficulty, fee, flussi, età dei coin e metriche di valutazione, senza dipendere da provider a pagamento.

---

## 3. Macro

### 3.1 Perimetro geografico

- **USA** come area principale: il dollaro è la valuta di riferimento di BTC, la Fed guida la liquidità globale e i flussi degli ETF spot sono quotati negli USA.
- **Estensione proposta:** G7 + area euro + Cina.
  - Nota: il G8 non esiste più dal 2014 (esclusione della Russia); oggi si parla di **G7** (USA, Canada, UK, Francia, Germania, Italia, Giappone).
  - La **Cina** va inclusa anche se non è nel G7: è la seconda economia mondiale e la PBoC è uno dei principali motori della liquidità globale.
  - Il **Giappone** è importante per un motivo specifico: lo yen è la valuta di finanziamento del *carry trade*, e un suo unwind (es. agosto 2024) colpisce tutti gli asset rischiosi, BTC compreso.
- Per i paesi non-USA probabilmente basta un set ridotto per paese: tasso di policy, CPI, PMI, rendimento 10y, valuta, bilancio della banca centrale.

### 3.2 Politica monetaria e liquidità (probabilmente il blocco macro più importante per BTC)

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

### 3.3 Inflazione

- **CPI** headline e core (MoM e YoY).
- **PCE** headline e core (la misura preferita dalla Fed).
- **PPI** (inflazione alla produzione, anticipa parzialmente il CPI).
- **Aspettative di inflazione**:
  - di mercato: breakeven inflation 5y e 10y, 5y5y forward;
  - da survey: University of Michigan (1y e 5–10y).
- **Inflation nowcast** (es. Cleveland Fed).
- **Salari**: Average Hourly Earnings, Employment Cost Index (ECI).
- **Sorpresa rispetto al consensus**: più del dato in sé, il mercato reagisce allo scarto *actual − forecast*. Conviene salvare anche il consensus.

### 3.4 Crescita e mercato del lavoro

- **GDP** (advance, second, third estimate) e **GDPNow** (nowcast Atlanta Fed).
- **Lavoro**: Non-Farm Payrolls, unemployment rate, initial e continuing jobless claims (settimanali, quindi più tempestive), JOLTS (job openings).
- **PMI / survey**: ISM manufacturing e services, S&P Global PMI; equivalenti per area euro, Cina (Caixin e NBS), Giappone, UK.
- **Consumi**: retail sales, personal income & spending.
- **Produzione**: industrial production.
- **Fiducia**: Conference Board Consumer Confidence, University of Michigan Sentiment.
- **Immobiliare**: housing starts, building permits, Case-Shiller, tassi sui mutui.
- **Leading indicators**: Conference Board LEI.
- **Recession probability** (es. modello NY Fed basato sulla curva).

### 3.5 Politica fiscale e Tesoro USA

- **Deficit e debito federale.**
- **Aste dei Treasury**: bid-to-cover, *tail*, quota di acquirenti esteri (indirect bidders).
- **Quarterly Refunding Announcement (QRA)**: mix di emissioni tra T-bill e coupon; influisce su liquidità e term premium.
- **Debt ceiling**: durante gli scontri sul tetto il TGA si svuota (liquidità in ingresso), dopo la risoluzione si riempie (liquidità drenata).

### 3.6 Credito e stress finanziario

- **Spread creditizi**: High Yield OAS, Investment Grade OAS.
- **Indici di condizioni finanziarie**: Chicago Fed NFCI, St. Louis Fed Financial Stress Index.
- **MOVE index**: volatilità implicita dei Treasury (il "VIX dei bond").
- **Credito bancario**: Senior Loan Officer Survey (SLOOS), crescita dei prestiti.
- **Utilizzo delle facility di emergenza Fed** (es. discount window), utile in episodi come la crisi bancaria del marzo 2023.

### 3.7 Eventi e regolamentazione (variabili dummy / calendario)

- Date di release dei dati macro principali (CPI, NFP, FOMC).
- Elezioni USA (presidenziali e midterm) e decisioni politiche rilevanti per il settore crypto.
- Regolamentazione: SEC/CFTC, approvazioni di ETF, MiCA in Europa, decisioni su stablecoin.
- Eventi specifici crypto: fallimenti di exchange, hack, halving.

---

## 4. Altre asset class (cross-asset)

### 4.1 Curva dei Treasury USA

- **Scadenze**: 1m, 3m, 6m, 1y, 2y, 3y, 5y, 7y, 10y, 20y, 30y.
  - Sì, ha senso andare oltre i 2–4 anni: il **10y** è il benchmark globale per il costo del capitale e il **30y** riflette i timori fiscali e il term premium.
  - Nota: non esiste un Treasury benchmark a 4 anni; le scadenze standard sono 2, 3, 5, 7 anni.
- **Spread della curva**: 2s10s e 3m10y (inversione / disinversione come segnale di ciclo), 5s30s.
- **Rendimenti reali**: TIPS 5y e 10y. Storicamente il tasso reale 10y è uno dei driver più chiari dell'oro e, in parte, di BTC (correlazione negativa).
- **Term premium** (es. stima ACM della NY Fed).
- **Curve estere**: Bund 2y/10y, BTP 10y e spread BTP-Bund, JGB 10y (soprattutto per la politica di yield curve control della BoJ), Gilt 10y, Cina 10y.

### 4.2 ETF obbligazionari

- **ETF a scadenza fissa (iShares iBonds)**: negli USA esistono dal 2010 (Treasury iBonds dal 2020); in Europa le versioni UCITS sono arrivate nel 2023. Utili perché si comportano come un singolo bond che invecchia, quindi il loro rendimento è facilmente interpretabile.
- **ETF a duration costante**: SHY (1–3y), IEF (7–10y), TLT (20y+), BIL (T-bill).
- **ETF creditizi**: HYG / JNK (high yield), LQD (investment grade).
- Cosa monitorare: prezzo, ma soprattutto **flussi** (creazioni/rimborsi) come misura del risk appetite.
- Osservazione: per il livello dei tassi il rendimento del Treasury è un dato più pulito del prezzo dell'ETF. Gli ETF servono di più per i flussi e per avere prezzi intraday.

### 4.3 Metalli

- **Oro**: spot, flussi negli ETF (GLD, IAU), acquisti delle banche centrali. È il confronto naturale per la narrativa "digital gold".
- **Argento** e **rapporto oro/argento**.
- **Rame** e **rapporto rame/oro**: proxy della crescita globale, storicamente correlato con il rendimento 10y.
- **Platino / palladio**: probabilmente marginali, da valutare.

### 4.4 Energia e materie prime

- **Petrolio** WTI e Brent: impatto su inflazione e quindi su aspettative di politica monetaria.
- **Gas naturale** ed **elettricità**: trattati in dettaglio nella sezione 6.2, perché legati direttamente al costo del mining.
- **Indice materie prime** (es. Bloomberg Commodity Index) come sintesi.

### 4.5 Azionario USA

- **S&P 500** e **Nasdaq 100**: la correlazione BTC–equity è variabile nel tempo, quindi conviene monitorare la *rolling correlation* oltre agli indici.
- **Russell 2000**: le small cap sono molto sensibili a liquidità e tassi, spesso si muovono in modo simile a BTC.
- **Settori**: semiconduttori (SOX), tech growth.
- **Volatilità**: VIX, VVIX, term structure del VIX (contango/backwardation).
- **Azioni legate a BTC**: Strategy (MSTR), Coinbase (COIN), miner e produttori di hardware (vedi sezione 6.3). Il premio di Strategy rispetto al BTC detenuto (mNAV) misura la domanda "leveraged" di esposizione a BTC.
- **Put/call ratio** sull'azionario.

### 4.6 Indici esteri

- **Europa**: Euro Stoxx 50, DAX (Germania), CAC 40 (Francia), FTSE MIB (Italia), FTSE 100 (Londra), IBEX 35 (Spagna), SMI (Svizzera).
  - Sono molto correlati tra loro: probabilmente basta Euro Stoxx 50 + FTSE 100, con gli altri come riserva.
- **Asia**: Nikkei 225 / TOPIX (Giappone), CSI 300 / Shanghai Composite (Cina continentale), **Hang Seng** (Hong Kong, dove esistono ETF spot su BTC), **KOSPI** (Corea del Sud, mercato retail crypto molto attivo), Nifty 50 (India), ASX 200 (Australia).
- **Altri**: Bovespa (Brasile), TSX (Canada).
- **Aggregati**: MSCI World, MSCI Emerging Markets.
- Utilità specifica: BTC scambia anche quando Wall Street è chiusa. Le sessioni asiatiche ed europee danno informazioni durante la notte USA.

### 4.7 Valute

- **DXY** (indice del dollaro): storicamente correlazione negativa con BTC.
- **EUR/USD**, **USD/JPY** (carry trade), **USD/CNH** (yuan offshore, segnale di stress / svalutazione cinese), GBP/USD, USD/CHF.
- **Valute emergenti** in paesi con alta adozione di BTC/stablecoin (ARS, TRY, NGN): svalutazioni locali possono spingere la domanda.

### 4.8 Altre crypto e stablecoin

- **ETH** e rapporto **ETH/BTC**, SOL, capitalizzazione totale del mercato crypto.
- **BTC dominance**: quota di BTC sulla capitalizzazione totale (rotazione verso/da altcoin).
- **Stablecoin**: capitalizzazione di USDT e USDC, emissioni (mint) e rimborsi (burn). Nuove emissioni = "polvere da sparo" in ingresso nel mercato crypto.

---

## 5. Micro: struttura del mercato BTC

### 5.1 Prezzi e volumi spot

- Prezzo e volume **per exchange** (Coinbase, Binance, Kraken, Bitstamp, OKX, Bybit, ecc.), non solo aggregato.
- **Taker buy/sell volume** e **Cumulative Volume Delta (CVD)**: chi aggredisce il book, compratori o venditori.
- **Coinbase premium**: differenza di prezzo Coinbase (USD) vs Binance (USDT), proxy della domanda USA / istituzionale.
- **Kimchi premium**: premio sugli exchange coreani, proxy della domanda retail asiatica.
- **Stagionalità intraday e settimanale**: vedi sezione 2.5.

### 5.2 Order book e grandi ordini

- **Profondità** del book entro ±0,5% / ±1% / ±2% dal mid price.
- **Bid-ask spread.**
- **Order book imbalance**: rapporto tra liquidità in bid e in ask.
- **"Muri" di liquidità**: grandi ordini limite a livelli specifici, con monitoraggio di quanto durano (gli ordini che appaiono e spariscono possono essere *spoofing*).
- **Large trades / whale trades**: eseguiti sopra una soglia (es. > 1 M$), con direzione.
- **Iceberg orders** (ordini grandi spezzati in tranche) e **block trade OTC** dove osservabili.

### 5.3 Derivati

- **Futures perpetui**: open interest, **funding rate** (positivo = long pagano gli short, eccesso di leva rialzista), long/short ratio.
- **Liquidazioni** long e short: le cascate di liquidazioni amplificano i movimenti.
- **Basis** dei futures a scadenza (incluso CME) rispetto allo spot, annualizzato: misura la domanda di leva e il *cash-and-carry*.
- **CME**: open interest e report **COT della CFTC** (posizioni di leveraged funds vs asset manager).
- **Opzioni** (Deribit, CME, opzioni su IBIT):
  - volatilità implicita (es. indice DVOL) e sua term structure;
  - **skew 25-delta** (quanto il mercato paga per protezione al ribasso vs al rialzo);
  - put/call ratio, open interest per strike, *max pain*;
  - date di scadenza mensili/trimestrali e stima della **gamma exposure** dei market maker.

### 5.4 Flussi istituzionali

- **ETF spot BTC USA** (IBIT, FBTC, GBTC, ecc.): flussi netti giornalieri, AUM, volumi, premio/sconto sul NAV. Dal 2024 è uno dei dati più seguiti.
- **Flussi ETF espressi in BTC confrontati con l'emissione giornaliera** (450 BTC/giorno): misura quanto la domanda istituzionale assorbe l'offerta nuova.
- **ETP europei e asiatici** su BTC.
- **Tesorerie aziendali**: acquisti di BTC da parte di società quotate (Strategy e simili).
- **Holdings governative** (es. vendite di BTC sequestrati, riserve strategiche).

### 5.5 Volatilità realizzata

- Volatilità realizzata su più finestre (1d, 7d, 30d).
- Confronto **volatilità implicita vs realizzata** (volatility risk premium).

---

## 6. Fondamentali di Bitcoin: mining, energia, on-chain

Dati unici di BTC: la blockchain è pubblica, quindi si possono osservare direttamente comportamenti di miner, exchange e holder.

### 6.1 Mining e sicurezza della rete

- **Hashrate** (potenza di calcolo totale) e **difficulty**, con il suo aggiustamento ogni 2.016 blocchi (circa ogni due settimane).
- **Hash price**: ricavo per unità di hashrate al giorno (es. $/PH/giorno). Sintetizza in un solo numero prezzo, difficulty e fee: è il dato che guida le decisioni dei miner.
- **Miner revenue**: block reward + fee, e **quota delle fee** sul totale (con ricompense sempre più basse, le fee diventano strutturalmente più importanti).
- **Costo di produzione di 1 BTC**: stimato da costo dell'energia, efficienza media della rete (J/TH) e difficulty. Storicamente funziona da "pavimento morbido": sotto il costo, i miner meno efficienti spengono le macchine o vendono.
- **Prezzo di breakeven per modello di ASIC** (es. generazioni S19, S21): quando il prezzo scende sotto il breakeven delle macchine vecchie, l'hashrate cala.
- **Miner reserves e flussi in uscita** verso gli exchange: vendite dei miner.
- **Miner capitulation**: Puell Multiple, Hash Ribbons (incrocio tra medie dell'hashrate), cali di hashrate dopo l'halving.
- **Tempo medio di blocco** e **distribuzione dell'hashrate per mining pool**.
- **Distribuzione geografica dell'hashrate**: dopo il divieto cinese del 2021 la quota USA è cresciuta molto; spostamenti geografici hanno causato in passato cali bruschi di hashrate.

### 6.2 Energia

Ipotesi da studiare: il prezzo dell'energia influenza soprattutto i **margini dei miner** e quindi l'**hashrate**; l'effetto sul prezzo di BTC è probabilmente indiretto e debole. È anzi plausibile che la causalità vada principalmente nell'altra direzione: è il prezzo di BTC a guidare l'hashrate ("hashrate follows price"), con ritardo di settimane o mesi. Lo studio storico deve testare entrambe le direzioni.

- **Prezzi dell'elettricità** nelle principali aree di mining:
  - Texas (**ERCOT**): prezzi spot e real-time; i miner partecipano al *demand response* e spengono le macchine nei picchi estivi, causando cali di hashrate visibili.
  - altre aree USA (Georgia, New York, ecc.), Canada, paesi nordici, Paraguay, Etiopia, Kazakistan, Russia.
- **Gas naturale** (Henry Hub USA, TTF europeo): in Texas è spesso il combustibile marginale che fissa il prezzo dell'elettricità; inoltre esiste il mining con gas di torcia (*flare gas*).
- **Petrolio**: canale indiretto via inflazione e politica monetaria.
- **Stagionalità storica:** prima del 2021, la stagione delle piogge nel Sichuan (energia idroelettrica a basso costo) spostava i miner cinesi e muoveva l'hashrate. Oggi la stagionalità è più legata ai picchi estivi in Texas.
- **Shock energetici** da usare come casi di studio: crisi energetica europea 2022, tempesta invernale in Texas (febbraio 2021), divieto cinese (2021).

### 6.3 Aziende dell'hardware e del mining

Domanda da studiare: le aziende legate al mining **anticipano** BTC, lo **seguono** con leva, o si muovono per conto loro?

- **Produttori di ASIC:**
  - **Bitmain** (leader di mercato) e **MicroBT**: non quotate, quindi niente prezzi di borsa; si possono seguire solo notizie, prezzi delle macchine e indiscrezioni su ordini.
  - **Canaan** (CAN, Nasdaq) ed **Ebang** (EBON): quotate, ma piccole e poco liquide.
  - **Bitdeer** (BTDR): miner che produce anche i propri ASIC.
- **Fornitori di chip:** TSMC e Samsung producono i chip degli ASIC, ma il mining è una frazione minima del loro fatturato: correlazione attesa quasi nulla.
- **Miner quotati:** MARA, RIOT, CleanSpark (CLSK), Cipher (CIFR), Hut 8 (HUT), IREN, Core Scientific (CORZ), Bitfarms (BITF), TeraWulf (WULF). ETF tematico: WGMI.
- **Prezzo di mercato degli ASIC** ($/TH): molto legato all'hash price, è un buon termometro della fiducia dei miner.
- **Osservazione importante:** dal 2024 molti miner quotati stanno convertendo parte dei data center in infrastruttura per **AI/HPC**. Questo cambia il loro profilo: il loro titolo dipende sempre più dal settore AI e sempre meno da BTC. La correlazione storica va quindi studiata **per periodi** (prima e dopo questa svolta).
- **Dati trimestrali**: produzione di BTC, costo per BTC prodotto, BTC venduti/trattenuti, ordini di macchine (dai report dei miner e dei produttori di ASIC).
- **Test lead-lag:** i titoli scambiano solo in orario USA mentre BTC scambia sempre; va verificato se i movimenti dei titoli all'apertura contengono informazione su BTC o se si limitano a recepire quanto BTC ha fatto nella notte.

### 6.4 Exchange

- **Riserve di BTC sugli exchange** e **netflow** (afflusso = potenziale pressione in vendita, deflusso = accumulo in self-custody).
- **Stablecoin sugli exchange** (potere d'acquisto pronto all'uso).

### 6.5 Attività della rete

- Indirizzi attivi, numero di transazioni, volume trasferito.
- **Fee** e dimensione della **mempool** (congestione).

### 6.6 Comportamento degli holder

- **Long-Term Holder vs Short-Term Holder supply** e loro *realized price*.
- **HODL waves** (distribuzione dei BTC per età dell'ultimo movimento).
- **Coin Days Destroyed**: movimento di coin vecchi (vendite di holder storici).
- **Coorti di wallet** per dimensione (es. indirizzi con > 1.000 BTC).
- **Illiquid supply**: BTC detenuti da wallet che storicamente non vendono.

### 6.7 Metriche di valutazione

- **Realized cap** e **realized price** (prezzo medio a cui i BTC si sono mossi l'ultima volta).
- **MVRV** e **MVRV Z-score**.
- **NUPL** (net unrealized profit/loss).
- **SOPR** (spent output profit ratio).
- **NVT** (network value to transactions).
- Modelli di lungo periodo (stock-to-flow, regressione logaritmica / *power law*): da trattare con molta cautela, ma da conoscere perché molti partecipanti li seguono.

---

## 7. Sentiment e dati alternativi

- **Crypto Fear & Greed Index.**
- **Google Trends** ("bitcoin", "buy bitcoin").
- **Social media**: volume e sentiment su X, Reddit, Telegram (NLP).
- **News sentiment**: analisi testuale dei titoli finanziari e crypto.
- **Ranking delle app** (Coinbase, Binance) negli store: proxy dell'ingresso di retail nuovo.
- **Wikipedia pageviews** della voce Bitcoin.
- **Mercati predittivi** (es. Polymarket) su eventi macro e politici.
- **Discorsi e testi delle banche centrali**: tono *hawkish/dovish* misurato con NLP.

---

## 8. Analisi tecnica

Osservazione importante: gli indicatori tecnici sono trasformazioni del prezzo e del volume stessi, quindi non aggiungono informazione *nuova*. Possono però aiutare il modello a catturare relazioni non lineari, e contano anche perché molti trader li usano (effetto autorealizzante su livelli molto seguiti). Vanno testati out-of-sample come qualsiasi altra feature.

- **Trend**: medie mobili SMA/EMA (20, 50, 200 giorni; 200 settimane), golden/death cross, ADX.
- **Momentum**: RSI, MACD, stocastico, rate of change.
- **Volatilità**: Bollinger Bands, ATR, Keltner Channels.
- **Volume**: OBV, VWAP e anchored VWAP, volume profile (livelli con più scambi).
- **Livelli**: supporti/resistenze, massimi storici (ATH) e distanza dall'ATH, numeri tondi, ritracciamenti di Fibonacci, pivot point.
- **Multi-timeframe**: lo stesso indicatore su 4h, 1d, 1w, 1M, coerentemente con gli orizzonti della sezione 2.

---

## 9. Studio storico: cosa ha mosso BTC in passato

Prima di costruire il modello, l'idea è studiare per ogni variabile delle sezioni precedenti **se**, **quando** e **come** ha influenzato BTC. Questo studio serve anche a scartare le variabili inutili prima di darle al modello.

### 9.1 Domande da porsi per ogni variabile

- **Correlazione:** si muove insieme a BTC? Con che segno e che intensità?
- **Stabilità nel tempo:** la relazione è costante o cambia? Esempio: la correlazione BTC–Nasdaq era vicina a zero prima del 2020 ed è diventata forte dopo; con BTC–oro è successo il contrario in vari periodi.
- **Lead-lag:** la variabile anticipa BTC, lo segue o si muovono insieme? Con quale ritardo? (Esempio molto citato: BTC seguirebbe l'M2 globale con circa 10–12 settimane di ritardo. Da verificare, non da dare per buono.)
- **Asimmetria:** l'effetto è uguale quando la variabile sale e quando scende? (Es. stress del credito: forse conta solo quando esplode.)
- **Dipendenza dal regime:** l'effetto cambia tra fasi del ciclo, tra bull e bear market, tra periodi di tassi alti e bassi?

### 9.2 Strumenti di analisi

- **Rolling correlation** su più finestre (30, 90, 365 giorni).
- **Cross-correlation** a diversi ritardi, per gli effetti lead-lag.
- **Test di Granger causality**: verifica se il passato di una variabile aiuta a prevedere BTC oltre al passato di BTC stesso (non è causalità in senso stretto, solo potere predittivo).
- **Cointegrazione**, per relazioni di lungo periodo tra livelli (es. BTC e liquidità).
- **Event study**: comportamento di BTC nelle ore/giorni attorno a release macro (CPI, FOMC, NFP), halving, approvazioni di ETF, crolli di exchange.
- **Regressioni per regime** e modelli a cambio di regime (*Markov switching*).
- **Test di rottura strutturale** per individuare i momenti in cui le relazioni cambiano.

### 9.3 Rotture strutturali e periodi da distinguere

- **2010–2013:** mercato minuscolo, dati poco affidabili (era Mt. Gox).
- **2014–2019:** mercato retail, BTC poco legato ai mercati tradizionali.
- **2020–2023:** ingresso istituzionale, Covid e liquidità record, poi rialzo dei tassi; correlazione forte con l'azionario tech.
- **Dal 2024:** ETF spot USA, nuovi grandi acquirenti (ETF, tesorerie aziendali), svolta AI dei miner.

### 9.4 Archivio degli eventi eccezionali

Un elenco datato di shock da etichettare nel dataset, per non confonderli con relazioni normali: crollo di Mt. Gox (2014), ban cinesi (2017 e 2021), crollo del marzo 2020 (Covid), collasso di Terra/Luna e Celsius (2022), fallimento di FTX (novembre 2022), crisi bancaria USA (marzo 2023), approvazione degli ETF spot (gennaio 2024), unwind del carry trade sullo yen (agosto 2024).

### 9.5 Rischi da evitare

- **Data snooping / multiple testing:** testando centinaia di variabili, alcune risulteranno correlate con BTC per puro caso. Contromisure: tenere da parte un periodo di test mai guardato durante l'analisi, correggere per test multipli, preferire relazioni con una spiegazione economica.
- **Correlazioni spurie tra trend:** lavorare su rendimenti e variazioni, non su livelli.
- **Survivorship bias:** includere anche exchange e asset che non esistono più, dove rilevante.

---

## 10. Priorità proposta

Per non partire con centinaia di variabili, una prima classificazione (da discutere).

- **Tier 1, da avere subito**: storico prezzo/volume BTC (daily dal 2010, intraday dal 2013–2014); indicatori di fase del ciclo (giorni dall'halving, drawdown dall'ATH); hashrate, difficulty, hash price; net liquidity Fed; M2 globale; DXY; rendimenti Treasury 2y e 10y e tasso reale 10y; S&P 500 e Nasdaq 100; VIX; oro; flussi ETF spot BTC; funding rate e open interest; date FOMC/CPI/NFP con relative sorprese; capitalizzazione stablecoin.
- **Tier 2, da aggiungere poi**: resto della curva e spread; credit spread e MOVE; USD/JPY e USD/CNH; indici esteri principali; opzioni (IV, skew); liquidazioni; Coinbase premium; metriche on-chain principali (exchange netflow, MVRV, LTH/STH, Puell Multiple); prezzi energia (ERCOT, gas naturale); titoli dei miner e dei produttori di ASIC.
- **Tier 3, esplorativo**: order book a livello di tick; sentiment social/news via NLP; macro dettagliata dei singoli paesi non-USA; metriche on-chain di nicchia; prezzi dell'elettricità nelle aree di mining non USA.

---

## 11. Domande aperte e prossimi passi

- **Orizzonti:** proposta attuale 4h, 1d, 1w (+ 1M e il ciclo come livello lungo), partendo dal 1d. Da confermare.
- **Target per ciascun orizzonte:** rendimento, direzione (su/giù) o volatilità?
- **Profondità storica:** da quando? (ETF spot solo dal 2024, molte metriche di derivati solo dal 2019–2020.)
- **Nodo Bitcoin proprio** per i dati on-chain: conviene gestirne uno?
- **Fonti dei dati**: prossimo step, mappare per ogni variabile una fonte gratuita o a pagamento, frequenza, storico disponibile e ritardo di pubblicazione.
- **Storage e pipeline**: formato, frequenza di aggiornamento, gestione dei vintage.
