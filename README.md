# Trace-driven Simulation of Vision-Aided Mobile RAN Control Using ns-3 Digital Twin

This repository contains the simulation framework and source code for the Master's thesis **"Trace-driven Simulation of Vision-Aided Mobile RAN Control Using ns-3 Digital Twin"** by Seyed Mohammad Hossein Omrani, Universidade do Porto. 

This project develops a Trace-Based Digital Twin (TBDT) to evaluate autonomous, vision-aided mobile base stations in 6G Open Radio Access Network (O-RAN) architectures. By combining real-world empirical trace data with kinematically bounded Machine Learning (ML) actuation, this simulation environment aims to overcome the fragility of millimeter-wave (mmWave) signals in dynamic environments[cite: 3].

## System Architecture

The digital twin framework decouples the simulation into three distinct functional domains to mirror physical O-RAN topologies[cite: 3]:

1. **Physical Twin Engine (ns-3)**: A C++ based simulation environment that provides a low-overhead geometric physics model[cite: 3]. It utilizes a custom `MlPropagationLossModel` to evaluate dynamic line-of-sight (LoS) blockages. 
2. **Virtual E2 Interface (UDP IPC)**: A transport-layer bridge that operates over local UDP Port 9999 to facilitate bidirectional communication between the simulation and external controllers[cite: 1, 2, 3].
3. **Intelligence Plane (Python Near-RT RIC)**: An external Python-based controller hosting the VisionApp (xApp)[cite: 3]. It utilizes a `RandomForestRegressor` to dynamically predict signal attenuation (SNR) based on spatial telemetry and dictates physical evasion maneuvers within a strict $1.5 \text{ m/s}$ robotic speed limit[cite: 1, 3].

## Repository Structure

*   `vision_app_xapp.py`: The Python-based Near-RT RIC xApp[cite: 1, 3]. It processes real-time spatial telemetry, predicts mmWave signal degradation, and calculates optimal base station evasion targets.
*   `digital_twin_ns3.cc`: The core C++ ns-3 simulation script (`Step3DynamicDigitalTwin`). It simulates a 28 GHz mmWave 5G channel, manages node mobility, and handles TCP/UDP traffic generation alongside the 1.5 MB bounded RLC queues.
*   `Cleaned_Trace.csv`: Historical empirical spatial trajectory data (X, Y coordinates and time intervals) used to drive the UE and Obstacle mobility in the ns-3 simulator[cite: 2, 3].
*   `Experimental_Data_VisionApp.txt`: The physical testbed dataset containing spatial features and empirical SNR values used to dynamically train the Random Forest ML model upon initialization.

## Prerequisites

To execute this digital twin, the following dependencies must be installed on your system:

### 1. ns-3 Environment
*   **ns-3** (configured with the **5G-LENA** module).
*   Standard C++ build tools (GCC/G++).

### 2. Python Environment
*   Python 3.8+
*   `pandas`: For parsing empirical datasets[cite: 1].
*   `scikit-learn`: For the Random Forest Regressor[cite: 1].

## Setup and Execution

Because this framework relies on synchronous Inter-Process Communication (IPC) over a UDP socket, the Python controller and the ns-3 simulator must be run concurrently.

### Step 1: Initialize the Intelligence Plane (Near-RT RIC)
First, start the Python xApp. Ensure that the `Experimental_Data_VisionApp.txt` dataset is in the same working directory[cite: 1]. The script will train the ML model and begin listening on `127.0.0.1:9999`[cite: 1].

```bash
python3 vision_app_xapp.py
