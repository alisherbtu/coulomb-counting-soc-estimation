# Battery SOC Estimation by Coulomb Counting: How Far Does It Drift?

![MATLAB](https://img.shields.io/badge/MATLAB-Simulink-orange) ![Simscape](https://img.shields.io/badge/Simscape-Battery-blue) ![Status](https://img.shields.io/badge/status-v1%20synthetic%20profile-lightgrey)

**A Simulink / Simscape model that estimates a lithium-ion cell's state of charge (SOC) by counting charge, then breaks the estimator on purpose to show exactly how and why it drifts.**

Part of my 30-day self-study series on battery management systems (BMS). Author: Ali Sher, M.Sc. Hybrid Electric Propulsion Technology, BTU Cottbus-Senftenberg.

---

## TL;DR
You cannot measure SOC directly, so a BMS counts the charge that flows in and out:

```
SOC(t) = SOC(0) - (1/Q) * integral of I dt
```

It is simple and needs no battery model, but it never corrects itself. I built the estimator, checked it against the cell model's true SOC, and then injected three real-world faults. Each one leaves a different fingerprint in the error.

| Case | Fault injected | Error at 7200 s | Error signature |
|---|---|---|---|
| Ideal | none | about 1e-16 (numerical noise) | flat zero |
| **A** | wrong initial SOC (0.90 instead of 0.80) | **+0.10** | constant offset, never recovers |
| **B** | current sensor offset of +0.05 A | **about -0.033** | ramp, grows even while the cell rests |
| **C** | capacity assumed 3.3 Ah (true 3.0 Ah) | **about +0.015** | steps, changes only while current flows |

Reading the *shape* of the error tells you which fault you have.

---

## The model
- **Cell:** Battery (Table-Based), 3 Ah, R0 = 20 mOhm, OCV 3.0 V (0 %) to 4.2 V (100 %), true initial SOC 0.80. Its SOC output is the ground truth.
- **Load profile:** +3 A for 600 s, rest 600 s, -2 A for 600 s, rest 600 s, repeated for 7200 s. The cell is net discharged, so true SOC ends at **0.633**.
- **Estimator:** Current Sensor, plus a fault-injection offset, times -1/Q (Q = 10,800 As), integrated from the initial SOC.
- **Error:** estimated SOC minus true SOC.

![Simulink model](<day 12 main Model.png>)

---

## Results

### Case A: wrong starting point
The estimator starts at 0.90 but the cell is at 0.80. Coulomb counting only adds and subtracts, so the 0.10 gap is carried forever.

![Case A SOC](<Case A Soc.png>)
![Case A error](<Case A Estimated Minus value.png>)

### Case B: current sensor offset
A +0.05 A bias on the sensor looks like a tiny extra discharge every second, including at rest. The error is a straight ramp of about 0.05 A x 7200 s / 10,800 As = 0.033 SOC.

![Case B SOC](<Case B SOC.png>)
![Case B error](<case B Estimated minus value.png>)

### Case C: wrong capacity
Using 3.3 Ah instead of 3.0 Ah makes the estimator under-react to every ampere. The error only moves while current flows, so it rises during discharge, falls during charge, and keeps a small net drift.

![Case C SOC](<Case C Soc.png>)
![Case C error](<Case C estimated minus Value.png>)

---

## What this shows
1. **Coulomb counting is exact only when three things are right:** the starting SOC, the current measurement, and the capacity.
2. **Error signatures are diagnostic:** a flat line is a bad start, a ramp is a sensor offset, steps that follow the current are a capacity error.
3. **Small biases become big errors:** 0.05 A of offset would add roughly 0.4 SOC over 24 hours, which is useless without correction.
4. **Capacity must track aging:** a faded cell holds less than its label, which is Case C in real life.
5. **This is why production BMSs fuse methods:** Coulomb counting for smooth, fast updates, plus an open-circuit-voltage correction at rest or a Kalman filter to remove the drift.

## Where this is used
The same integrator is the core of the fuel gauge in phones, laptops, e-bikes, drones and EV battery management systems. The correction layer on top of it (OCV reset, extended Kalman filter) is what separates a toy gauge from a trustworthy one.

---

## How to run
1. Open `day12_coulomb_counting.slx` in MATLAB (Simulink, Simscape, Simscape Electrical, Simscape Battery required).
2. Set the stop time to 7200 s and run: this is the ideal case.
3. Reproduce the faults by changing one value at a time and resetting it afterwards:
   - Case A: Integrator initial condition = 0.9
   - Case B: offset Constant = 0.05
   - Case C: Gain = -1/11880

## Limitations (honest version)
- The load is a synthetic stair profile, not measured data.
- The current sensor is ideal apart from the injected offset: no noise, no gain error, no quantization.
- No temperature effect and no coulombic efficiency.
- The cell model is R0-only with no RC dynamics.

## Roadmap
- [ ] **v2: validate on a real EV drive cycle** (measured UDDS / US06 current from a public lithium-ion dataset) and report RMS and maximum SOC error
- [ ] Add sensor noise, gain error and quantization
- [ ] OCV-based correction at rest
- [ ] Extended Kalman filter SOC estimator

---

## More from this series
| Project | Topic |
|---|---|
| [battery-pack-grid](https://github.com/alisherbtu/battery-pack-grid) | 10.8 kWh grid-connected pack driven by a power command |
| [hybrid-cell-battery-model](https://github.com/alisherbtu/hybrid-cell-battery-model) | energy cell + power cell sharing a load |
| [battery-charge-discharge-circuit](https://github.com/alisherbtu/battery-charge-discharge-circuit) | charging and discharging a cell through a simple circuit |
| [passive-cell-balancing](https://github.com/alisherbtu/passive-cell-balancing) | bleed-resistor balancing of a 4S pack |
| [active-cell-balancing](https://github.com/alisherbtu/active-cell-balancing) | charge-transfer balancing and controller stability |
| **coulomb-counting-soc-estimation** | this project |

*Learning project: results come from the simulation model described above, not from lab measurements.*
