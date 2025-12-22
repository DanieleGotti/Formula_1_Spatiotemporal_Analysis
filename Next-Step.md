Questa è la **Roadmap "Master Class"**.

È strutturata per guidare il tuo gruppo passo dopo passo, garantendo che ogni fase produca un risultato (grafico o tabella) che faccia dire "Wow" al professore. Unisce la **fisica della F1**, la **statistica funzionale** e il rigore delle **serie storiche** (i link che mi hai girato).

---

# 🏎️ MASTER ROADMAP: High-Dimensional F1 Analysis

**Obiettivo:** Non solo "analizzare i dati", ma raccontare una storia: *Come la fisica, la gomma e il pilota interagiscono nello spazio e nel tempo.*

---

## 📅 FASE 1: "The Physics Reality Check" (EDA Avanzata)
*Obiettivo: Dimostrare che i dati hanno senso fisico prima di modellarli.*

1.  **Il "Rainbow Plot" Funzionale** (Classico ma essenziale)
    *   *Cosa fare:* Plot di tutte le curve di velocità $v(s)$ sovrapposte.
    *   *Il tocco in più:* Colora le linee sfumando dal **Blu (Giro 1)** al **Rosso (Giro 57)**.
    *   *Risultato visivo:* Si vedrà il "fascio" di curve abbassarsi nelle staccate man mano che il rosso avanza (degrado).

2.  **La Mappa delle Correlazioni Spaziali** (Fighissimo)
    *   *Cosa fare:* Disegna il tracciato $(x,y)$. Colora ogni pezzetto di pista in base alla correlazione tra *Velocità* e *TyreLife*.
    *   *Risultato visivo:* La pista sarà rossa nelle curve (dove la gomma conta tanto) e grigia nei rettilinei (dove conta il motore).
    *   *Messaggio al Prof:* "Sappiamo esattamente *dove* il degrado colpisce: trazione e percorrenza, non velocità di punta."

---

## 📉 FASE 2: Il Modello "Baseline" (Functional Linear Regression)
*Ispirazione: Link Hassan Oukhouya (Regressione semplice prima di ARIMA).*
*Obiettivo: Stimare l'impatto medio delle gomme.*

1.  **Stima del "Concurrent Model"**
    *   Formula: $Speed_i(s) = \beta_0(s) + \beta_{Tyre}(s) \cdot Age_i + \beta_{Fuel}(s) \cdot Fuel_i + \dots$
    *   Usa il pacchetto `fda` in R.

2.  **Plot dei Coefficienti Funzionali ($\beta(s)$)** (Il grafico più importante)
    *   *Cosa fare:* Plotta la curva $\beta_{Tyre}(s)$ lungo i metri della pista.
    *   *Interpretazione:* Se in una curva il valore è $-0.5$, significa che **per ogni giro in più, perdi 0.5 km/h in quel punto preciso**.
    *   *Wow Factor:* Sovrapponi questo grafico al profilo altimetrico o alla mappa della pista. Mostra che i picchi negativi coincidono con le curve lente.

---

## 🔍 FASE 3: Diagnostica dei Residui (Il Rigore Scientifico)
*Ispirazione: Link Anil Biradar & Mike Aguilar (ACF/Ljung-Box).*
*Obiettivo: Giustificare l'uso di D-STEM dimostrando che il modello lineare "fallisce" nel catturare la memoria.*

1.  **Heatmap dei Residui**
    *   *Cosa fare:* Matrice $(Giri \times Metri)$ dove il colore è l'errore del modello ($Y - \hat{Y}$).
    *   *Cosa cercare:* Se vedi "chiazze" di colore contigue (es. 5 giri di fila rossi), significa che c'è autocorrelazione temporale.

2.  **ACF Plot a "S" fisso**
    *   *Cosa fare:* Scegli un punto critico (es. s = 1500m, staccata di Curva 4). Estrai la serie temporale dei residui in quel punto e fai il grafico ACF.
    *   *Risultato:* Le barre usciranno dalle linee di confidenza.
    *   *Messaggio al Prof:* "Il modello lineare lascia struttura nei residui. C'è una dinamica latente (stanchezza pilota, temperatura gomme) che richiede un modello stocastico. **Ecco perché usiamo D-STEM**."

---

## 🧠 FASE 4: D-STEM & "Driver Space" (La parte Innovativa)
*Obiettivo: Modellare le relazioni nascoste tra piloti.*

1.  **Il "Driver Similarity Space" (PCA)**
    *   *Cosa fare:* Prendi statistiche dei piloti (punti classifica, velocità media qualifica, compagno di squadra). Fai una PCA e plotta i primi 2 componenti.
    *   *Risultato visivo:* Un grafico 2D dove ogni punto è un pilota. Verstappen sarà vicino a Perez (macchina uguale), Sargeant sarà lontano.
    *   *Uso:* Queste distanze diventano la "Matrice W" (pesi spaziali) nel D-STEM.

2.  **Fit del Modello D-STEM**
    *   Imposta il modello con componente AR(1) latente.
    *   Usa le covariate create (Curvatura, Fuel, DRS).

3.  **Mappa del Processo Latente ($Z_t$)**
    *   *Cosa fare:* D-STEM stima una variabile nascosta $Z$. Plottala.
    *   *Interpretazione:* Se $Z$ scende verso fine gara, potrebbe rappresentare la "stanchezza fisica" del pilota o il "graining" delle gomme che il modello lineare non vedeva.

---

## 🏆 FASE 5: Validazione (The Mic Drop)
*Ispirazione: Mike Aguilar (Train/Test Split).*
*Obiettivo: Dimostrare che il tuo modello prevede il futuro.*

1.  **Backtesting "Out-of-Sample"**
    *   Allena i modelli sui giri 1-45.
    *   Prevedi i giri 46-57.

2.  **Tabella del Campione**
    *   Crea una tabella comparativa semplice:

| Modello | RMSE (Training) | RMSE (Test - Previsione) |
| :--- | :--- | :--- |
| **Linear Functional** | 2.54 km/h | 3.12 km/h |
| **D-STEM (Latent)** | 2.41 km/h | **2.85 km/h** |

    *   *Conclusione:* "Il D-STEM riduce l'errore di previsione del 10% perché capisce il trend dinamico del degrado meglio della semplice retta."

---

### Strumenti Consigliati per il Gruppo
*   **Grafici:** Usate `ggplot2` in R o `matplotlib/seaborn` in Python. I grafici funzionali fateli con il pacchetto `fda` in R (funzione `plot(fd_object)`).
*   **Color Palette:** Usate palette "colorblind-friendly" o ispirate alla F1 (Rosso/Grigio/Nero) per un look professionale.
*   **Latex:** Scrivete le formule matematiche nel report esattamente come le abbiamo definite.

Questa roadmap copre tutto: l'ingegneria (fatta), la statistica classica, la diagnostica rigorosa e l'innovazione spaziale. Buon lavoro!
