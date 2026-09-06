```markdown
# Trace-driven Simulation of Vision-Aided Mobile RAN Control Using ns-3 Digital Twin

This repository contains the simulation framework and source code for the Master's thesis **"Trace-driven Simulation of Vision-Aided Mobile RAN Control Using ns-3 Digital Twin"** by Seyed Mohammad Hossein Omrani, Faculdade de Engenharia da Universidade do Porto (FEUP)[cite: 3]. 

This project develops a Trace-Based Digital Twin (TBDT) to evaluate autonomous, vision-aided mobile base stations in 6G Open Radio Access Network (O-RAN) architectures[cite: 3]. By combining empirical trace data with kinematically bounded Machine Learning (ML) actuation, this simulation environment addresses the fragility of millimeter-wave (mmWave) signals in dynamic environments[cite: 3]. 

## System Architecture

The digital twin framework decouples the simulation into three distinct functional domains to mirror physical O-RAN topologies[cite: 3]:

1. **Physical Twin Engine (ns-3)**: A C++ based simulation environment providing a low-overhead geometric physics model[cite: 3]. It utilizes a custom `MlPropagationLossModel` to evaluate dynamic line-of-sight (LoS) blockages and enforces a strict 1.5 MB RLC tail-drop buffer to emulate physical hardware constraints[cite: 2].
2. **Virtual E2 Interface (UDP IPC)**: A transport-layer bridge operating over local UDP Port 9999, facilitating bidirectional, synchronized communication between the simulation and external controllers[cite: 1, 2, 3].
3. **Intelligence Plane (Python Near-RT RIC)**: An external Python-based controller hosting the VisionApp (xApp)[cite: 3]. It utilizes a Random Forest Regressor to dynamically predict signal attenuation (SNR) based on spatial telemetry and dictates evasion maneuvers within a strict 1.5 m/s robotic speed limit[cite: 1, 3].

## System Compatibility and Software Versions

To ensure full scientific reproducibility and prevent CMake build errors, the host environment must align with the component versions specified below:

| Component | Required Version | Repository / Source |
| :--- | :--- | :--- |
| **Operating System** | Ubuntu 22.04 / 24.04 LTS | Canonical (x86_64) |
| **Base Simulator** | ns-3.45 | Git branch: `ns-3.45` |
| **5G-NR Module** | 5g-lena-v4.1.y | Git branch: `5g-lena-v4.1.y` |
| **C++ Core Libraries**| `libeigen3`, `sqlite3`, `libc6` | Ubuntu APT Repositories |
| **Python Environment**| Python 3.10+ | PyPI (`scikit-learn`, `pandas`, `numpy`) |

## Environment Setup and Compilation

### 1. System Packages and ns-3 Dependencies
The framework requires specific compilers and 5G-LENA dependencies (such as Eigen3 for MIMO calculations and SQLite for tracing). Install these on the Linux host prior to compilation:

```bash
sudo apt-get update
sudo apt-get install -y build-essential g++ gcc cmake ninja-build git \
  python3 python3-dev python3-pip python3-venv libc6-dev sqlite3 \
  libsqlite3-dev libeigen3-dev libxml2-dev libboost-all-dev

```

### 2. Python xApp Environment

The external intelligence plane requires an isolated virtual environment containing the specific Machine Learning libraries used by the controller:

```bash
python3 -m venv ~/digital_twin_env
source ~/digital_twin_env/bin/activate
pip install --upgrade pip
pip install pandas scikit-learn numpy

```

### 3. Cloning and Aligning ns-3 and 5G-LENA

Because the 5G-LENA module operates as an ns-3 plug-in, the repositories must be cloned, and the specific compatible branches must be checked out:

```bash
# Clone base simulator and checkout release ns-3.45
git clone [https://gitlab.com/nsnam/ns-3-dev.git](https://gitlab.com/nsnam/ns-3-dev.git)
cd ns-3-dev
git checkout -b ns-3.45 ns-3.45

# Clone and checkout 5G-LENA inside the contrib directory
cd contrib
git clone [https://gitlab.com/cttc-lena/nr.git](https://gitlab.com/cttc-lena/nr.git)
cd nr
git checkout -b 5g-lena-v4.1.y origin/5g-lena-v4.1.y
cd ../..

```

### 4. Configuring and Building the Simulator

Configure the project using the CMake wrapper script. Verify that SQLite and Eigen3 support are enabled during the configuration phase, then compile:

```bash
./ns3 configure --enable-examples --enable-tests --build-profile=release
./ns3 build

```

## File Structure and Placement

For the executable scripts to locate the necessary datasets and correctly output telemetry, all custom simulation components must reside in specific paths relative to the `ns-3-dev/` root directory:

```text
ns-3-dev/
 ├── scratch/
 │   └── mmwave-obstacle.cc          (C++ ns-3 Digital Twin source code)
 ├── Cleaned_Trace.csv               (Empirical waypoint coordinates)
 ├── Experimental_Data_VisionApp.txt (ML dataset for Random Forest training)
 ├── vision_app_controller.py        (Python O-RAN xAPP server)
 └── DlDataSinr.txt                  (Trace file initialized before run)

```

**Note:** The `DlDataSinr.txt` file must be manually initialized in the root directory prior to execution, as the telemetry parser relies on its existence:

```bash
touch DlDataSinr.txt

```

## Execution Instructions

Because the ns-3 simulation utilizes a blocking `recvfrom()` call on UDP port 9999, the Python ML controller must be actively listening before the ns-3 simulation is launched. This requires parallel execution across two terminal instances.

### Terminal 1: Initialize the O-RAN Controller

Activate the virtual environment and start the Python xApp:

```bash
cd ~/ns-3-dev
source ~/digital_twin_env/bin/activate
python3 vision_app_controller.py

```

*Expected Output:*

```text
==================================================
 INITIALIZING ML MODEL (Random Forest) 
 ML MODEL READY! Dynamic Inference Active.
==================================================
 6G VISION-APP xAPP (STRESS TEST MODE - NO TELEPORT) 
 Listening on Port 9999... Waiting for ns-3. 
==================================================
[Exp A (Base)] Target: 1.63 | Active gNB: 1.63 | Penalty: 0.00 dB
[Exp A (Base)] Target: 1.56 | Active gNB: 1.56 | Penalty: 0.00 dB
...

```

### Terminal 2: Execute the Digital Twin

With the Python controller actively listening, launch the ns-3 simulation in a new terminal. The default setup simulates a 28 GHz mmWave channel with 100 MHz bandwidth and a target CBR throughput of ~42.0 Mbps.

```bash
cd ~/ns-3-dev
./ns3 run "scratch/mmwave-obstacle --protocol=tcp --txPower=20.0 --enableXapp=false"

```

*Expected Output (Streaming 10ms-resolution telemetry):*

```text
==================================================
[Time: 23s] SPATIAL TOPOLOGY:
  -> Tower (gNB) Loc: X=1.63, Y=5
  -> Phone (UE)  Loc: X=2.458, Y=8.561
  -> Obstacle    Loc: X=3.989, Y=6.912
[Time: 23s] RF METRICS:
  -> Line-of-Sight  : CLEAR (LoS) [✓]
  -> ML Signal Loss : -0 dB
  -> Channel Power  : 0 dBm
SIM_THROUGHPUT,23.1,14.7117
SIM_SNR,23.1,30.8399
...

```

## Validation, Telemetry, and Output Analysis

Upon completion, the framework generates extensive telemetry characterizing the physical, MAC, and transport layers. The repository includes custom Python scripts to parse these trace files and generate comparative validation graphs matching the thesis.

### 1. End-to-End Flow Statistics (FlowMonitor)

JSON files generated natively by the ns-3 FlowMonitor module containing final application-layer statistics:

* `digital_twin_flowmon_udp.json` / `digital_twin_flowmon_tcp.json`: Baseline metrics. For UDP, expect a Packet Loss Ratio of ~7.7%, Throughput of ~39.68 Mbps, and Mean Delay of ~59.56 ms.
* `expA_replication_flowmon_tcp.json` / `expB_counterfactual_flowmon_tcp.json`: Output files comparing the baseline stationary network (Exp A) against the proactive mobile network (Exp B).

### 2. 5G-LENA Physical and MAC Layer Traces

Text files generated by 5G-LENA tracking radio frequency physics and queueing mechanics:

* `DlDataSinr.txt` / `DlCtrlSinr.txt`: Downlink SINR logs actively read by the C++ engine to export live RF metrics.
* `DlPathlossTrace.txt` / `UlPathlossTrace.txt`: Distance and blockage-induced pathloss logs combining 3GPP attenuation and the ML dynamic penalties.
* `NrDlMacStats.txt` / `NrUlMacStats.txt`: MAC layer scheduling and Modulation and Coding Scheme (MCS) adaptations.
* `NrDlRlcStatsE2E.txt` / `NrDlPdcpStatsE2E.txt`: Radio Link Control (buffer) metrics validating the strict 1.5 MB tail-drop buffer limitation and capturing post-blockage throughput impulses.
* `RxPacketTrace.txt`: Exact packet reception timestamps for microscopic delay and jitter analysis.

### 3. Transport Protocol Metrics (Kinematic Jitter Analysis)

Files isolating the cross-layer vulnerability defined as "Kinematic Jitter," where physical robotic movement tricks TCP into detecting network congestion:

* `tcp_cwnd.csv` / `baseline_tcp_cwnd.csv`: TCP Congestion Window size over time.
* `tcp_rtt.csv` / `baseline_tcp_rtt.csv`: Round Trip Time (RTT) fluctuations induced by mobility.
* `5g-traffic-*.pcap`: Raw packet capture files for Wireshark analysis.

### 4. Python Analysis and Visualization Scripts

Scripts designed to parse the raw ns-3 trace files and generate validation graphs:

* `validate_twin.py` / `plot_validation.py` / `new_validation.py`: Synchronizes the empirical ground-truth dataset (`Cleaned_Trace.csv`) with the simulation output to generate 2D spatial trajectory maps and validate the spatial ingestion engine.
* `tcp-compare.py` / `tcp_plot_new.py` / `comparision-new-tcp.py`: Parses the FlowMonitor JSONs and `tcp_cwnd.csv` traces to plot UDP versus TCP performance, highlighting the TCP Congestion Collapse caused by Kinematic Jitter.
* `plot_snr.py` / `comparison.py`: Visualizes physical-layer fidelity, plotting the moment the robotic base station hits its physical speed limit, the geometric collision, ML signal penalty application, and subsequent throughput drop.

## Author and Citation

* **Seyed Mohammad Hossein Omrani** - Master in Electrical and Computer Engineering, Faculdade de Engenharia da Universidade do Porto (FEUP).



If you utilize this code or digital twin framework for academic research, please cite the corresponding thesis:

> Omrani, S. M. H. (2026). *Trace-driven Simulation of Vision-Aided Mobile RAN Control Using ns-3 Digital Twin*. Faculdade de Engenharia da Universidade do Porto.
> 
> 

```

```
