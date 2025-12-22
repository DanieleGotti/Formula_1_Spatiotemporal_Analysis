Ecco il report tecnico definitivo. È stato redatto adottando uno stile accademico/ingegneristico rigoroso, strutturato come i *Case Studies* di serie storiche che hai condiviso (Anil Biradar, Mike Aguilar), ma adattato alla complessità dei dati funzionali (FDA).

Questo documento serve a dimostrare al professore non solo che avete "fatto girare il codice", ma che avete **dominio completo** della teoria statistica e fisica sottostante.

---

# REPORT TECNICO: Data Engineering & Functional Modeling Framework for F1 Telemetry Analysis

**Progetto:** Statistics for High Dimensional Data  
**Oggetto:** Analisi Spazio-Temporale della Degradazione Gomme in Formula 1 (D-STEM Framework)  
**Data:** Dicembre 2025

---

## 1. Abstract e Definizione del Problema
L'obiettivo del progetto è modellare la curva di velocità $v_{t,k}(s)$ (dove $t$ è il tempo/giro, $k$ il pilota, $s$ la posizione spaziale) per quantificare il decadimento prestazionale dovuto all'usura degli pneumatici e isolare l'effetto "pilota".
La sfida principale risiede nella natura intrinseca dei dati di telemetria (FastF1): si tratta di serie temporali campionate a frequenza fissa (Hz), che risultano **disallineate nel dominio spaziale**. Un confronto diretto $v_{t_1}(time)$ vs $v_{t_2}(time)$ violerebbe l'assunto di comparabilità statistica, in quanto al medesimo istante temporale corrispondono posizioni diverse lungo il circuito a seconda della velocità di percorrenza.

Il presente report documenta la pipeline di **Data Engineering** e **Feature Extraction** sviluppata per trasformare i dati grezzi in **Oggetti Funzionali** definiti su un dominio $s \in [0, L]$ comune, pronti per l'analisi D-STEM (Distributed Spatio-Temporal Expectation-Maximization).

---

## 2. Pipeline di Data Engineering: Dalla Serie Storica al Dato Funzionale

### 2.1 Selezione e Filtraggio del Campione (Stationarity Enforcement)
Per garantire la robustezza delle stime statistiche, è necessario operare su un processo il più possibile stazionario (al netto del trend di degrado).
*   **Safety Car (SC/VSC):** Sono stati rimossi tutti i giri con `TrackStatus != 1`. L'introduzione di regimi di SC rappresenta uno shock esogeno che altera la temperatura delle gomme e il consumo, introducendo *outlier strutturali* che invaliderebbero l'ipotesi di linearità del degrado nel modello D-STEM.
*   **Giri In/Out:** Rimossi per garantire che ogni realizzazione funzionale copra l'intero dominio $s$.

### 2.2 Registrazione Spaziale (Spatial Registration)
Questa è la scelta architetturale più critica. Abbiamo abbandonato il dominio temporale per passare al dominio spaziale.
*   **Metodologia:** Creazione di una griglia fissa $\mathbf{s} = [0, 10, 20, \dots, L]$ metri.
*   **Algoritmo:** Per ogni giro $t$, le variabili di stato (Velocità, Posizione X/Y) sono state rimappate sulla griglia $\mathbf{s}$ tramite **Interpolazione Lineare**.
*   **Giustificazione:** L'interpolazione lineare su una griglia fitta ($10m$) preserva i gradienti locali di velocità (accelerazione/frenata) senza introdurre le oscillazioni spurie (fenomeno di Runge) tipiche delle interpolazioni polinomiali di alto ordine su dati rumorosi.

---

## 3. Feature Engineering Fisica e Modellistica

Per ridurre la varianza residua ($\epsilon$) e migliorare il fit del modello D-STEM, abbiamo introdotto covariate esogene basate sulla fisica del veicolo.

### 3.1 Curvatura Geometrica ($\kappa$) - Regressore Spaziale
La velocità in curva è limitata dall'aderenza laterale secondo la legge $v_{max} \approx \sqrt{\mu g R}$. Pertanto, la curvatura è il predittore principale della velocità.
Invece di usare dati rumorosi, abbiamo calcolato la curvatura analitica $\kappa(s)$ sulla traiettoria interpolata:

$$ \kappa(s) = \frac{|x'(s)y''(s) - y'(s)x''(s)|}{(x'(s)^2 + y'(s)^2)^{3/2}} $$

*   **Implementazione:** Calcolo tramite gradienti finiti (`np.gradient`) su coordinate spazializzate.
*   **Utilità:** Funziona come "regressore invariante" nel modello FDA, spiegando la variabilità della velocità dovuta al layout della pista.
Questa è un'ottima domanda, perché spesso in statistica usiamo numeri senza capire il loro senso fisico.

La **Curvatura ($\kappa$)** ha un significato geometrico molto preciso: è l'inverso del Raggio di Curvatura ($R$).

$$ \kappa = \frac{1}{R} $$

Dove $R$ è il raggio del cerchio che approssima meglio la curva in quel punto.

### 1. Come leggere i numeri (La Scala)

L'unità di misura è $m^{-1}$ (metri alla meno uno). Ecco una "tabella di traduzione" per la F1:

DA RICORDARE-- > abbiamo poi aggiunto le dummy solo per i grafici, nel D-stem ha senso tenere il valore numerico

| Valore Curvatura ($\kappa$) | Raggio ($R = 1/\kappa$) | Interpretazione F1 | Esempio Reale |
| :--- | :--- | :--- | :--- |
| **0.00002** | 50.000 metri | **Rettilineo** | Main Straight (Bahrain) |
| **0.002 - 0.005** | 200 - 500 metri | **Curvone Veloce** | Curva 12 (Bahrain), Parabolica (Monza) |
| **0.01 - 0.02** | 50 - 100 metri | **Curva Media** | Curve 6-7 (Bahrain) |
| **0.03 - 0.05** | 20 - 33 metri | **Curva Lenta / 90°** | Curva 1 (Bahrain) |
| **> 0.08** | < 12 metri | **Tornantino (Hairpin)** | Loews (Monaco), Curva 10 (Bahrain) |

**Implicazione per il Modello D-STEM:**
L'introduzione di $\kappa(s)$ permette al modello di distinguere se una bassa velocità in un punto $s$ è dovuta a *degrado gomme* (residuo anomalo rispetto al $\kappa$ locale) o semplicemente alla *geometria della pista* (bassa velocità predetta da un alto $\kappa$). Senza questo regressore, il modello interpreterebbe erroneamente le curve lente come zone di scarsa performance del pilota.

### 3.2 Aerodinamica Attiva (DRS) - Variabile Binaria
L'attivazione del DRS (Drag Reduction System) riduce il coefficiente di resistenza ($C_d$), incrementando la velocità non per "bravura" del pilota ma per configurazione meccanica.
*   **Scelta Architetturale:** Interpolazione **Nearest Neighbor**.
*   **Motivazione:** Il DRS è un sistema *on/off* (funzione a gradino). Un'interpolazione lineare avrebbe creato valori spuri (es. 0.5) nei punti di transizione, privi di significato fisico. La variabile risultante è strettamente binaria $\{0, 1\}$.

### 3.3 Dinamica del Carburante (Fuel Load) - Regressore Deterministico
L'alleggerimento della vettura (~110kg $\to$ 0kg) migliora i tempi sul giro (~0.03s/giro), mascherando il vero degrado gomme.
*   **Modellazione:** $Fuel(t) = Fuel_{start} - \alpha \cdot t$.
*   **Utilità:** Inserendo questa variabile nel modello multivariato, forziamo il coefficiente delle gomme ($\beta_{tyre}$) a catturare esclusivamente il degrado chimico/fisico della mescola, depurandolo dall'effetto peso.

### 3.4 Gestione Categorica delle Mescole (Robust Encoding)
Per gestire le variabili categoriche (Soft, Medium, Hard, Wet) abbiamo implementato un sistema di **One-Hot Encoding** universale.
*   **Robustezza:** Il codice verifica dinamicamente quali mescole sono state usate. Se una mescola (es. Medium in Bahrain 2024) non è presente, viene generata una colonna di zeri.
*   **Dummy Variable Trap:** La struttura dati è pronta per la regressione. Sappiamo che in fase di fitting dovremo escludere una categoria (es. Soft) per usarla come *baseline* ed evitare la multicollinearità perfetta ($\sum P_i = 1$).

---

## 4. Analisi Metodologica e Prossimi Passi (Roadmap Scientifica)

Seguendo l'approccio rigoroso dei *Time Series Case Studies* (cfr. Box-Jenkins methodology), il dataset generato abilita la seguente strategia di analisi:

### 4.1 Validazione della Stazionarietà e Baseline Model
Prima di applicare D-STEM, stimeremo un modello di **Regressione Funzionale Lineare (Concurrent Model)**:
$$ v_t(s) = \beta_0(s) + \beta_1(s) \cdot \text{TyreAge}_t + \beta_2(s) \cdot \text{Fuel}_t + \epsilon_t(s) $$
Analizzeremo i residui $\epsilon_t(s)$ tramite **ACF (Autocorrelation Function)** e test di **Ljung-Box**.
*   **Ipotesi:** Ci aspettiamo che i residui **non** siano *White Noise*, ma mostrino autocorrelazione significativa ai lag 1-2 (memoria del sistema, evoluzione pista).
*   **Conclusione attesa:** Il fallimento del test di indipendenza dei residui sarà la **giustificazione formale** per l'adozione del modello **D-STEM**, che introduce una struttura di dipendenza spaziale e temporale latente.

### 4.2 Modellazione Spaziale Latente
Utilizzeremo le feature statiche (Team, Performance Qualifica) ridotte via PCA per definire una "distanza di similarità" tra piloti. Questo permetterà al modello D-STEM di stimare come l'errore o il degrado di un pilota (es. Verstappen) informi la stima di un pilota simile (es. Perez), sfruttando la correlazione spaziale latente.

---

## 5. Conclusione
L'architettura dati presentata risolve il problema dell'allineamento dimensionale tipico della telemetria F1. L'arricchimento del dataset con variabili fisiche derivate (Curvatura, Fuel, DRS) e la rigorosa pulizia degli outlier (SC) forniscono una base solida ("High Quality Data") che permette di applicare modelli FDA avanzati non come "scatole nere", ma come strumenti di inferenza causale su un sistema fisico complesso.
