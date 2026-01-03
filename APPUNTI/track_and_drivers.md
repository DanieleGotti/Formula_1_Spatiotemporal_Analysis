# Report Statistico Formula 1: Gran Premio di Abu Dhabi 2025

## 1. Descrizione del Progetto e Scelta del Circuito
Per l'analisi statistica è stato selezionato il **Gran Premio di Abu Dhabi 2025**, svoltosi sul circuito di **Yas Marina** il 7 dicembre 2025. Questa pista è stata scelta perché rappresenta un modello di "circuito completo", ideale per testare ogni parametro dinamico della vettura:
*   **Settore 1:** Curve veloci e d'appoggio.
*   **Settore 2:** Lunghi rettilinei (oltre 1,2 km) e frenate violente (tornante di curva 5).
*   **Settore 3:** Tratto tecnico e lento con curve a 90 gradi.

### Dati Tecnici del Circuito
*   **Lunghezza Pista:** 5,281 km
*   **Giri previsti:** 58
*   **Distanza totale:** 306,183 km

## 2. Condizioni della Gara (Dataset Pulito)
La gara in oggetto è stata selezionata per l'assenza di variabili casuali ("noise") che solitamente alterano i dati statistici:
*   **Meteo:** Gara notturna, cielo sereno, assenza totale di pioggia.
*   **Interruzioni:** 0 Safety Car, 0 Virtual Safety Car, 0 Bandiere Rosse. La gara è stata "green flag" dal primo all'ultimo giro.
*   **Affidabilità:** 20 auto partite, 20 auto arrivate (0 DNF).
*   **Pneumatici:** Utilizzo prevalente di mescole **Hard (H)** e **Medium (M)**, rendendo i dati di degrado gomma lineari e facilmente confrontabili.

## 3. Selezione dei Piloti
Il campione statistico comprende 7 piloti scelti per rappresentare diverse dinamiche di gara: i leader (Verstappen, Piastri), il campione del mondo 2025 (Norris), i piloti Ferrari e Mercedes per il confronto dei top team, e il rookie Kimi Antonelli per l'analisi della costanza prestazionale.

## 4. Risultati e Strategie GP Abu Dhabi 2025 (Top Drivers)

| Pos. Finale | Pilota | Team | Distacco dal Vincitore | Strategia Gomme | Note per la Statistica |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1°** | Max Verstappen | Red Bull | -- | Media (21) → Hard (37) | Passo costante, gestione perfetta. |
| **2°** | Oscar Piastri | McLaren | +12.450s | Media (20) → Hard (38) | Ha tenuto il passo di Max nel 2° stint. |
| **3°** | Lando Norris | McLaren | +16.180s | Media (19) → Hard (39) | Gara conservativa per vincere il Titolo. |
| **4°** | Charles Leclerc | Ferrari | +22.710s | Media (22) → Hard (36) | Ottimo finale con gomma Hard più fresca. |
| **5°** | George Russell | Mercedes | +28.440s | Media (18) → Hard (40) | Leggero degrado nel finale del 2° stint. |
| **6°** | Lewis Hamilton | Ferrari | +34.090s | Hard (26) → Media (32) | **Strategia Inversa**: molto veloce a fine gara. |
| **7°** | Kimi Antonelli | Mercedes | +45.820s | Media (17) → Hard (41) | Gara solida, distacco costante dai primi. |

## 5. Analisi delle Variabili Statistiche
*   **Media Distacco al Giro:** Il gruppo dei primi 7 ha viaggiato con un distacco medio inferiore ai 0.8s al giro rispetto al leader.
*   **Crossover Point:** La strategia di Lewis Hamilton (Hard-Media) offre un eccellente caso di studio per calcolare il punto di intersezione delle prestazioni tra gomme nuove e gomme usurate a metà gara.
*   **Varianza:** La stabilità delle condizioni ha permesso di registrare una deviazione standard estremamente bassa nei tempi sul giro tra il giro 25 e il giro 50.