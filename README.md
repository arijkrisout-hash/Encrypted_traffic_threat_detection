# Encrypted Traffic Threat Detection

Machine learning on **flow metadata** to detect attacks hidden in **encrypted network traffic**, without ever decrypting a payload.

> Security of Systems and Networks – course project

The notebook has two parts:

1. **Real data:** train and compare classifiers on the CIC-IDS-Collection dataset, restricted to encrypted flows, with a focus on rare and stealthy attacks (Infiltration, Web attacks, Port scans).
2. **Traffic generation and feature extraction:** build a synthetic labelled capture (`.pcap` + labels), extract flow features with NFStream and with two emulated feature sets (PcapPlusPlus-style and Cisco Joy-style), compare them, and prepare a real-time detector.

---

## Table of Contents

- [Dataset](#dataset)
- [Pipeline](#pipeline)
- [Results](#results)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Notes and Limitations](#notes-and-limitations)
- [Tech Stack](#tech-stack)

---

## Dataset

**CIC-IDS-Collection** ([Kaggle](https://www.kaggle.com/datasets/dhoogla/cicidscollection)), a harmonized merge of four Canadian Institute for Cybersecurity benchmarks: CIC-IDS2017, CIC-DoS2017, CSE-CIC-IDS2018 and CIC-DDoS2019.

- 9,167,581 flows, 59 features, stored as a Parquet file
- Pre-cleaned (no duplicates, no missing values)
- Two label levels: `Label` (precise attack name) and `ClassLabel` (macro class)
- Strong class imbalance between benign traffic and rare attacks, as in real networks

The dataset is **not included** in this repository. Download it from the link above.

### Isolating encrypted traffic

Port numbers (443, 22, ...) are not available in the dataset, so encrypted sessions are isolated with a behavioral filter instead:

```python
(Init Fwd Win Bytes > 0) & (Init Bwd Win Bytes > 0) & (Fwd Seg Size Min >= 20)
```

This keeps **4,758,611 flows**, so the classifier cannot rely on plain-text protocol artifacts.

| Class        | Flows (encrypted subset) |
|--------------|--------------------------|
| Benign       | 3,761,442 |
| DDoS         | 535,204 |
| DoS          | 170,383 |
| Botnet       | 143,932 |
| Bruteforce   | 101,211 |
| Infiltration | 42,663 |
| Webattack    | 2,567 |
| Portscan     | 1,209 |

## Pipeline

```
CIC-IDS-Collection ──► Encrypted-flow filter ──► 70/30 stratified split ──► StandardScaler
                                                          │
              ┌───────────────────────────────────────────┤
              ▼                                           ▼
   Random Forest baseline                    Imbalance handling
   + feature importance                      • class_weight='balanced'
                                             • SMOTE
                                             • 90% benign under-sampling + class weights
                                             • MLP on the reduced set

Synthetic traffic (Scapy) ──► simulation_trafic.pcap + simulation_labels.csv
        │
        ├─► NFStream (real flow extraction, statistical analysis)
        ├─► PcapPlusPlus-style features (emulated with Scapy)
        └─► Cisco Joy-style features (emulated with Scapy)
                │
                ▼
   Join with labels ──► Random Forest (multi-class and binary Benign/Attack)
                │
                ▼
   Saved binary model ──► Real-time detection with NFStreamer
```

## Results

### Part 1 – Real data (test set: 1,427,584 flows)

| Strategy | Accuracy | Infiltration precision / recall |
|---|---|---|
| Random Forest + `class_weight='balanced'` | 82.51% | 0.05 / 0.93 |
| Random Forest + SMOTE | 99% | 0.60 / 0.03 |
| 10% benign sampling + `class_weight='balanced'` | 81.52% | 0.04 / 0.94 |
| MLP (50, 25) on the reduced set | 98.45% | 0.17 / 0.11 |

Main observations:

- Volumetric attacks (DDoS, DoS, Botnet, Bruteforce) are detected almost perfectly in every setup.
- The most important features are `Bwd Packet Length Std` (6.78%) and `Bwd Packet Length Mean` (5.91%): the model relies on backward packet-size structure, not on initial TCP window sizes.
- **Infiltration is the hard case.** SMOTE gives near-perfect recall on Portscan (0.99) but does not help Infiltration. Cost-sensitive learning plus benign under-sampling raises Infiltration recall to 0.94, at the price of very low precision (many false alarms). This trade-off can be acceptable in a SOC that prioritizes early containment.
- Without timestamps in the dataset, temporal behavior cannot be modeled, which limits what flow statistics alone can separate.

### Part 2 – Synthetic traffic (100,000 generated flows)

| Feature source | Task | Accuracy |
|---|---|---|
| NFStream (68 numeric features) | Multi-class | 91.43% |
| NFStream | Binary (Benign vs Attack) | 97.81% |
| PcapPlusPlus-style (2 features) | Binary | 78.39% |
| Cisco Joy-style (2 features) | Binary | 79.45% (attack recall 0.05) |

Richer, flow-level features (NFStream) clearly outperform the minimal header-level feature sets. Re-balancing the Cisco Joy-style data (under-sampling or class weights) did not change the outcome.

## Project Structure

```
.
├── Encrypted_Traffic_Threat_Detection.ipynb   # Full pipeline (both parts)
├── README.md
├── .gitignore
├── data/                                      # Put cic-collection.parquet here (not versioned)
└── output/                                    # Created by the notebook (not versioned)
```

Files generated in `output/` when running the notebook:

```
encrypted_clean.parquet
simulation_traffic.pcap      simulation_labels.csv
train_nfstream.csv           train_pcapplusplus.csv       train_cisco_joy.csv
random_forest_binary.joblib  standard_scaler.joblib
matrix_*.png                 (normalized confusion matrices)
```

## Getting Started

### 1. Install dependencies

```bash
pip install pandas numpy scikit-learn imbalanced-learn matplotlib seaborn \
            joblib fastparquet scapy nfstream jupyter
```

### 2. Get the data

Download `cic-collection.parquet` from [Kaggle](https://www.kaggle.com/datasets/dhoogla/cicidscollection) and place it in `data/`. The location is set by a single variable, `DATASET_PATH`, in the first code cell of the notebook.

### 3. Run

```bash
jupyter notebook Encrypted_Traffic_Threat_Detection.ipynb
```

Run the cells in order ("Run All" works). The first cell installs the dependencies with `%pip`. The full dataset is large (9M+ rows): SMOTE in particular needs a lot of RAM, and the heavy cells can take several minutes.

Everything after the traffic-generation step (Part 2) only depends on the class proportions of the dataset, so it runs in a few minutes.

### Live detection

The last cell scores flows with the saved binary model. By default it **replays the synthetic capture** (`output/simulation_traffic.pcap`), so it works anywhere. To monitor live traffic, set `SOURCE` to a network interface name (e.g. `"en0"` on macOS, `"eth0"` on Linux) and run Jupyter with enough privileges. If the source cannot be opened, the cell prints an explanation instead of failing.

## Notes and Limitations

- **Part 2 uses synthetic data.** Flows are forged with Scapy (e.g. fixed ports and simple size/duration profiles per class), so the accuracies above show how the feature sets compare on this simulation, not real-world detection performance. In particular, NFStream's top features (`application_is_guessed`, `application_confidence`) carry most of the importance, which likely reflects how the traffic was generated.
- **Cisco Joy and PcapPlusPlus are emulated, not executed.** Their feature sets are reproduced with Scapy to avoid cross-platform build issues. The cells that call the real tools are kept as commented-out reference code.
- **Live capture** is optional and disabled by default (the detection cell replays the synthetic pcap). Since the model was trained on that same synthetic data, the replay demonstrates the scoring pipeline, not generalization to real traffic.
- The real-data results come from a single stratified split (`random_state=42`) with no cross-validation.

## Tech Stack

- Python 3
- scikit-learn, imbalanced-learn
- pandas, NumPy
- NFStream, Scapy
- Matplotlib, Seaborn
- Jupyter / Google Colab

## Source

CIC-IDS-Collection: https://www.kaggle.com/datasets/dhoogla/cicidscollection
