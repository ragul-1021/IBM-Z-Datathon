# CascadeGuard AI

**Predict payment-service failures before they spread.**
An IBM Z Datathon project, built in Python and Jupyter on IBM LinuxONE.

CascadeGuard AI watches live service metrics, detects that a service is starting to fail, ranks which service is the likely source, and traces which dependent services (including the payment flow) are at risk. It then gives a human operator an explainable alert with a suggested action. It never changes any system on its own.

---

## Table of Contents
1. [Problem](#problem)
2. [Our Solution](#our-solution)
3. [System Flowchart](#system-flowchart)
4. [Dataset](#dataset)
5. [How It Works](#how-it-works)
6. [Results](#results)
7. [Example Alert](#example-alert)
8. [Tech Stack](#tech-stack)
9. [Getting Started](#getting-started)
10. [Limitations](#limitations)
11. [Future Work](#future-work)
12. [Acknowledgements](#acknowledgements)
13. [Demo Video](#demo-video)

---

## Problem
In a payment system, one slow service, such as a balance check or the payment service itself, quietly slows every service that depends on it. Conventional monitoring fires only after an error threshold is crossed, so customers are already seeing failed payments. Outages like this hurt people who depend on timely payments: wages, benefits, medical bills and small merchants.

## Our Solution
Fraud models ask whether a payment is suspicious. **CascadeGuard asks whether the payment system itself is failing.**

- **Detect early:** spot multi-metric drift (CPU, memory, disk I/O, sockets, latency, errors) as a fault develops.
- **Locate the source:** rank which service is the likely origin of the incident.
- **Trace the impact:** follow the call graph to the callers and payment path at risk.
- **Explain and recommend:** show the evidence, confidence and a suggested action for human review.

## System Flowchart

<img width="2400" height="1344" alt="image" src="https://github.com/user-attachments/assets/df4e8068-77ad-410d-8f26-279a327a95c2" />


## Dataset
**RCAEval**, a benchmark of real fault-injection experiments on **Online Boutique**, a demo e-commerce microservice application that includes checkout and payment services.

| Item | Detail |
|---|---|
| Cases used | 125 (RE1-OB) |
| Fault types | `cpu`, `mem`, `disk`, `delay`, `loss` |
| Root-cause services | 5 services, 5 repetitions per fault |
| Sampling | 1 second per metric |
| Labels | Ground-truth root-cause service, fault type and injection time |

Faults are injected on a lab deployment, not production bank traffic.

## How It Works
1. **Metric replay.** Per-second metrics are replayed and grouped into 5-second batches, like a live stream.
2. **Feature engineering.** Each metric becomes a z-score against that case's healthy baseline, smoothed with a backward-looking rolling mean (no future data). Each service is also compared with the services it calls.
3. **Two detection layers.**
   - **Isolation Forest** is trained only on healthy data and gives an anomaly score.
   - **Gradient Boosting** predicts whether a service is the active source of an incident and outputs a risk score.
4. **Alert rule.** An alert fires when risk is at least 0.6 for 3 consecutive batches (15 seconds).
5. **Call-graph impact.** A rule-based step traces the suspected source up the Online Boutique call graph to the callers and payment path at risk.
6. **Explainable alert.** The operator sees the source, evidence, impact, suggested action and confidence.

Training uses repetitions 1 to 3 of each experiment and testing uses repetitions 4 and 5, so test cases are runs the models never saw. The fault type is never given to the models.

## Results
Evaluated on 50 unseen test cases and compared against two conventional monitors:
the **N-sigma monitor** (any metric drifts 6 or more standard deviations) and an **SLO latency monitor** (frontend latency above 2 times baseline).

| Metric | CascadeGuard AI | N-sigma monitor | SLO latency monitor |
|---|---|---|---|
| Incidents detected | 47 / 50 | 49 / 50 | 14 / 50 |
| False-alarm cases (before the fault) | **2 / 50** | 18 / 50 | 3 / 50 |
| Source service correct (top-1) | **47 / 47** | 39 / 49 | 7 / 14 |
| Median time-to-detect | 34 s | 29 s | 31.5 s |

Other measurements from our run:
- Row-level test scores (per service and batch): precision 0.97, recall 0.93.
- Call-graph impact check: precision 0.57, recall 0.47 against the services whose latency actually rose.
- Scoring time: about 12 ms per 5-second batch on IBM LinuxONE (`s390x`).

**Honest reading:** CascadeGuard's advantage is far fewer false alarms and much better source identification. It is not faster than the N-sigma monitor, and it was slower on `loss` faults (59 s against 29 s).

## Example Alert
```
EARLY WARNING  (t = injection + 29s)
  Suspected source : adservice  (risk 1.00)
  Evidence         : cpu +30.0 sigma vs baseline
  Callers at risk  : frontend
  Payment flow     : not on the call path
  Suggested action : investigate adservice; check CPU saturation / autoscaling limits
                     (operator-approved runbook only)
  Confidence       : High
```

## Tech Stack
- **Platform:** IBM LinuxONE (`s390x`), Jupyter Notebook
- **Language:** Python 3.12
- **Libraries:** pandas, NumPy, scikit-learn (Isolation Forest, Gradient Boosting), Matplotlib
- **Data:** RCAEval (Online Boutique, RE1-OB)

## Getting Started
**1. Install the requirements**
```bash
pip install numpy pandas scikit-learn matplotlib
```
`pyarrow` is not needed. It has no prebuilt wheel for IBM Z, so the notebook reads the JSON/CSV version of the data.

**2. Download the dataset** (the first notebook cell does this)
```python
import urllib.request, zipfile
urllib.request.urlretrieve("https://zenodo.org/records/14590730/files/RE1-OB.zip?download=1", "RE1-OB.zip")
zipfile.ZipFile("RE1-OB.zip").extractall("data/RE1")
```

**3. Run the notebook**
Open `Cascade_guard.ipynb` in Jupyter and run the cells from top to bottom. Each section is explained in a markdown cell above its code.

## Limitations
- Lab fault injection on one demo application, not production bank traffic.
- Faults start at a known moment, so this measures detection speed, not forecasting.
- Not faster than a plain N-sigma monitor, and slower on `loss` faults.
- The call-graph impact estimate matches about half of the real spread.
- Test cases repeat the same fault and service combinations seen in training, so perfect source accuracy does not prove generalisation to new faults.
- A few cases use different metric column names (`latency`, `load`), so those metrics are currently unused.
- Running on LinuxONE does not mean integration with a production mainframe, z/OS or AI accelerators.

## Future Work
- Hold out an entire fault type (leave-one-fault-out) to test generalisation.
- Add a baseline that ignores the metrics, and repeat with several random splits.
- Map the unused `latency` and `load` column names.
- Test on other RCAEval systems and add logs and traces.

## Acknowledgements
- **RCAEval** benchmark: https://github.com/phamquiluan/RCAEval
- **Online Boutique** demo application by Google.
- Built for the **IBM Z Datathon 2026**.

## Demo Video

<!-- PASTE YOUR VIDEO HERE.
     Option 1 (GitHub): edit this README on github.com, drag and drop your video file into this spot, and GitHub inserts the link for you.
     Option 2 (YouTube / Drive): replace the link below.
-->

[Watch the demo video](PASTE_VIDEO_LINK_HERE)
