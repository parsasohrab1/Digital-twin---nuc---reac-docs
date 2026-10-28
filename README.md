Below is a **complete, integrated Software Requirements Specification (SRS)** based on the latest specialized version of the product (focused on a nuclear power plant with PINN, real-time EKF adaptation, LSTM-ARIMA transient prediction, mass scenario simulation, and a nuclear DSS).

This document is prepared in accordance with **IEEE Std 830-1998** and Iranian patent requirements and includes all the sections necessary for a deliverable software requirements engineering document.

---

# Software Requirements Specification (SRS)

## Nuclear Digital Twin System for Real-Time Thermal-Hydraulic Simulation, Safety Transient Prediction and Decision Support in Nuclear Power Plant Reactors

---

**Version:** 1.0
**Date:** 1405/03/13
**Status:** Final
**Audience:** Development team, patent examiners, nuclear systems engineers, safety assessors

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [General Description](#2-general-description)
3. [Operational Requirements (Nuclear-Specific)](#3-operational-requirements-nuclear-specific)
4. [Interface Requirements](#4-interface-requirements)
5. [Detailed Functional Requirements](#5-detailed-functional-requirements)
6. [Non-Functional Requirements](#6-non-functional-requirements)
7. [Nuclear Security and Safety Requirements](#7-nuclear-security-and-safety-requirements)
8. [Appendices](#8-appendices)

---

## 1. Introduction

### 1.1 Document Purpose

This Software Requirements Specification (SRS) specifies the **Nuclear Digital Twin (NDT)** system. The goal of the system is to create a real-time simulation model of the thermal-hydraulic behavior of the reactor core and the primary circuit of a nuclear power plant, with the following capabilities:

- Real-time simulation of coolant temperature, pressure and velocity fields using a **Physics-Informed Neural Network (PINN)**
- **Automatic adaptation of physical parameters** of the reactor core (fuel rod roughness coefficient, fouling coefficient, pressure drop coefficient) without shutting down the reactor
- **Early prediction of safety transients** (LOFA, small LOCA, critical boiling DNB) through mass "what-if" scenario simulation
- **Nuclear Decision Support System (Nuclear DSS)** providing ranked corrective actions to the control room operator

### 1.2 Product Scope

The system is installed as an **intelligent software layer** on top of the nuclear power plant's existing I&C system. The scope includes:

| Component | Description |
|------|--------|
| Data acquisition | Connection to pressure, temperature, flow and neutron sensors through OPC UA / Modbus |
| PINN model | Direct prediction of the thermal-hydraulic fields of the reactor core |
| Adaptation unit | Updating physical parameters with EKF in less than 100 milliseconds |
| Prediction unit | Trend analysis (LSTM+ARIMA) and simulation of ≥500 scenarios on GPU |
| DSS | Multi-objective optimization (Pareto Front) and ranking of safety actions |
| User interface | Web-based control room dashboard with latency <200 milliseconds |

### 1.3 Definitions and Abbreviations

| Term | Definition |
|-------|--------|
| **PINN** | Physics-Informed Neural Network |
| **EKF** | Extended Kalman Filter |
| **LSTM** | Long Short-Term Memory |
| **ARIMA** | AutoRegressive Integrated Moving Average |
| **DSS** | Decision Support System |
| **LOFA** | Loss of Flow Accident – coolant flow reduction accident |
| **LOCA** | Loss of Coolant Accident |
| **DNB** | Departure from Nucleate Boiling – critical boiling |
| **RPV** | Reactor Pressure Vessel |
| **DNBR** | Departure from Nucleate Boiling Ratio – critical boiling safety ratio |
| **TTE** | Time to Event – remaining time until an event |
| **RUL** | Remaining Useful Life |

### 1.4 References

| Reference | Title |
|-------|-------|
| IEEE 830-1998 | Recommended Practice for Software Requirements Specifications |
| IEEE 7-4.3.2 | Standard for Digital Computers in Safety Systems of Nuclear Plants |
| 10 CFR 50.46 | Acceptance criteria for emergency core cooling systems (US NRC) |
| IAEA SSG-39 | Design of Instrumentation and Control Systems for Nuclear Power Plants |
| IRIPO | Iranian patent law (approved 1386) |

---

## 2. General Description

### 2.1 Product Vision

The NDT system is installed as an **intelligent add-on** on top of the nuclear power plant's existing I&C system. Data is received from existing sensors (and a few suggested additional sensors), processed, and the output is sent to the control room dashboard. The system operates **independently** and its output can only be applied after operator confirmation (semi-automatic mode).

### 2.2 System Users

| Role | Description |
|------|--------|
| Senior reactor operator | Use of the dashboard, viewing alerts and DSS recommendations |
| Nuclear safety engineer | Analyzing outputs, setting safety thresholds |
| I&C systems engineer | Maintenance, initial calibration, updating the PINN model |
| Plant manager | Receiving summary reports and system performance statistics |

### 2.3 Design Constraints (Nuclear-Specific)

| Constraint | Value |
|----------|--------|
| Maximum end-to-end latency | ≤ 500 milliseconds |
| Maximum simulation time per scenario | ≤ 10 milliseconds |
| Maximum simulation time for 500 scenarios | ≤ 1 second |
| Temperature prediction accuracy (relative to sensors) | Relative error ≤ 2% |
| Pressure prediction accuracy | Relative error ≤ 5% |
| Maximum sensor disconnection time (before alert) | 3 seconds |
| Sensor sampling frequency | ≥ 10 Hz |
| Physical parameter storage buffer | ≥ 1000 samples |
| Required GPU memory | ≥ 16 GB |
| Server operating temperature (per nuclear standard) | 5 to 40 degrees Celsius |

### 2.4 Assumptions and Dependencies

- Pressure, temperature and flow sensors are installed at key points of the primary circuit.
- A GPU server with independent cooling is deployed in the control room.
- The operator has access to a display with a resolution of at least 1920×1080.
- The existing I&C system supports the OPC UA protocol.
- Reactor design data (geometry, material properties, fuel assembly map) is available.

---

## 3. Operational Requirements (Nuclear-Specific)

### 3.1 System Operating Modes

| Mode | Description | Activation time |
|------|--------|----------------|
| **Initial training (Offline Training)** | Training the PINN based on simulation data (such as RELAP5) | Before operation, about 24 hours |
| **Calibration** | Initial estimation of physical parameters with real data from the first 7 days | First 7 days after installation |
| **Normal Operation** | Closed-loop adaptation + prediction + display | All other times |
| **Transient Mode** | Mass scenario simulation + DSS active | Automatically if a parameter rate crosses a threshold |
| **Safe Mode** | If communication with sensors is lost, continue with the last parameters | After 3 seconds of disconnection |

### 3.2 Startup Requirements

| Requirement | Description |
|------|------|
| NDT-ST-01 | Cold start time (from server power-on to standby) ≤ 2 minutes |
| NDT-ST-02 | Loading the latest version of the PINN model from non-volatile memory |
| NDT-ST-03 | Automatic connection to the OPC UA gateway in less than 10 seconds |
| NDT-ST-04 | Verification of sensor validity (min/max) before entering normal operation |

### 3.3 Shutdown Requirements

| Requirement | Description |
|------|------|
| NDT-SD-01 | Automatic saving of the latest physical parameters (ε_fuel, R_f, K_spacer) before shutdown |
| NDT-SD-02 | Saving the last 1000 sensor data samples in the database |
| NDT-SD-03 | Creating a summary report of the last 24 operating hours |

---

## 4. Interface Requirements

### 4.1 User Interface (UI)

#### 4.1.1 Main Control Room Dashboard

| UI-RQ-01 | Display a heatmap of the reactor core in (r,θ,z) coordinates with an update ≤ 200 milliseconds |
| UI-RQ-02 | Numeric display of temperature, pressure and flow at key points (core inlet, core outlet, primary loop) |
| UI-RQ-03 | Display of DNBR (critical boiling safety ratio) for the hottest fuel assembly |
| UI-RQ-04 | Display of the estimated time to event (TTE) for each active alert |
| UI-RQ-05 | Display of three corrective actions suggested by the DSS with a safety rank (1 to 5) and success probability |
| UI-RQ-06 | An "Approve action" button to send a command to the control system (with two-step confirmation) |
| UI-RQ-07 | A low-light "night" mode (Dark mode) for the control room |

#### 4.1.2 Settings and Configuration Page

| UI-RQ-08 | Setting safety thresholds (allowable cladding temperature, critical DNBR, allowable roughness rate of change) |
| UI-RQ-09 | Viewing and editing the initial parameters of the PINN model |
| UI-RQ-10 | Reloading the PINN model from a file |
| UI-RQ-11 | Viewing the event and alert log of the past 30 days |

### 4.2 Hardware Interface (I/O)

| HW-RQ-01 | Support for the OPC UA protocol (Client) to receive data from the nuclear PLC |
| HW-RQ-02 | Support for Modbus TCP as an alternative |
| HW-RQ-03 | Galvanic isolation between the server and the main control network |
| HW-RQ-04 | Support for digital (binary) output if automatic mode is enabled (optional) |
| HW-RQ-05 | Configurable sampling rate from 1 to 100 Hz |

### 4.3 Software Interface (API)

| API-RQ-01 | REST API to receive the system status as JSON |
| API-RQ-02 | Ability to request manual simulation of a specific scenario (by the safety engineer) |
| API-RQ-03 | Output of historical data for offline analysis (CSV/Parquet) |

---

## 5. Detailed Functional Requirements

### 5.1 Data Ingestion Module

| ID | Requirement |
|----|----------|
| DI-01 | Connection to at least 20 data points including: core inlet/outlet pressure (2 points), temperature (8 points), flow (2 points), neutron linear power (8 points) |
| DI-02 | Noise filtering with a median filter (window of 5 samples) |
| DI-03 | Sensor disconnection detection: if no data is received for 3 seconds, a "Cut Sensor" alert must be issued |
| DI-04 | Storing raw data in the time-series database (InfluxDB) with 30-day retention |

### 5.2 PINN Module (Core Model)

| ID | Requirement |
|----|----------|
| PINN-01 | Architecture: 6 hidden layers, 128 neurons per layer, Swish activation function |
| PINN-02 | Input: (r, θ, z, t, T_in(t), P_in(t), m_dot(t), Q_total(t), θ_phys) |
| PINN-03 | Output: T_coolant(r,θ,z,t), P(r,θ,z,t), v_z(r,θ,z,t), T_fuel_surface(r,θ,z,t) |
| PINN-04 | Cost function = Loss_MSE + λ₁×Loss_NS + λ₂×Loss_Energy + λ₃×Loss_NeutronicsCoupling |
| PINN-05 | λ_values configurable in the configuration file (default: λ₁=0.1, λ₂=0.1, λ₃=0.01) |
| PINN-06 | Prediction time of each step ≤ 50 milliseconds on an NVIDIA A100 GPU |

### 5.3 Real-Time Adaptation Module (EKF-based Adaptive Unit)

| ID | Requirement |
|----|----------|
| AD-01 | State vector: x = [ε_fuel, R_f, K_spacer, T_coolant_out] |
| AD-02 | Measurement: output of temperature and pressure sensors at key points |
| AD-03 | Process and measurement noise covariance matrices: configurable by the engineer |
| AD-04 | Update frequency: every 1 second |
| AD-05 | Time of each update ≤ 20 milliseconds |
| AD-06 | Storing the EKF output (estimated parameters) in a 1000-sample ring buffer |
| AD-07 | If the EKF diverges (normalized innovation > 3), reset with the last valid parameter |

### 5.4 Transient Prediction and Scenario Simulation Module

| ID | Requirement |
|----|----------|
| FP-01 | Trend analysis of physical parameters with LSTM (50 memory steps, 2 layers) + ARIMA(2,1,2) |
| FP-02 | Trend analysis execution frequency: every 10 seconds |
| FP-03 | Anomaly detection: if the rate of change of ε_fuel or R_f in the past 24 hours > dynamic threshold |
| FP-04 | When an anomaly is detected, simulate ≥500 "what-if" scenarios by varying parameters within the 95% uncertainty corridor |
| FP-05 | Time horizon of each scenario: 300 seconds with a 1-second time step |
| FP-06 | Total simulation time of 500 scenarios on GPU ≤ 1 second |
| FP-07 | Output: statistical distribution of time to DNB, time to reaching a cladding temperature of 620°C |

### 5.5 Nuclear Decision Support System (Nuclear DSS)

| ID | Requirement |
|----|----------|
| DSS-01 | Objective functions: (1) maximum T_fuel_surface, (2) maximum pressure fluctuation, (3) deviation from setpoint power |
| DSS-02 | Decision variables: reactor power (50-100%), main pump flow (80-120%), boric acid injection rate (0-100 ppm/min) |
| DSS-03 | Optimizer: NSGA-II (Non-dominated Sorting Genetic Algorithm) with an initial population of 100 |
| DSS-04 | Optimizer execution time ≤ 500 milliseconds |
| DSS-05 | Output: Pareto front with a maximum of 10 non-dominated solutions |
| DSS-06 | Final ranking based on the "safety-first" index = w1×(1 - T_fuel/620) + w2×DNBR |
| DSS-07 | Display the top three solutions along with: action description, safety rank (1-5), success probability (0-100%), suggested execution time |

### 5.6 Alert and Reporting Module

| ID | Requirement |
|----|----------|
| AL-01 | Red level alert (critical): DNB risk within the next ≤ 300 seconds |
| AL-02 | Orange level alert (high): risk of exceeding a cladding temperature of 600°C within ≤ 600 seconds |
| AL-03 | Yellow level alert (medium): abnormal rate of change of physical parameters |
| AL-04 | Blue level alert (informational): automatic calibration was performed |
| AL-05 | Recording all alerts in the database with exact time (NTP synchronized) |
| AL-06 | Ability to temporarily mute audible alerts (snooze) for 5 minutes |

---

## 6. Non-Functional Requirements

### 6.1 Reliability

| ID | Requirement |
|----|----------|
| NFR-R-01 | MTBF (mean time between failures) ≥ 8760 hours (one year) |
| NFR-R-02 | MTTR (mean time to repair) ≤ 2 hours |
| NFR-R-03 | Availability ≥ 99.95% |
| NFR-R-04 | Automatic failover to a backup server (Active-Passive) in less than 1 second |

### 6.2 Performance

| ID | Requirement |
|----|----------|
| NFR-P-01 | End-to-end response time (from sensor reception to dashboard update) ≤ 500 milliseconds |
| NFR-P-02 | Maximum RAM usage ≤ 32 GB |
| NFR-P-03 | Maximum GPU VRAM usage ≤ 16 GB |
| NFR-P-04 | Ability to process at least 20 API requests simultaneously |

### 6.3 Maintainability

| ID | Requirement |
|----|----------|
| NFR-M-01 | Modularity (microservice architecture with Docker) |
| NFR-M-02 | API documentation with OpenAPI 3.0 |
| NFR-M-03 | Coupling coefficient between modules ≤ 0.4 (CBO metric) |
| NFR-M-04 | Ability to update the PINN model without stopping the system (Hot-swap) |

### 6.4 Scalability

| ID | Requirement |
|----|----------|
| NFR-S-01 | Ability to increase the number of sensors up to 100 points without architectural change |
| NFR-S-01 | Ability to add a second reactor (multi-twin) by adding a new GPU server |

---

## 7. Nuclear Security and Safety Requirements

### 7.1 Cybersecurity

| ID | Requirement |
|----|----------|
| SEC-01 | Two-factor authentication (PIN + smart card) for the operator |
| SEC-02 | Encryption of all communications with TLS 1.3 |
| SEC-03 | Network separation (VLAN) between the NDT server and the plant's general network |
| SEC-04 | Logging of all failed login attempts |
| SEC-05 | No storage of passwords in configuration files (use Vault) |

### 7.2 Nuclear Safety

| ID | Requirement |
|----|----------|
| SAF-01 | The system must **never** issue an executive command to the control system automatically (only a recommendation to the operator) |
| SAF-02 | The DSS output must be validated by three independent mechanisms (anomaly detection, expert logic, comparison with RELAP5) |
| SAF-03 | If the DSS output conflicts with the plant's approved safety procedures, the message "Not approved - operator decision required" must be displayed |
| SAF-04 | The system must have a hardware switch "Disconnect from DSS" in the control room |
| SAF-05 | All DSS recommendations must be recorded in an immutable log (Blockchain-based or WORM storage) |

---

## 8. Appendices

### 8.1 Requirement ID Table

| Prefix | Meaning |
|--------|------|
| NDT | General |
| DI | Data Ingestion |
| PINN | PINN Core |
| AD | Adaptive Unit (EKF) |
| FP | Fault Prediction |
| DSS | Decision Support |
| AL | Alert |
| UI | User Interface |
| HW | Hardware Interface |
| API | Software Interface |
| NFR | Non-Functional |
| SEC | Security |
| SAF | Safety |

### 8.2 Requirements Traceability Matrix

| Requirement | Component | Priority | Source |
|----------|-------|--------|------|
| PINN-01 to 06 | PINN Core | Critical | Claim 1 |
| AD-01 to 07 | EKF Adapter | Critical | Claim 1 |
| FP-01 to 07 | Predictor | High | Claims 1, 2 |
| DSS-01 to 07 | DSS | High | Claim 4 |
| SAF-01 to 05 | Safety | Critical | IEEE 7-4.3.2 standard |

### 8.3 Verified Execution Environment

| Component | Specifications |
|------|--------|
| CPU | Intel Xeon Gold 6330 (28 cores) |
| GPU | NVIDIA A100 40GB or RTX 6000 Ada |
| RAM | 128 GB DDR4 ECC |
| Storage | NVMe SSD 2TB (RAID 1) |
| OS | Ubuntu 22.04 LTS with PREEMPT_RT kernel |
| Software | Python 3.10, TensorFlow 2.13, PyTorch 2.0, FastAPI, InfluxDB 2.7 |

### 8.4 Additional Glossary

| Term | Definition |
|-------|--------|
| **Hot Channel** | The hottest fuel assembly in the reactor core |
| **Cladding** | The zircaloy sheath around fuel pellets |
| **Scram** | Emergency reactor shutdown (rapid insertion of control rods) |
| **Boration** | Injection of boric acid to absorb neutrons and reduce power |

---

**End of SRS document**

I am ready to help complete other sections (such as UML diagrams, test scenarios, or conversion to the format required by the Iranian patent office).
