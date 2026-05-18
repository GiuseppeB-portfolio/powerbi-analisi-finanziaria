# Analisi vendite e performance commerciali — Power BI

Report interattivo di Power BI per il monitoraggio delle performance commerciali, il controllo degli obiettivi economici e la simulazione dell'impatto di variazioni di prezzo. Sviluppato su due anni di dati di vendita (gennaio 2022 – dicembre 2023).

![tool](https://img.shields.io/badge/Power%20BI-F2C811?logo=powerbi&logoColor=black)
![language](https://img.shields.io/badge/DAX-217346)
![period](https://img.shields.io/badge/periodo-2022--2023-blue)
![status](https://img.shields.io/badge/status-completato-success)

---

## 🎯 Obiettivo

Costruire un'unica vista, navigabile e auto-esplicativa, che risponda a tre domande di business:

1. **Stiamo raggiungendo gli obiettivi economici** (budget e target) a livello aggregato e per area geografica?
2. **Quanto stiamo guadagnando davvero**, misurando il margine lordo reale a partire da costi e prezzi unitari?
3. **Cosa succede se aumentiamo i prezzi del 15%?** — simulazione what-if con impatto su utile e ranking prodotti.

Il report è progettato per supportare riunioni commerciali periodiche, in cui management e responsabili di area possono filtrare per agente e periodo e leggere lo stato delle vendite in pochi secondi.

---

## 🏗️ Modello dati

Il dataset di partenza era composto da **8 file CSV** con granularità eterogenee. Il modello finale è stato strutturato secondo uno **schema a stella** classico, con una tabella dei fatti centrale e dimensioni esplicite.

```
                  ┌─────────────────┐
                  │   Calendario    │
                  │  (dim. tempo)   │
                  └────────┬────────┘
                           │
                           │ 1 : *
                           ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│  prodotti    │───►│ vendite_avanzate │◄───│   agenti     │
│  (dim.)      │ *  │     (fatti)      │  * │   (dim.)     │
└──────────────┘    └────────┬─────────┘    └──────────────┘
                             │
                             │ *
                             ▼
                  ┌──────────────────────┐
                  │  target_settimanale  │
                  │       (dim.)         │
                  └──────────────────────┘
```

### Decisioni di modellazione

- **Tabella Calendario dedicata.** Creata da zero, con anno, mese, settimana e attributi derivati. È prerequisito per misure temporali corrette (CAGR, confronti anno su anno, scostamento settimanale) e per evitare il fallback automatico di Power BI sulle gerarchie di data.
- **Relazioni 1:\*** tra dimensioni e tabella dei fatti, su chiavi univoche (`ID Prodotto`, `ID Agente`, chiave temporale). Garantisce filtri coerenti in tutte le visualizzazioni.
- **Pulizia dei valori `NULL` sulla colonna `Area`** della tabella vendite: filtrati a monte in fase di caricamento Power Query per evitare righe ambigue nei breakdown geografici.
- **Estensione manuale del Calendario** fino a coprire l'intero 2023, dopo aver scoperto che la copertura originale interrompeva il calcolo del CAGR.
- **Riconciliazione di granularità diverse** (giorni dei dati di vendita vs settimane dei target) gestita con misure DAX dedicate invece che con aggregazioni a monte, preservando la flessibilità di drill-down.

---

## 📐 Misure DAX principali

Sono state implementate **10 misure** che rappresentano la spina dorsale analitica del report. Quelle chiave:

### Margine Lordo Reale
Differenza tra prezzo di vendita e costo unitario, moltiplicata per la quantità venduta. Fornisce la marginalità effettiva contro i ricavi nominali.

### % Raggiungimento Budget per Area
Rapporto tra vendite reali e valori attesi mensili, segmentato geograficamente. Alimenta i KPI di overview e il confronto colonnare per area.

### Scostamento Settimanale Target per Agente
Gap tra target settimanale e venduto effettivo, calcolato a granularità agente × settimana. Base della pagina "Focus Agenti".

### Margine Scenario Prezzi +15%
Misura **what-if** che simula l'incremento di prezzo del 15% e ricalcola il margine, mantenendo invariati costi e quantità. Permette di valutare l'impatto sull'utile prima di proporre manovre commerciali.

### CAGR (Compound Annual Growth Rate)
Tasso di crescita annuo composto tra 2022 e 2023.

```dax
CAGR = 
VAR AnnoIniziale = MIN(Calendario[Anno])
VAR AnnoFinale = MAX(Calendario[Anno])
VAR VenditeIniziali = CALCULATE([Vendite per Area], Calendario[Anno] = AnnoIniziale)
VAR VenditeFinali  = CALCULATE([Vendite per Area], Calendario[Anno] = AnnoFinale)
VAR Anni = AnnoFinale - AnnoIniziale
RETURN
    IF(
        Anni > 0,
        (VenditeFinali / VenditeIniziali) ^ (1 / Anni) - 1,
        BLANK()
    )
```

> La formula è scritta con variabili (`VAR ... RETURN`) per leggibilità e singola valutazione di ogni componente, e usa `BLANK()` come fallback quando il periodo è inferiore all'anno — evita errori a runtime nei contesti di filtro che restringono il calendario a un singolo anno.

---

## 🖥️ Struttura del report

Il report è organizzato in **tre pagine collegate** da un page navigator, in modo che la fruizione segua un flusso narrativo dal generale al dettaglio.

### 🏠 Home
Pagina introduttiva con la sintesi del progetto e un menu di navigazione. Serve a chi apre il report per la prima volta per orientarsi tra le altre pagine.

### 📊 Overview Vendite e Obiettivi
Cruscotto principale, pensato per riunioni di stato.
- **4 card KPI**: vendite totali, budget, % raggiungimento, CAGR
- **Area chart**: andamento temporale delle vendite per area
- **Clustered column chart**: vendite reali vs budget per area
- Spazi per insight testuali a corredo dei numeri

### 👤 Focus Agenti + Simulazione Scenario Prezzi
Pagina di approfondimento operativo e simulazione.
- **3 card KPI** sintetiche su agente / margine / scostamento
- **2 slicer** per filtrare per agente e periodo
- **Line + column combo chart**: andamento settimanale del venduto contro il target
- **Scatter chart**: dispersione prodotti con confronto Margine Reale vs Margine Scenario +15%, per identificare i prodotti più sensibili alla leva prezzo

---

## 💡 Cosa permette di rispondere

Il report risponde direttamente a domande operative come:

- *In quale area stiamo sotto budget e di quanto?*
- *Qual è il tasso di crescita reale tra i due anni, depurato da effetti mensili?*
- *Quale agente ha lo scostamento settimanale più alto e su quali settimane si concentra?*
- *Se aumentiamo i prezzi del 15% in modo uniforme, di quanto cresce il margine complessivo e su quali prodotti il guadagno è massimo rispetto al rischio di perdita volumi?*

---

## 📁 File del repository

```
.
├── Analisi_finanziaria.pbix     → file Power BI con modello, misure e visualizzazioni
├── Nota_esplicativa.pdf         → documento tecnico con scelte progettuali e criticità
└── README.md                    → questo file
```

---

## 🚀 Come consultare il report

1. Scarica il file `Analisi_finanziaria.pbix`
2. Aprilo con **Microsoft Power BI Desktop** (download gratuito da [powerbi.microsoft.com](https://powerbi.microsoft.com/desktop/))
3. La pagina Home offre la navigazione tra le viste; gli slicer della seconda e terza pagina permettono di filtrare interattivamente per agente e periodo

> ⚠️ Il file `.pbix` contiene un'istantanea dei dati incorporati. Per aggiornare la sorgente sarebbe necessario ricollegare le otto tabelle originali in Power Query.

---

## 🛠️ Stack tecnico

- **Microsoft Power BI Desktop** — modellazione, DAX, visualizzazione
- **Power Query (M)** — pulizia ed estensione del Calendario
- **DAX** — 10 misure dedicate, con uso intensivo di `CALCULATE`, `VAR ... RETURN`, funzioni temporali
- **Schema a stella** — fatti + 4 dimensioni

---

*Progetto sviluppato in autonomia su dati commerciali biennali, con focus su modellazione robusta, misure documentate e supporto alla decisione manageriale.*
