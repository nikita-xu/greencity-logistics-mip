# Sustainable Urban Logistics Optimization — GreenCity Logistics

An integrated **mixed-integer programming (MIP)** decision system for designing a sustainable last-mile delivery network, built with **Python and Gurobi**.

> **Course:** ISBA 2405 — Prescriptive Analytics, Santa Clara University
> **Team 5:** Wenni Xu · Tianqi Wang · Xiao Yue Lin · Jiaru Li
> **Date:** June 1, 2026

---

## Problem

GreenCity Logistics must configure an urban last-mile network at minimum cost while meeting capacity, emissions, and service requirements. The model answers four decisions at once:

1. **Which** candidate micro-fulfillment centers (MFCs) to open
2. **Which** facility serves each demand zone
3. **Which** vehicle type runs each facility–zone route, and how many trips
4. **Whether** the network satisfies capacity, range, time-window, low-emission-zone (LEZ), and emission-cap constraints

The model also encodes **Client C103** (e-commerce provider) requirements: electric vehicles in LEZs, 2-hour delivery windows, at most 6 stops per facility route pattern, and complete service of every zone.

## Data

| Dataset | Size | Description |
|---|---|---|
| `location_data.csv` | 25 candidate facilities | Setup cost, lease, utilities, capacity, suitability |
| `demand_data.csv` | 30 demand zones | Daily demand, priority %, time windows |
| `vehicle_data.csv` | 10 vehicle types | Capacity, range, cost, CO₂/km, fuel type |
| `distance_matrix_SAMPLE_50routes.csv` | 50 routes | Provided sample, completed to 750 routes (see below) |
| `emission_constraints.csv` | 30 zones | Per-zone emission caps, LEZ flag, time restrictions |

The provided distance file covers only 50 of the 750 facility–zone pairs (25 × 30). The notebook builds the full matrix with haversine distances from coordinates and rule-based estimates for travel time, congestion, emission factor, toll, and reliability.

> The course datasets are not included in this repo. See [`data/README.md`](data/README.md).

## Model

**Decision variables**

| Variable | Type | Meaning |
|---|---|---|
| `y[i]` | binary | Facility *i* is opened |
| `z[i,j]` | binary | Facility *i* serves zone *j* |
| `x[i,j]` | continuous | Packages shipped *i → j* |
| `w[i,j,v]` | binary | Vehicle type *v* serves route *(i,j)* |
| `r[i,j,v]` | integer | Trips of type *v* on *(i,j)* |
| `n[v]` | integer | Vehicles of type *v* acquired |

**Three objectives**

- **Z₁ Cost (min):** facility daily cost + transport cost per trip + fleet daily cost
- **Z₂ Emissions (min):** route emissions per trip × trips
- **Z₃ Service (max):** reliability × (1 / travel time), summed over assigned routes

**Multi-objective method:** a payoff table (optimize each objective alone) supplies normalization bounds, then a normalized weighted-sum objective is solved under four weight scenarios.

**Constraints:** demand satisfaction · facility capacity (open facilities only) · flow–assignment linking · single-sourcing · one vehicle type per active route · vehicle trip capacity · fleet sizing · vehicle range · time-window feasibility · per-zone emission caps · LEZ ⇒ electric · C103 rules.

**Key assumptions:** per-operating-day cost basis, 10-year setup amortization, 250 operating days/year, 8 trips per vehicle per day, C103 zones defined as zones with priority ≥ 35% (13 of 30 zones).

## Results

**Recommended plan (Balanced scenario, weights 0.40 / 0.30 / 0.30):**

| Metric | Value |
|---|---|
| Facilities opened | 11 of 25 |
| Zones served | 30 of 30 (single-sourced) |
| Total cost | $10,983 / day (about $1.29 per package) |
| Emissions | 0.57 kg CO₂ / day |
| Fleet | All-electric delivery trucks |
| Solver status | Optimal, 0.00% MIP gap, about 0.6 s |

**Scenario comparison**

| Scenario | Weights (cost, emissions, service) | Cost ($/day) | Emissions (kg/day) | Service | Facilities |
|---|---|---|---|---|---|
| Balanced | 0.40, 0.30, 0.30 | 10,983 | 0.572 | 28.61 | 11 |
| Cost-priority | 0.70, 0.15, 0.15 | 5,219 | 1.057 | 25.04 | 5 |
| Emission-priority | 0.15, 0.70, 0.15 | 22,972 | 0.165 | 29.10 | 23 |
| Service-priority | 0.15, 0.15, 0.70 | 12,025 | 0.530 | 29.10 | 12 |

**Parameter sensitivity (Balanced re-solved):**

| Shock | Cost ($/day) | Facilities | Recommendation changes? |
|---|---|---|---|
| Demand +20% | 12,008 | 12 | Yes |
| Emission caps −50% | 10,983 | 11 | No |

**Takeaways**

- Facility lease, setup, and utilities make up roughly 97% of daily cost. Every route is under about 2 km, so transport cost is negligible and the cost objective is mostly about how few facilities can cover demand.
- The optimizer picks the electric delivery truck (350-package capacity) everywhere, which cuts trip counts and satisfies the LEZ rule in 24 of 30 zones.
- The Cost-priority plan (5 facilities) is a documented budget alternative.
- A two-phase rollout is proposed in the notebook (Report 10).

## Outputs

The notebook produces 11 reports: selected facilities, demand allocation, vehicle assignment, facility–zone service matrix, environmental compliance, cost breakdown, district performance, multi-objective trade-off, parameter sensitivity, implementation plan, and managerial recommendation, plus a network map and cost breakdown chart.

## Repository structure

```
greencity-logistics-mip/
├── README.md
├── requirements.txt
├── .gitignore
├── notebook/
│   └── Group_5_GreenCity_Capstone_Notebook.ipynb
└── data/
    └── README.md
```

## Getting started

**Requirements:** Python 3.9+ and a Gurobi license. Students and faculty can get a free [WLS Academic license](https://www.gurobi.com/academia/academic-program-and-licenses/).

```bash
git clone https://github.com/<your-username>/greencity-logistics-mip.git
cd greencity-logistics-mip
pip install -r requirements.txt
```

1. Place the five CSV files in `data/` (or any folder you like).
2. Open the notebook. It was written for **Google Colab**, so the data-loading cell mounts Google Drive. To run locally, remove the `google.colab` import and set `BASE_PATH` to your data folder.
3. Provide your Gurobi credentials through environment variables instead of pasting them into the notebook:

```python
import os
import gurobipy as gp

ENV = gp.Env(params={
    "WLSACCESSID": os.environ["WLSACCESSID"],
    "WLSSECRET":   os.environ["WLSSECRET"],
    "LICENSEID":   int(os.environ["LICENSEID"]),
})
```

4. Run all cells top to bottom.

## Limitations and next steps

- **Data:** the completed 750-route matrix relies on coordinate-based distances and rule-based assumptions, not observed traffic. The C103 zone mapping is an assumption.
- **Model:** linear MIP with fixed congestion, emissions, and reliability parameters; time windows are a feasibility filter, not full routing with sequencing; uniform 8 trips/day ignores EV charging time.
- **Extensions:** vehicle-specific flow for mixed fleets, multi-client modeling (e.g., C101 cold-chain with C103), stochastic or robust demand using the weekend demand factor, an ε-constraint Pareto frontier in place of the weighted sum, and piecewise-linear congestion curves.

## Tech stack

Python · Gurobi (`gurobipy`) · pandas · NumPy · Matplotlib

## Team

Wenni Xu · Tianqi Wang · Xiao Yue Lin · Jiaru Li

## Acknowledgments

ISBA 2405 Prescriptive Analytics, Santa Clara University. Datasets and the distance-matrix generation logic were provided by the course instructor.
