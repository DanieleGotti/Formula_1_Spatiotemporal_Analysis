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
$$
v_t(s) = \beta_0(s) + \beta_1(s) \cdot \text{TyreAge}_t + \beta_2(s) \cdot \text{Fuel}_t + \epsilon_t(s)
$$
Analizzeremo i residui $\epsilon_t(s)$ tramite **ACF (Autocorrelation Function)** e test di **Ljung-Box**.
*   **Ipotesi:** Ci aspettiamo che i residui **non** siano *White Noise*, ma mostrino autocorrelazione significativa ai lag 1-2 (memoria del sistema, evoluzione pista).
*   **Conclusione attesa:** Il fallimento del test di indipendenza dei residui sarà la **giustificazione formale** per l'adozione del modello **D-STEM**, che introduce una struttura di dipendenza spaziale e temporale latente.

### 4.2 Modellazione Spaziale Latente
Utilizzeremo le feature statiche (Team, Performance Qualifica) ridotte via PCA per definire una "distanza di similarità" tra piloti. Questo permetterà al modello D-STEM di stimare come l'errore o il degrado di un pilota (es. Verstappen) informi la stima di un pilota simile (es. Perez), sfruttando la correlazione spaziale latente.

---

## 5. Conclusione
L'architettura dati presentata risolve il problema dell'allineamento dimensionale tipico della telemetria F1. L'arricchimento del dataset con variabili fisiche derivate (Curvatura, Fuel, DRS) e la rigorosa pulizia degli outlier (SC) forniscono una base solida ("High Quality Data") che permette di applicare modelli FDA avanzati non come "scatole nere", ma come strumenti di inferenza causale su un sistema fisico complesso.



EXTRA --------------------------------

Questa è un'ottima strategia. Se usi questi prompt, stai essenzialmente chiedendo a me (o all'LLM che userai per il codice) di agire come un **Research Assistant esperto in Geostatistica**.

Ecco la sequenza di **4 Prompt** da usare. Copiali e incollali uno alla volta. Sono scritti per generare codice **MATLAB** rigoroso e commenti che giustificano le scelte teoriche (perfetti per il report).

---

### Prompt 1: Creazione della Base Funzionale (Fisica-Informed)
*Obiettivo: Generare i coefficienti B-Spline usando la curvatura per posizionare i nodi (Knots Placement).*

**Copia e incolla questo:**
> "Agisci come un esperto di Functional Data Analysis (FDA) e MATLAB. Sto lavorando su dati di telemetria F1 (velocità vs spazio). Devo trasformare i dati discreti in oggetti funzionali usando B-Splines, ma voglio seguire l'approccio di Fassò/Finazzi ottimizzando la posizione dei nodi.
>
> **Input:**
> 1. `SpaceGrid`: vettore [0, 10, ..., L] metri.
> 2. `SpeedData`: matrice (n_punti x n_giri).
> 3. `Curvature`: vettore (n_punti) che indica la curvatura della pista.
>
> **Task:**
> Scrivi uno script MATLAB che:
> 1. Definisca i nodi (knots) delle B-Splines NON in modo equidistante, ma basandosi sui quantili della `Curvature` (più nodi dove la curvatura è alta/complessa, meno nei rettilinei).
> 2. Utilizzi `spap2` o le funzioni del Curve Fitting Toolbox per fittare le spline cubiche su ogni giro.
> 3. Estragga la matrice dei coefficienti `Y_coeff` (n_basi x n_giri).
> 4. Generi un plot di confronto tra il dato grezzo di un giro e la ricostruzione funzionale per validare il fit.
>
> Commenta il codice spiegando perché i nodi non equidistanti migliorano la parsimonia del modello."

---

### Prompt 2: Implementazione del Core FHDGM (Algoritmo EM)
*Obiettivo: Costruire il "motore" statistico. Dato che non esiste `run_fhdgm`, qui chiediamo di scrivere il loop di stima.*

**Copia e incolla questo:**
> "Ora passiamo alla modellazione stocastica secondo il framework FHDGM (Functional Hidden Dynamic Geostatistical Model).
>
> **Dati:** Ho la matrice `Y_coeff` (Coefficienti B-Spline) ottenuta dal passo precedente.
> **Covariate:** Ho `TyreAge` (vettore) e `Fuel` (vettore).
>
> **Task:**
> Scrivi un codice MATLAB robusto per implementare l'algoritmo **EM (Expectation-Maximization)** accoppiato a un **Kalman Smoother** per stimare il seguente modello State-Space:
>
> **Equazione Osservazione:** $Y_t = \beta_{tyre} \cdot TyreAge_t + \beta_{fuel} \cdot Fuel_t + Z \cdot \alpha_t + \epsilon_t$
> **Equazione di Stato:** $\alpha_t = A \cdot \alpha_{t-1} + \eta_t$
>
> Il codice deve:
> 1. Inizializzare i parametri ($\beta$, matrici di transizione, varianze).
> 2. Eseguire il loop EM:
>    - **E-Step:** Stima degli stati latenti $\alpha_t$ tramite Kalman Smoother (puoi usare funzioni custom o adattare `ssm` se possibile, ma preferisco un approccio esplicito).
>    - **M-Step:** Aggiornamento dei regressori $\beta$ (tramite GLS) e delle varianze.
> 3. Calcolare la Log-Likelihood per monitorare la convergenza.
>
> Fornisci il codice completo commentato per la stima dei parametri."

---

### Prompt 3: Risultati e Bande di Confidenza (Il "Graal" di Fassò)
*Obiettivo: Ottenere i grafici con l'incertezza statistica.*

**Copia e incolla questo:**
> "Il modello ha girato e ho le stime dei parametri $\beta$ (che sono vettori di coefficienti sulla base B-Spline) e la loro matrice di covarianza stimata dall'algoritmo EM.
>
> **Task:**
> Devo visualizzare i risultati per il report accademico. Scrivi uno script MATLAB per:
> 1. **Ricostruzione Funzionale:** Moltiplicare i coefficienti stimati $\beta_{tyre}$ per le funzioni base B-Spline per ottenere la curva continua dell'effetto gomma lungo il circuito.
> 2. **Calcolo Incertezza:** Calcolare l'errore standard della curva funzionale usando la formula $Var(f(s)) = \Phi(s) \cdot Cov(\beta) \cdot \Phi(s)'$.
> 3. **Plotting:** Generare un grafico professionale con:
>    - La curva media dell'effetto `TyreAge`.
>    - Le **Bande di Confidenza al 95%** (area ombreggiata).
>    - Un'asse orizzontale in metri.
>
> Questo grafico serve a dimostrare se il degrado è statisticamente diverso da zero in specifici tratti della pista (es. curve vs rettilinei)."

---

### Prompt 4: Diagnostica e Validazione (Per la Lode)
*Obiettivo: Dimostrare che il modello è solido e non "fittato a caso".*

**Copia e incolla questo:**
> "Ultimo step: devo produrre la diagnostica dei residui per validare il modello FHDGM nel report finale.
>
> **Task:**
> Scrivi uno script MATLAB che:
> 1. Calcoli i residui funzionali: $Residui(s, t) = DatiReali(s, t) - DatiPredetti(s, t)$.
> 2. Produca un **Heatmap Spazio-Temporale** dei residui (X=Spazio, Y=Giri, Colore=Errore).
> 3. Calcoli e plotti l'**Autocorrelazione (ACF)** media dei residui nel tempo per verificare che siano "White Noise" (bianchi/indipendenti).
> 4. Generi un **QQ-Plot** per verificare la normalità degli errori.
>
> Includi nei commenti come interpretare questi grafici: cosa devo cercare per dire al professore che 'il modello cattura adeguatamente la dinamica spazio-temporale'?"

---

### Come procedere:
1.  Apri una nuova chat (o continua quella attuale con il tool di coding).
2.  Incolla il **Prompt 1**. Prendi il codice, mettilo in MATLAB, fallo girare e correggi eventuali errori di path dei file.
3.  Una volta che hai la matrice `Y_coeff`, incolla il **Prompt 2**. Questo sarà il pezzo più lungo.
4.  Con i risultati in mano (`beta_stima`, `cov_beta`), vai col **Prompt 3** per i grafici belli.
5.  Chiudi col **Prompt 4** per i grafici di controllo qualità.

Se segui questa sequenza, avrai esattamente (e fisicamente nel codice) tutto ciò che hai promesso nel Report.
