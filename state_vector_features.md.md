# Class 1: Identity-Related Features (The Subject)

## Feature 1A: Authentication Type ($f_{auth\_Type}$)

### 1. Definition
The **Authentication Type** ($f_{authType}$) is a discrete ordinal feature that identifies the cryptographic protocol used by the **5G Core (5GC)** to verify a User Equipment (UE) identity. It serves as the **Root Trust Anchor** for the state vector $S_t$.

In the context of **Causal Reinforcement Learning**, this feature is a **Causal Antecedent**. It determines the "Baseline Integrity" of the session. By encoding the protocol directly into $S_t$, the Agent accounts for the inherent resistance of the connection to spoofing. A transition in this feature (e.g., from 5G-AKA to EAP-TLS) provides the Agent with a causal explanation for why it might now allow a more permissive access posture (e.g., moving from `RESTRICTED` to `TRUSTED`) without needing to inspect historical handshake logs.

### 2. Mathematical Formula
The feature $s_1$ is derived through a deterministic mapping function $\text{Map}(m)$ that converts the 3GPP discrete authentication method $m_{active}$ into a normalized scalar value.

$$s_1 = \text{Map}(m_{active})$$

The mapping is defined based on the hierarchical security strength of 5G protocols:

$$ \text{Map}(m_{active}) = \begin{cases} 
1.0 & \text{if } m = \text{EAP-TLS (Certificate-based, Mutual Auth)} \\
0.7 & \text{if } m = \text{5G-AKA (Symmetric Key, SIM-based)} \\
0.4 & \text{if } m = \text{EAP-AKA'} (Backward compatible 4G/5G) \\
0.0 & \text{if } m = \text{Unauthenticated / Null Auth}
\end{cases} $$

### 3. Parameter Definitions
* **$m_{active}$**: The active authentication protocol identifier (e.g., `5G_AKA`, `EAP_TLS`) currently stored in the UE Security Context within the 5G Core.
* **$s_1$**: The normalized state value ($s_1 \in [0, 1]$) representing the authentication strength in the state vector.
* **Quantization Scale**: A fixed interval (0.3) used to ensure the Neural Network's input layer can clearly distinguish the "security distance" between different enforcement requirements.

### 4. Implementation Logic
* **Source Network Function (NF)**: **AMF** (Access and Mobility Management Function) and **AUSF** (Authentication Server Function).
* **Service Interface**: **Namf_Communication** (specifically the `ueContext` retrieval) and **Nausf_UEAuthentication**.
* **Engineering Logic (Event-Driven State)**:
    1.  **Subscription-Based Update**: The Agent subscribes to the AMF via the `N12` interface for **Security Context Update** notifications.
    2.  **Immutability**: The value $s_1$ is treated as immutable for the duration of the Registration Timers. It only updates upon a re-authentication event.
    3.  **Local Caching**: To maintain the **Zero-Gap Failover** (Class 7 requirement), the current $s_1$ value is mirrored in a local high-speed state buffer to ensure $O(1)$ lookup time during the Critic's inference phase.

### 5. Detailed Calculation Workflow

#### Phase A: Initialization (UE Attachment)
1.  **Handshake Completion**: The UE and AUSF complete the primary authentication. The AMF stores the resulting `authType` (e.g., `5G_AKA`) in the UE Context.
2.  **State Extraction**: The Security Agent queries the AMF or receives an asynchronous notification containing the `authType`.
3.  **Deterministic Mapping**: The system maps the string `5G_AKA` to the value **0.7**.
4.  **Vector Injection**: The value **0.7** is written into the first position of $S_t$.

#### Phase B: Steady State (Markovian Inference)
1.  **Decision Step**: At each time step $t$, the Agent reads $s_1 = 0.7$.
2.  **Causal Reasoning**: If the Reward Function $f_{AS}$ (Attack Surface) indicates high risk, the Agent knows—based on $s_1=0.7$—that the identity is verified by a SIM card but lacks the certificate-based non-repudiation of EAP-TLS.
3.  **No History Needed**: The Agent does not need to know *when* the user logged in; the current $s_1$ value is a sufficient statistic for the current "Trust Strength."

#### Phase C: Refresh / Upgrade
1.  **Trigger**: A "Step-up Authentication" is triggered (e.g., user provides a certificate for a sensitive slice).
2.  **Notification**: The AMF notifies the Agent of the update to `EAP_TLS`.
3.  **Update**: $s_1$ transitions from **0.7** to **1.0**. The Agent immediately adjusts its posture to `TRUSTED` in the next cycle.

## Feature 1B: Credential Freshness Timer ($f_{cred\_Age}$)

### 1. Definition
The **Credential Freshness Timer** ($f_{credAge}$) is a normalized temporal metric quantifying the "age" of the current security context. It measures the duration elapsed since the last successful Primary Authentication or Key Refresh (update of the $K_{AMF}$ anchor) relative to a slice-specific expiration threshold.

In **Causal Reinforcement Learning**, this feature fulfills the **Markov Property** by transforming historical event timing into a current, measurable decay value. This allows the Agent to causally link the increased risk of "Session Hijacking" or "Token Staleness" to the current state $S_t$. As the value approaches $0.0$, the Agent is mathematically incentivized to transition the posture to `RESTRICTED` or trigger a re-authentication to minimize the **Residual Exploitation Risk** ($f_{RER}$) and satisfy **Compliance** ($f_{Pol}$).

### 2. Mathematical Formula
We use a linear decay function to map the raw time delta into a normalized scalar $f_{credAge} \in [0, 1]$.

$$f_{credAge} = \max\left(0, 1 - \frac{T_{current} - T_{last\_auth}}{T_{threshold}}\right)$$

### 3. Parameter Definitions
* **$T_{current}$**: The current system timestamp (Unix epoch) at the moment of inference.
* **$T_{last\_auth}$**: The specific timestamp of the last successful 3GPP Primary Authentication, retrieved from the UE context.
* **$T_{threshold}$**: The maximum Time-to-Live (TTL) for trust defined by the Network Slice Instance (S-NSSAI).
    * *Example:* 3600s for high-security URLLC; 86400s for standard eMBB.
* **$f_{credAge}$**: 
    * **1.0**: Trust is at maximum freshness (Handshake just completed).
    * **0.0**: Trust has reached the architectural expiration limit.

### 4. Implementation Logic
* **Source Network Function (NF)**: **AMF** (Access and Mobility Management Function).
* **Service Interface**: **Namf_Communication** (UE Context retrieval) or **Namf_EventExposure**.
* **Engineering Logic (Low-Computation Calculation)**:
    1.  **Local Delta Computation**: To minimize signaling overhead on the 5G Control Plane, the Agent does not query the AMF for the *age* at every step. It fetches the `lastAuthTimestamp` once and performs the subtraction ($T_{current} - T_{last\_auth}$) locally.
    2.  **Slice-Aware Scaling**: The $T_{threshold}$ is pulled from the Policy Control Function (**PCF**) based on the current SST (Slice Service Type).
    3.  **Deterministic Decay**: The value decreases monotonically at each time step until a re-authentication event occurs, ensuring the state transition $S_t \rightarrow S_{t+1}$ is predictable and causal.

### 5. Detailed Calculation Workflow

#### Phase A: Initialization (Authentication Event)
1.  **Trigger**: A successful Registration or Service Request procedure concludes.
2.  **Timestamp Logging**: The AMF updates the `lastAuthTimestamp`. The Agent caches this value in its local state buffer.
3.  **Normalization**: At $t=0$, $T_{current} = T_{last\_auth}$.
    $$f_{credAge} = 1 - \frac{0}{T_{threshold}} = \mathbf{1.0}$$

#### Phase B: Steady State (Inference Loop)
1.  **Snapshot**: At each inference epoch (e.g., $t = 1800s$ into a $3600s$ threshold), the Feature Extractor pulls the local system clock.
2.  **Calculation**: 
    $$f_{credAge} = 1 - \frac{1800}{3600} = \mathbf{0.5}$$
3.  **Injection**: The value **0.5** is written to the state vector. The Critic uses this to penalize "TRUSTED" actions as the value drifts toward $0.0$.

#### Phase C: Refresh and Reset
1.  **Trigger**: The Agent (or the AMF timer) triggers a UE Configuration Update or Re-authentication.
2.  **Update**: Upon the `SUCCESS` notification from the AUSF, the Agent refreshes the cached `lastAuthTimestamp`.
3.  **Reset**: $f_{credAge}$ immediately returns to **1.0**, signaling to the Agent that the "Subject" identity is once again verified and fresh.

## Feature 1C: Concurrent PDU Session Count ($f_{sess\_Count}$)

### 1. Definition
The **Concurrent PDU Session Count** ($f_{sess\_Count}$) is a non-linear, normalized metric representing the active session footprint of a SUPI (Subscription Permanent Identifier). 

In the context of **Causal Reinforcement Learning**, this feature identifies the **Entity Complexity**. By utilizing a squashing function (Sigmoid) instead of a hard linear cap, we preserve the **Causal Gradient**. This allows the agent to distinguish between "heavy usage" (e.g., 10 sessions) and "systemic compromise or signaling storms" (e.g., 100+ sessions). This distinction is vital for satisfying the **Availability Risk** ($f_{Avail}$) objective, as it enables the Agent to react with different magnitudes of enforcement to exponential session growth.

### 2. Mathematical Formula
We employ a **Logistic Sigmoid Function** to map the raw integer count $N_{sess}$ into a continuous range $(0, 1)$. This removes the dependency on a non-standardized $N_{max}$ and provides high sensitivity at critical policy transition points.

$$f_{sess\_Count} = \frac{1}{1 + e^{-k(N_{sess} - x_0)}}$$

### 3. Parameter Definitions
* **$N_{sess}$**: The raw integer count of active PDU sessions associated with the UE, retrieved from the AMF Context.
* **$x_0$ (Midpoint)**: Set to **4**. This is the inflection point ($f(x) = 0.5$). In 5G-MEC, exceeding 4 concurrent sessions typically marks the transition from a standard user to a complex/anomalous entity.
* **$k$ (Steepness)**: Set to **0.5**. This ensures the curve is sensitive enough to differentiate counts in the "danger zone" (5-20 sessions) while providing a smooth gradient for the neural network.
* **$f_{sess\_Count}$**: The final normalized scalar.
    * 1 session $\approx 0.18$
    * 11 sessions $\approx 0.97$
    * 20 sessions $\approx 0.99$

### 4. Implementation Logic
* **Source Network Function (NF)**: **AMF** (Access and Mobility Management Function).
* **Service Interface**: **Namf_Communication** (specifically the `pduSessionList` within the UE Context).
* **Engineering Logic**:
    1. **Hyperparameter Stability**: $x_0$ and $k$ are static parameters of the state-encoder. They do not require 5GC signaling, ensuring deterministic behavior across the federation.
    2. **Gradient Preservation**: Unlike linear clipping, the sigmoid is differentiable across its entire domain, which is essential for stable Actor-Critic gradient updates.
    3. **Sensitivity Mapping**: The approach is highly sensitive to the first few sessions where security policy changes are most frequent and "compresses" extreme outliers.

### 5. Detailed Calculation Workflow

#### Phase A: Initialization
1. **Discovery**: The Agent queries the AMF and finds $N_{sess} = 1$ (a standard smartphone attachment).
2. **Calculation**: 
   $$f_{sess\_Count} = \frac{1}{1 + e^{-0.5(1 - 4)}} = \frac{1}{1 + e^{1.5}} \approx \mathbf{0.18}$$

#### Phase B: The "11 vs 1000" Problem (Saturation Handling)
1. **Case 1 ($N_{sess}=11$)**:
   $$f_{sess\_Count} = \frac{1}{1 + e^{-0.5(11 - 4)}} \approx \mathbf{0.970}$$
2. **Case 2 ($N_{sess}=20$)**:
   $$f_{sess\_Count} = \frac{1}{1 + e^{-0.5(20 - 4)}} \approx \mathbf{0.999}$$
3. **Outcome**: The Agent mathematically perceives 11 as "High Risk" but still detects a delta for 20. At 1000, the value is effectively 1.0, signaling a total saturation/denial-of-service state.

#### Phase C: Dynamic Update
1. **Notification**: The AMF signals a new PDU Session Establishment via the `N11` interface.
2. **Remapping**: The new $N_{sess}$ is immediately passed through the sigmoid function. 
3. **Causal Update**: The state $S_{t+1}$ reflects the new complexity level. If the objective $f_{Res}$ (Resource Utilization) is threatened, the Agent can pivot the posture to `RESTRICTED` in the same time step.

# Class 2: Session-Related Features (The Kinetic)

## Feature 2A: Session Throughput ($\rho_{sess}$)

### 1. Definition
**Session Throughput ($\rho_{sess}$)** is a kinetic state metric that measures the **Traffic Intensity** of the PDU Session relative to its subscribed Quality of Service (QoS) parameters. It quantifies the saturation of the authorized **Session-AMBR** (Aggregate Maximum Bit Rate) using a logarithmic scale.

The metric is calculated using the following Logarithmic Ratio:

$$\rho_{sess} = \frac{\log_{10}(R_{measured} + 1)}{\log_{10}(R_{AMBR} + 1)}$$

Where:
* **$R_{AMBR}$** is the static authorized limit defined in the subscription.
* **$R_{measured}$** is the real-time bit rate derived from the 3GPP PFCP Volume Measurement:
    $$R_{measured} = \frac{(V_{uplink} + V_{downlink}) \times 8}{\Delta t}$$

In the context of the **10-Objective Vector**, this feature acts as the primary **Volumetric Anomaly Indicator**, enabling the Agent to distinguish between legitimate usage and kinetic threats that impact **Risk ($f_{PCD}$)** and **Availability ($f_{Avail}$)**.

### 2. Causal Link to Objectives
This feature provides the critical state information to maximize the following objectives:

* **Objective 3B: Post-Compromise Damage Risk ($f_{PCD}$)**
    * **Signal:** **Data Exfiltration.**
    * **Causal Logic:** A sudden spike in $\rho_{sess} \to 1.0$ (Saturation) on a non-video slice indicates an attacker attempting to offload sensitive data (e.g., database dumps) rapidly. The Agent detects this kinetic anomaly and triggers a `Quarantine` action to prevent data loss.
* **Objective 2C: Availability Risk ($f_{Avail}$)**
    * **Signal:** **Volumetric DoS.**
    * **Causal Logic:** If $\rho_{sess}$ remains consistently high without authorized traffic patterns (e.g., IoT sensor acting like a streamer), it signals a DoS participation. The Agent throttles the session to prevent UPF buffer exhaustion, protecting the slice's availability.
* **Objective 1A: Session Attack Surface ($f_{AS}$)**
    * **Signal:** **Low-and-Slow Scanning.**
    * **Causal Logic:** A low but constant non-zero log value indicates "beaconing" or port scanning (Reconnaissance) rather than true idleness, prompting the Agent to tighten topological restrictions.

### 3. Parameter Definitions
* **$V_{uplink}$**: The value of the `Total Uplink Volume` field (measured in **Octets**) extracted from the **Volume Measurement** IE (Type 153) in the PFCP Session Report.
* **$V_{downlink}$**: The value of the `Total Downlink Volume` field (measured in **Octets**) extracted from the same IE.
* **$\Delta t$**: The measurement window duration (in seconds), strictly defined by the **Measurement Period** field in the **Usage Reporting Rule (URR)**.
* **$R_{AMBR}$**: The **Session-Aggregate Maximum Bit Rate** (bps), defined in **3GPP TS 23.501**. It represents the maximum bit rate that can be expected across all Non-GBR QoS Flows of a specific PDU Session.
* **$\rho_{sess}$**: A normalized scalar $[0, 1]$.
    * **~0.0**: Idle (Traffic Volume = 0).
    * **~0.1 - 0.3**: Keep-Alive / Beaconing (Potential C2 activity).
    * **~0.6**: Nominal Load (User Plane traffic within expected bounds).
    * **~0.99**: Saturation (Session is hitting the policer limit).

### 4. Implementation Logic
* **Source Network Function:** **UPF** (User Plane Function).
* **Service Interface:** **N4 Interface** (PFCP - Packet Forwarding Control Protocol).
* **Engineering Logic:** The Agent correlates dynamic telemetry from the Data Plane (UPF) with static subscription context from the Control Plane (SMF/UDM).



### 5. Detailed Calculation Workflow (3GPP Standards Based)
1.  **Context Retrieval (TS 23.502):**
    * Upon PDU Session Establishment, the Agent extracts the **`Session-AMBR`** (e.g., `uplink: 1 Gbps`, `downlink: 1 Gbps`) from the **Subscription Data** retrieved by the SMF from the UDM.
2.  **URR Configuration (TS 29.244):**
    * The SMF provisions a **Usage Reporting Rule (URR)** IE on the UPF with:
        * `Measurement Method` = **Volume** (Bit 0 set).
        * `Reporting Trigger` = **Periodic Reporting** (Bit 1 set).
        * `Measurement Period` = **2000ms** (2 seconds).
3.  **Telemetry Ingestion (TS 29.244):**
    * The UPF generates a **PFCP Session Report Request** message.
    * The Agent parses the message to find the **`Usage Report`** IE.
    * Inside the Usage Report, it locates the **`Volume Measurement`** IE (Type 153).
4.  **Bit Rate Derivation:**
    * It extracts the integer values for `Total Uplink Volume` and `Total Downlink Volume`.
    * It sums these values (Octets), multiplies by 8 (Bits), and divides by the URR `Measurement Period` ($\Delta t$) to obtain $R_{measured}$.
5.  **Normalization:**
    * The Agent applies the logarithmic formula: $\rho_{sess} = \log(R_{measured}) / \log(R_{AMBR})$.
    * This final value is fed into the Neural Network's Kinetic Input Head.

## Feature 2B: Flow Destination Entropy ($s_4$)

### 1. Definition
**Flow Destination Entropy** is a kinetic measure of **Target Dispersion**. It analyzes the "fan-out" behavior of a PDU Session to determine if the device is adhering to Zero Trust segmentation principles or attempting to move laterally.

* **The Variable ($x_i$):** We measure the distribution of unique **Destination IP Addresses** (e.g., `10.0.1.5`, `172.16.0.1`) contacted by the session.
* **Normal Behavior (Low Entropy):** A legitimate IoT sensor or specific app usually talks to a small, repeated set of endpoints. The session is "Focused."
* **Attack Behavior (High Entropy):** During reconnaissance (Network Scanning), a device attempts to connect to *many* random IPs. The session is "Scattered."

### 2. Mathematical Formula
We calculate the **Normalized Shannon Entropy ($H_{norm}$)** of the destination IP distribution over a rolling time window $W$.

$$s_4 = \frac{H_{norm} - \mu_{slice}}{\sigma_{slice} + \epsilon}$$

Where the Normalized Entropy for the current window is:

$$H_{norm} = \frac{- \sum_{i=1}^{N} p_i \cdot \log_2(p_i)}{\log_2 N}$$

*(Note: If $N=1$, then $H_{norm} = 0$. If $N > 1$ but $\log_2 N = 0$, handle as 0).*

### 3. Parameter Definitions
* **$x_i$ (The Target):** A unique Destination IP Address observed in the session flows.
* **$p_i$ (Target Probability):** The fraction of total packets in the current window sent to target $x_i$:
    $$p_i = \frac{\text{Packets to IP } x_i}{\text{Total Packets in Window}}$$
* **$N$:** The count of *unique* Destination IPs contacted in the window.
* **$\mu_{slice}$ & $\sigma_{slice}$ (Global Slice Constants):**
    * These are **Global Statistics** stored in the RL Agent's memory for each Slice Type (SST).
    * **Low Overhead Logic:** They are **Read-Only** during real-time detection. They are updated only periodically (Offline Learning) using historical data from trusted sessions.

### 4. Implementation Logic (The "Zero-Overhead" Strategy)

To prevent latency in the 5G Data Plane, we separate **Detection** from **Learning**.

#### Phase A: Real-Time Detection (Fast Path)
* **Location:** UPF or Edge Agent.
* **Action:** When a session needs to be scored, the Agent performs a simple **Table Lookup**.
    * *Lookup:* `GET Stats FROM Slice_Table WHERE SliceID = "MIoT"`
    * *Return:* `Mean=0.05`, `StdDev=0.02`.
* **Calculation:** The Agent computes the formula $s_4 = (H - 0.05) / 0.02$.
* **Cost:** Negligible ($O(1)$).

#### Phase B: Background Learning (Slow Path)
* **Location:** NWDAF (Analytics Function) or Central Controller.
* **Action:** Once every $T_{train}$ (e.g., 1 hour), the system refines the baselines using **Welford’s Algorithm** on a batch of *verified benign* logs.
* **Update Logic:**
    1.  Collect logs from 1,000 trusted sessions (clean history).
    2.  Calculate average Entropy ($H_{avg}$) for this batch.
    3.  Update the Global Table:
        * $\mu_{new} = (1 - \alpha) \cdot \mu_{old} + \alpha \cdot H_{avg}$
* **Why efficient:** This happens asynchronously and never blocks user traffic.

### 5. Detailed Calculation Workflow (Example)

**Step A: Data Collection (In the Window)**
* **Context:** A PDU Session on Slice `MIoT_01`.
* **Traffic Log (Last 10s):**
    1.  `192.168.1.50` (Main Server) - 90 packets
    2.  `192.168.1.50` (Main Server) - 10 packets (Keep-alive)
    3.  `10.0.0.1` (DNS Server) - 5 packets
    4.  `192.168.1.200` (Unknown IP) - 2 packets
    5.  `192.168.1.201` (Unknown IP) - 2 packets

**Step B: Calculate Probabilities ($p_i$)**
* **Total Packets:** $90 + 10 + 5 + 2 + 2 = 109$.
* **Unique IPs ($N$):** 4 targets (`...1.50`, `...0.1`, `...1.200`, `...1.201`).
* **Distribution:**
    * $p_{server} = 100/109 \approx 0.917$
    * $p_{dns} = 5/109 \approx 0.046$
    * $p_{scan1} = 2/109 \approx 0.018$
    * $p_{scan2} = 2/109 \approx 0.018$

**Step C: Compute Entropy ($H_{norm}$)**
1.  **Raw Entropy ($H$):**
    $$H = - (0.917 \log_2 0.917 + 0.046 \log_2 0.046 + 2 \times 0.018 \log_2 0.018)$$
    $$H \approx - (-0.11 - 0.20 - 0.21) = \mathbf{0.52 \text{ bits}}$$
2.  **Normalize:**
    Max Entropy ($N=4$) is $\log_2 4 = 2.0$.
    $$H_{norm} = 0.52 / 2.0 = \mathbf{0.26}$$

**Step D: Standardization ($s_4$)**
1.  **Fast Lookup:**
    * Agent checks `MIoT_Table`. Found: $\mu=0.05, \sigma=0.02$. (No calculation, just read).
2.  **Calculate Feature:**
    $$s_4 = \frac{0.26 - 0.05}{0.02} = \mathbf{+10.5}$$
3.  **Result:** The Z-score of **+10.5** flags the session as an anomaly (Scanning/Lateral Movement).


## Feature 2C: Session Policy Violation Rate ($s_5$)

### 1. Definition
**Session Policy Violation Rate ($s_5$)** is a kinetic measure of the **Authorization Stability** of the PDU Session. It quantifies the frequency with which the session attempts actions that are explicitly rejected by the active Zero Trust Policy.

* **Zero Violations ($s_5 = 0$):** A compliant session. The User Equipment (UE) only sends traffic that matches the active **"Allow-List"**.
* **High Violations ($s_5 \to 1$):** A compromised session. The UE is scanning the network, hitting "Implicit Deny" rules in the Firewall or attempting unauthorized API calls.

This feature serves as the primary sensor for **Objective 1C ($f_{LPE}$ - Least Privilege Enforcement)** and **Objective 3B ($f_{PCD}$)**.

### 2. Mathematical Formula
We calculate the ratio of **Blocked Events** to **Total Events** observed for this Session ID.

$$s_5 = \frac{N_{dropped} + N_{4xx}}{N_{total} + \epsilon}$$

### 3. Parameter Definitions
* **$N_{total}$:** Total packet flows or requests initiated by the session in the window.
* **$N_{dropped}$ (L3/L4 Violations):** The count of packets dropped by the UPF because they did not match any allowed traffic descriptor (Policy: "Implicit Deny").
* **$N_{4xx}$ (L7 Violations):** The count of Application Layer errors (HTTP 401/403) reported by the Application Function (AF) or Service Proxy.
* **$\epsilon$:** Small constant ($10^{-6}$) to ensure stability.

### 4. Implementation Logic (3GPP Standards Based)

To implement this without breaking 3GPP architecture, we fuse data from the **UPF (User Plane)** and the **AF (Application Plane)**.

#### Path A: Network Layer Violations (UPF - TS 29.244)
* **Mechanism:** We use the **"Match-All-Drop"** pattern in PFCP.
* **Configuration:**
    1.  **Allow Rules:** The SMF installs PDRs with high precedence for allowed traffic (e.g., PDR-1: `Match: DST=10.0.0.1, Port=443`, `Action: FWD`).
    2.  **Catch-All Rule:** The SMF installs a "Floor" PDR with lowest precedence (e.g., PDR-255: `Match: Any`, `Action: DROP`, `URR-ID: Count_Drops`).
* **Detection:** Any packet that *fails* to match the Allow Rules falls through to the Catch-All PDR. The attached **URR (Usage Reporting Rule)** counts these drops and reports them to the Agent.



#### Path B: Application Layer Violations (AF via NEF/NWDAF)
* **Mechanism:** The **Service Mesh Sidecar** acts as a trusted **Application Function (AF)**.
* **Reporting:** The AF detects HTTP 403s and pushes a summary event to the **NWDAF** (Analytics Function) via the **Nnef_EventExposure** service.
* **Fusion:** The Agent subscribes to the NWDAF to receive the `$N_{4xx}$` counter.

### 5. Detailed Calculation Workflow (Data Fusion Example)

**Step A: Activity Monitoring (10s Window)**
* **Context:** Compromised Session `ID: 0x123`.
* **UPF Telemetry (Network Layer):**
    1.  `DST: 10.0.0.1 (Web Server)` $\to$ **Match PDR-1 (Allow)** $\to$ 4 packets.
    2.  `DST: 192.168.1.5 (Database)` $\to$ **Match PDR-255 (Drop)** $\to$ 1 packet.
    3.  `DST: 192.168.1.6 (Backup)` $\to$ **Match PDR-255 (Drop)** $\to$ 1 packet.
    * *UPF Counters:* $N_{total\_net} = 6$, $N_{dropped} = 2$.
* **AF Telemetry (Application Layer):**
    1.  `GET /index.html` $\to$ **200 OK** $\to$ 1 request.
    2.  `GET /admin/config` $\to$ **403 Forbidden** $\to$ 3 requests.
    * *AF Counters:* $N_{total\_app} = 4$, $N_{4xx} = 3$.

**Step B: Data Aggregation**
* **Total Events ($N_{total}$):** $6 \text{ (Net)} + 4 \text{ (App)} = \mathbf{10}$.
* **Total Violations:** $2 \text{ (Drops)} + 3 \text{ (403s)} = \mathbf{5}$.

**Step C: Feature Calculation ($s_5$)**
$$s_5 = \frac{2 + 3}{10 + 10^{-6}} = \frac{5}{10} = \mathbf{0.5}$$

**Step D: RL State Interpretation**
* **Value:** **0.5** (50% Violation Rate).
* **Meaning:** Half of everything this device attempts is illegal. This is a definitive signal of an active compromise (Script/Bot).
* **Agent Reaction:** The RL Agent correlates this with **Objective 3A ($f_{RER}$)** and immediately executes a **Block** action.

## Feature 2D: Session East-West Traffic Ratio ($s_6$)

### 1. Definition
**Session East-West Traffic Ratio ($s_6$)** is a kinetic state metric measuring **Traffic Directionality**. It quantifies the proportion of a session's traffic that attempts to communicate laterally within the local network (East-West) versus traffic exiting to the Gateway/Internet (North-South).



* **$s_6 \to 0.0$ (North-South):** The session is communicating strictly with the Internet or authorized Gateways (e.g., Video Streaming, Cloud App). This is the expected behavior for 99% of mobile UEs.
* **$s_6 \to 1.0$ (East-West):** The session is actively attempting to contact other internal IPs. In a "Guest" or "IoT" slice, this is a high-confidence signal of **Lateral Movement** (Scanning, Pivoting, or Worm propagation).

#### ⚠️ Key Distinction from Feature 2C ($s_5$)
While **Feature 2C ($s_5$)** measures **Rejection** (packets blocked by policy), this feature measures **Direction** (packets moving sideways).
* A hacker "brute-forcing" a password generates **High $s_5$** (Errors).
* A hacker "scanning" the local network for open ports generates **High $s_6$** (Lateral Flow), even if the packets are technically allowed by the firewall.

### 2. Mathematical Formula
We model $s_6$ as a weighted ratio of lateral packets, with an immediate "Saturation Trigger" for forbidden zone attempts.

$$s_6 = \min\left(1.0, \quad \alpha \cdot \frac{N_{lat}}{N_{total} + \epsilon} + \beta \cdot \mathbb{I}_{ZoneViolation} \right)$$

### 3. Parameter Definitions
* **$N_{total}$:** Total outbound packets sent by this Session ID in the observation window.
* **$N_{lat}$ (Lateral Count):** The count of packets matching the **"Lateral Trap" PDR**. These are packets destined for Private IP ranges (RFC 1918, e.g., `10.0.0.0/8`) that are *not* explicitly on the Allow List.
* **$\mathbb{I}_{ZoneViolation}$ (Binary Indicator):**
    * Returns **1.0** if *any* packet matches a **"Blacklist" PDR** (e.g., trying to reach the Management Subnet `10.50.0.0/24`).
    * Returns **0.0** otherwise.
* **Weights:**
    * **$\alpha = 0.8$:** Volume sensitivity. High lateral scanning pushes the score to 0.8.
    * **$\beta = 1.0$:** Critical penalty. A single attempt to cross a prohibited security zone saturates the score to **1.0 (Max Risk)**.
* **$\epsilon$:** Stability constant ($1$).

### 4. Implementation Logic (The "3-Rule Sieve")
To ensure **Scalability** and **3GPP Compliance**, we do *not* enumerate every forbidden IP. Instead, we use **PDR Precedence** in the UPF to create a logic funnel (Sieve) that traps lateral movement automatically.



* **Source Network Function:** UPF (User Plane Function).
* **Protocol:** N4 Interface (PFCP).
* **Configuration (Per Slice):**
    1.  **PDR-1 (Priority: 10) - The Allow List:**
        * **Match:** Specific authorized IPs (e.g., `Dst=10.5.5.5` NVR Server).
        * **Action:** FORWARD.
        * **Metric:** Ignored (Healthy Traffic).
    2.  **PDR-2 (Priority: 50) - The Lateral Trap:**
        * **Match:** Broad Internal CIDRs (e.g., `Dst=10.0.0.0/8`, `192.168.0.0/16`).
        * **Action:** DROP (or Forward to Honeypot).
        * **Metric:** Increments **$N_{lat}$** (Violation).
    3.  **PDR-3 (Priority: 100) - The Default Gateway:**
        * **Match:** `Dst=0.0.0.0/0` (Internet).
        * **Action:** FORWARD.
        * **Metric:** Increments **$N_{total}$**.

### 5. Calculation Example (The "Split Personality" UE)

**Scenario:** A User Equipment (UE) has two active PDU Sessions.
* **Session A (Slice: Voice):** Making a VoLTE call.
* **Session B (Slice: Internet):** Infected with malware, scanning the local network.

**Calculation for Session A (Voice):**
* **Traffic:** UDP packets to `208.x.x.x` (Carrier Gateway).
* **Logic:** Matches PDR-3 (Internet/Default).
* **Counters:** $N_{lat} = 0, \mathbb{I} = 0$.
* **Result:** $s_6 = 0.0$ (Healthy).

**Calculation for Session B (Infected):**
* **Traffic Stream:**
    1.  80 packets to `192.168.0.5` (Local Printer) $\to$ Matches **PDR-2** ($N_{lat}$ increases).
    2.  20 packets to `8.8.8.8` (Google DNS) $\to$ Matches **PDR-3** ($N_{total}$ increases).
    3.  1 packet to `10.50.0.1` (Admin Server) $\to$ Matches **PDR-1 of Blacklist** ($\mathbb{I}=1$).
* **Result:**
    $$s_6 = \min(1.0, \quad 0.8 \cdot \frac{80}{101} + 1.0 \cdot 1) = \min(1.0, \quad 0.63 + 1.0) = \mathbf{1.0}$$

**Outcome:** The Agent sees $s_6=1.0$ for **Session B** and terminates it immediately. **Session A** is untouched, preserving the user's phone call.

## Feature 2E: Active Microsegment Cost ($s_7$)

### 1. Definition
**Active Microsegment Cost ($s_7$)** is a static state metric representing the **Computational and Operational Overhead** inherent to the Microsegment where the session is currently placed.

In this Zero Trust architecture, every **Microsegment** enforces a specific stack of security functions. This feature quantifies the "Price" (CPU/Latency tax) of that stack.

* **$s_7 \approx 0.0$ (Drop/Deny):** The session is blocked. Resource usage is minimal.
* **$s_7 \approx 0.2$ (Lightweight):** The session is in a trusted zone with standard monitoring.
* **$s_7 \to 1.0$ (Heavyweight):** The session is in a high-security zone (Quarantine) requiring deep inspection, proxying, or sandboxing.

### 2. Mathematical Formula
We use a **Deterministic Lookup** based on the current Microsegment ID ($Microseg_{ID}$).

$$s_7 = CostMap(Microseg_{ID})$$

### 3. Parameter Definitions
* **$Microseg_{ID}$:** The identifier of the Microsegment currently assigned to this session.
* **$CostMap$:** A configuration vector defining the relative resource tax of each microsegment:
    * **DENY ($0.05$):** Packet Drop. (Cost is near zero, just the CPU cycle to match the drop rule).
    * **TRUSTED ($0.20$):** Standard L3/L4 Routing + Basic Logging. (Fast Path).
    * **RESTRICTED ($0.50$):** L7 Firewalling + DPI (Deep Packet Inspection) for specific protocols.
    * **QUARANTINE ($0.90$):** Full Proxy + Behavior Analysis + Full Packet Capture. (Maximum CPU/Latency impact).

### 4. Integration with RL Logic
This feature provides the **"Cost Context"** to the Neural Network:
* The Agent combines **Feature 2A ($\rho_{sess}$ - Volume)** and **Feature 2E ($s_7$ - Cost)**.
* **Learning Outcome:** The Agent learns that placing a high-volume stream ($\rho_{sess}=0.9$) into **QUARANTINE** ($s_7=0.9$) is extremely expensive for the infrastructure (Negative Reward on $f_{Res}$), whereas placing it in **TRUSTED** ($s_7=0.2$) is efficient.

# Class 3: Device Posture & Vulnerability Features (The Asset)

## Feature 3A: Device Health State ($s_8$)

### 1. Definition
**Device Integrity Level** is a deterministic, stateless measure of **Endpoint Hygiene**. It quantifies the "Trustworthiness" of the physical hardware connecting to the network by fusing three critical security signals: **Root Privilege Status**, **OS Vulnerability Severity**, and **Application Safety**.

In the Causal RL framework, this feature acts as the **Vulnerability Coefficient**. It normalizes diverse risk factors into a single scalar value $\in [0, 1]$ without requiring historical data or statistical baselines.

* **$s_1 \approx 0.0$ (Hardened):** A pristine, fully patched device with a locked bootloader running only managed apps.
* **$s_1 \to 1.0$ (Compromised):** A high-risk device (e.g., Rooted, unpatched Baseband vulnerabilities, or malware detected) that requires immediate quarantine.

### 2. Mathematical Formula
We model $s_1$ as a **Weighted Risk Sum** of three normalized inputs. This ensures the output is always bounded between 0 and 1, facilitating stable Neural Network convergence.

$$s_1 = \underbrace{w_{root} \cdot \mathbb{I}_{Root}}_{\text{Privilege Integrity}} + \underbrace{w_{vuln} \cdot \left( \frac{RS_{vuln}}{10.0} \right)}_{\text{Patch Health}} + \underbrace{w_{app} \cdot \Omega_{app}}_{\text{App Hygiene}}$$

Where the **Vulnerability Risk Score ($RS_{vuln}$)** is calculated using a **Weighted Average** of all discovered CVEs on the device:

$$RS_{vuln} = \frac{\sum_{i=1}^{n} (CVSS_i \cdot K_i)}{\sum_{i=1}^{n} K_i}$$

### 3. Parameter Definitions & Calculation Logic

#### A. The Aggregation Weights ($w$)
These fixed weights define the "Security Policy" importance of each factor. They must sum to **1.0**.
* **$w_{root} = 0.5$ (Critical):** The weight assigned to the Root/Jailbreak status. Bypassing OS privilege controls is the single most significant security risk.
* **$w_{vuln} = 0.3$ (Major):** The weight assigned to the aggregated Vulnerability Score.
* **$w_{app} = 0.2$ (Moderate):** The weight assigned to the Application Safety score.

#### B. Variable 1: Root Status ($\mathbb{I}_{Root}$)
* **Definition:** A Binary Indicator of privilege escalation.
* **Source:** MDM "Compromised Status" flag or Attestation API (Play Integrity / DeviceCheck).
* **Values:**
    * **1.0:** Device is Rooted, Jailbroken, or fails Attestation.
    * **0.0:** Device is Locked and Verified.

#### C. Variable 2: Vulnerability Score ($RS_{vuln}$)
* **$CVSS_i$:** The **Common Vulnerability Scoring System** Base Score (0.0 - 10.0) for the $i$-th vulnerability found on the device.
* **$K_i$ (5G Context Weights):** Multipliers adjusting importance based on the **Attack Vector**:
    * **$K = 10.0$ (Baseband/Modem):** Critical. Bugs in the Radio Stack (LTE/NR) can be exploited Over-The-Air (OTA).
    * **$K = 5.0$ (Kernel/OS):** High. Bugs in the OS Kernel allow privilege escalation.
    * **$K = 3.0$ (Local Connectivity):** Medium. Bugs in WiFi, Bluetooth, or NFC stacks.
    * **$K = 1.0$ (User Space):** Low. Bugs in standard apps (e.g., Calculator).

#### D. Variable 3: Application Risk ($\Omega_{app}$) **(DETAILED)**
We calculate this by checking the device's **App Inventory** against a **Threat Policy**. We use a **"Highest Severity Wins"** logic:

$$\Omega_{app} = \max \left( \mathbb{I}_{Malware}, \quad \mathbb{I}_{Blacklist}, \quad 0.5 \cdot \mathbb{I}_{Sideload} \right)$$

* **$\mathbb{I}_{Malware}$ (Value 1.0):**
    * **Condition:** The MDM/EDR agent detects a known malware signature (SHA-256 match) or a "Potentially Unwanted Program" (PUP) classified as `High Severity`.
* **$\mathbb{I}_{Blacklist}$ (Value 1.0):**
    * **Condition:** The device has an app installed that is explicitly forbidden by the organization (e.g., `com.termux`, `com.wireshark`, or known VPN bypass tools).
* **$\mathbb{I}_{Sideload}$ (Value 1.0):**
    * **Condition:** The Android setting `"Install Apps from Unknown Sources"` is ENABLED, or the iOS profile is `"Unmanaged"`.
    * *Note:* In the formula, this is multiplied by **0.5**, meaning a sideloading configuration contributes **0.5** risk (Medium).

### 4. Implementation Logic (3GPP Architecture)

* **Architecture:** Control Plane Integration via **Network Exposure Function (NEF)**.
* **Input Source:** The Enterprise **Application Function (AF)** acting as the connector to the MDM (e.g., Microsoft Intune, VMware Workspace ONE).
* **Service Interface:** `Nnef_EventExposure_Subscribe` / `Notify`.
* **Scope:** **Global (User Context)**. The value is calculated once per device and inherited by all active PDU Sessions (Internet, Voice, Corporate).

### 5. Detailed Implementation Action Plan

**Phase 1: Integration & Data Ingestion (AF $\to$ NEF)**
1.  **MDM Query:** The Enterprise AF queries the MDM agent on the device.
2.  **Standardized Payload:** The AF formats the response into a JSON Object and pushes it to the NEF.
    ```json
    {
      "supi": "123456789",
      "flags": {
        "isRooted": false,
        "hasMalware": false,
        "hasBlacklistedApp": true,  // Trigger found
        "unknownSourcesEnabled": false
      },
      "cveList": [
        {"id": "CVE-2024-001", "cvss": 9.8, "component": "BASEBAND"},
        {"id": "CVE-2023-055", "cvss": 5.3, "component": "BLUETOOTH"}
      ]
    }
    ```

**Phase 2: The Calculation Routine (NWDAF / Agent)**
1.  **Parse Root Status:** `Root` = 0.0.
2.  **Calculate App Risk ($\Omega_{app}$):**
    * `Malware` = 0.
    * `Blacklist` = 1.0 (True).
    * `Sideload` = 0.
    * **Result:** $\max(0, 1.0, 0) = \mathbf{1.0}$.
3.  **Calculate Vulnerability Risk ($RS_{vuln}$):**
    * Weighted Sum: $(9.8 \times 10.0) + (5.3 \times 3.0) = 98.0 + 15.9 = 113.9$.
    * Total Weights: $10.0 + 3.0 = 13.0$.
    * **Result:** $113.9 / 13.0 \approx \mathbf{8.76}$.

**Phase 3: Final Aggregation ($s_1$)**
1.  **Formula Application:**
    $$s_1 = (0.5 \times 0.0) + \left(0.3 \times \frac{8.76}{10.0}\right) + (0.2 \times 1.0)$$
    $$s_1 = 0.0 + 0.2628 + 0.2$$
    $$s_1 \approx \mathbf{0.46}$$
2.  **State Injection:** The value **0.46** is stored in the UDR and injected into state $s_1$ for every session this user initiates.

### 6. RL Interpretation
* **Value:** **0.46** (Moderate-High Risk).
* **Meaning:** "Device is not rooted, but has severe patching issues AND prohibited software."
* **Agent Reaction:** The RL Agent will likely block access to "Confidential" slices (due to the App Risk) but might allow "Internet" access with rate limiting.

## Feature 3B: Radio Access Trust Level ($s_3$)

### 1. Definition
**Radio Access Trust Level** is a deterministic state metric that quantifies the **Transport Layer Security** of the physical medium connecting the User Equipment (UE) to the Network.

In a 3GPP Hybrid environment, the "Pipe" itself is a variable risk factor. This feature acts as a **Link Integrity Sensor**.
* **$s_3 \approx 0.0$ (Secure):** User is connected via **5G Standalone (SA)**. The air interface uses 256-bit encryption (NEA3) and is integrity protected.
* **$s_3 \to 1.0$ (Hostile):** User is connected via **Untrusted Wi-Fi** (N3IWF) or has been downgraded to legacy **2G/3G** (indicating a possible IMSI Catcher attack).

### 2. Mathematical Formula
We model $s_3$ as a direct mapping function of the active **RAT (Radio Access Technology)** reported by the Core Network.

$$s_3 = \text{Map}(RAT_{Current})$$

### 3. Parameter Definitions & Calculation Logic
We define a fixed **Risk Lookup Table** based on the 3GPP `AccessType` and `RATType` enums.

| RAT Type | Value ($s_3$) | Security Justification |
| :--- | :--- | :--- |
| **NR (5G SA)** | **0.1** | **Gold Standard.** Native 5G Core. Supports Slice Isolation, SUCI (Identity Privacy), and full Integrity Protection. |
| **NR (5G NSA)** | **0.3** | **Silver Standard.** Uses 5G Radio but 4G Core (EPC). Inherits some 4G signaling vulnerabilities. |
| **E-UTRA (4G)** | **0.5** | **Baseline.** Strong AES encryption, but lacks SUCI (IMSI can be caught) and has no granular Slice protection. |
| **Untrusted Wi-Fi** | **0.9** | **High Risk.** Traffic is tunneled via N3IWF (IPSec). Data is encrypted, but the physical link is shared, jam-able, and allows traffic analysis. |
| **GERAN/UTRA (2G/3G)** | **1.0** | **Critical Risk.** These protocols have broken encryption (A5/1) and no mutual authentication. Presence here implies a **Downgrade Attack**. |

### 4. Implementation Logic (3GPP Architecture)
* **Architecture:** The **AMF (Access and Mobility Management Function)** owns the knowledge of *how* a user is connected.
* **Input Source:** The AMF triggers an event whenever the RAT Type changes (Handover).
* **Service Interface:** `Namf_EventExposure_Subscribe` / `Notify`.
* **Standard:** **3GPP TS 23.502**.
* **Global vs Local:** This is a **Global** feature. If the radio link is weak, *all* concurrent sessions (Voice, Web, Admin) are equally at risk.

### 5. Detailed Implementation Action Plan

**Phase 1: Subscription (Agent Setup)**
1.  **Configure:** The RL Agent (via NWDAF) subscribes to the AMF for a specific user (SUPI).
2.  **Event ID:** `ACCESS_TYPE_REPORT` and `LOCATION_REPORT`.
3.  **Rationale:** We need to know immediately when the phone switches from Tower to Wi-Fi or Tower to Tower.

**Phase 2: The Handover Event (Data Ingestion)**
* **Scenario:** User walks from the street (5G) into a coffee shop (Public Wi-Fi). The phone connects to the Wi-Fi and establishes an IPSec tunnel to the **N3IWF**.
* **Payload:** The AMF sends this JSON to the Agent:
    ```json
    {
      "supi": "123456789",
      "timeStamp": "2024-05-20T10:00:00Z",
      "accessType": "NON_3GPP_ACCESS",
      "ratType": "UNTRUSTED_NON_3GPP",
      "reason": "HANDOVER"
    }
    ```

**Phase 3: Calculation & Injection (The "Compute" Step)**
1.  **Parsing:** Extract `ratType`.
2.  **Lookup:**
    * Key: `UNTRUSTED_NON_3GPP`
    * Map Value: **0.9**
3.  **Validation:** Is this allowed?
    * *Policy Check:* "Does the user allow Wi-Fi access?"
    * If `Yes`, proceed.
4.  **Injection:** Update state $s_3 = 0.9$.

**Phase 4: Agent Reaction (The Outcome)**
* The RL Agent receives $s_3 = 0.9$ in the next time step.
* **Inference:** "The link is now hostile."
* **Action:**
    * **Action A:** Maintain the connection (Session Continuity).
    * **Action B:** Apply QoS throttling (Reduce bit rate to account for Wi-Fi instability).
    * **Action C:** Revoke access to **Class 1 Data** (Top Secret) until `ratType` returns to `NR` (5G).

# Class 4: Network & Topology-Related Features (The Map)

## Feature 4A: Service Criticality Level ($s_7$)

### 1. Definition
**Service Criticality Level** is a deterministic, stateless measure of the **Data Sensitivity** flowing through the network pipe. It quantifies the "Value at Risk" based on the **Network Slice** and **Target Network** (DNN), independent of the user's identity.

In the Causal RL framework, this feature acts as the **Impact Coefficient**. It informs the Agent of the potential damage cost of a breach.
* **$s_7 \approx 0.0$ (Commodity):** Low-value traffic (e.g., Public Internet, Guest Wi-Fi).
* **$s_7 \to 1.0$ (Mission Critical):** Life-safety or high-value financial data (e.g., Remote Surgery, SCADA Control).



### 2. Mathematical Formula
We model $s_7$ as the output of a **Hierarchical Policy Lookup Function** ($L$).

$$s_7 = L(S_{sst}, \ S_{sd}, \ D_{nn})$$

The function $L$ iterates through a prioritized configuration set $\mathcal{T}$ (The Trust Table) to find a match:

$$L(\dots) = \begin{cases} w_k & \text{if } \exists k \in \mathcal{T} : (S_{sst}, S_{sd}, D_{nn}) \text{ matches Rule}_k \\ w_{default} & \text{otherwise} \end{cases}$$

### 3. Parameter Definitions & Calculation Logic

#### A. The Input Variables (from 5G Core)
These are dynamic variables retrieved from the **SMF (Session Management Function)** at the start of a session.

* **$S_{sst}$ (Slice/Service Type):** An integer $[0-255]$ representing the broad category of the slice.
    * *Standard values:* 1 (eMBB/Broadband), 2 (URLLC/Critical), 3 (MIoT/IoT).
* **$S_{sd}$ (Slice Differentiator):** A 24-bit Hex string (e.g., `FFFFFF`) used to identify a specific tenant or use-case within a slice type.
    * *Note:* If no differentiator is assigned, this defaults to `null`.
* **$D_{nn}$ (Data Network Name):** A string identifier for the external network gateway (equivalent to APN). Examples: `"internet"`, `"ims"`, `"factory-sec"`.

#### B. The Hyperparameters (The Configuration)
These are static constants defined by the Network Administrator in the **Policy Table $\mathcal{T}$**.

* **$\mathcal{T}$ (The Trust Table):** A set of $k$ rules, where each rule is a tuple: $\langle SST, SD, DNN, w \rangle$.
* **$w_k$ (Rule Weight):** A scalar value $\in [0, 1]$ assigned to a specific rule, representing the risk level of that traffic profile.
* **$w_{default}$:** The fallback risk score applied if no rule matches the inputs. We set this to **0.5** (Neutral) to prevent the Agent from assuming unknown traffic is perfectly safe.

#### C. The Matching Logic (The "Hierarchy")
The function $L$ checks for matches in this specific order of precedence (Specific $\to$ Generic):

1.  **Exact Match:** Matches all three inputs $(S_{sst}, S_{sd}, D_{nn})$.
    * *Example:* `(2, FFFFFF, "remote-surgery")` $\to$ **$w = 1.0$**
2.  **Tenant Match:** Matches Slice and SD, ignores DNN $(S_{sst}, S_{sd}, *)$.
    * *Example:* `(1, A0001, *)` (All traffic from the Finance Dept) $\to$ **$w = 0.8$**
3.  **Broad Slice Match:** Matches only the Slice Type $(S_{sst}, *, *)$.
    * *Example:* `(1, *, *)` (All Public Broadband) $\to$ **$w = 0.1$**

### 4. Implementation Logic (3GPP Architecture)
* **Architecture:** Control Plane Integration via **SMF**.
* **Input Source:** The SMF assigns the Slice and DNN during the **PDU Session Establishment** procedure.
* **Service Interface:** `Nsmf_PDUSession_Create` (at setup) or `Namf_EventExposure` (for monitoring).
* **Standard:** **3GPP TS 23.501** (System Architecture for 5G).

### 5. Detailed Implementation Action Plan

**Phase 1: Policy Ingestion ($t < 0$)**
1.  **Admin Action:** The Security Architect defines `criticality_policy.json`.
    ```json
    [
      {"sst": 2, "sd": "FFFFFF", "dnn": "remote-surgery", "weight": 1.0},
      {"sst": 1, "sd": "A0001", "dnn": "*", "weight": 0.8},
      {"sst": 1, "sd": "*", "dnn": "*", "weight": 0.1}
    ]
    ```
2.  **Agent Action:** The RL Agent loads this into memory as the Table $\mathcal{T}$.

**Phase 2: Session Establishment ($t = 0$)**
1.  **Event:** A new PDU Session is created.
2.  **Data Capture:** The Agent receives the session attributes from the SMF:
    * $S_{sst} = 1$
    * $S_{sd} = \text{"A0001"}$ (Finance Dept)
    * $D_{nn} = \text{"google-services"}$

**Phase 3: Calculation Routine ($L$)**
1.  **Check Rule 1 (Exact):** Does `(1, A0001, google-services)` match `(2, FFFFFF, remote-surgery)`? **NO**.
2.  **Check Rule 2 (Tenant):** Does `(1, A0001)` match `(1, A0001)`? **YES**.
3.  **Result:** The match is found. Extract $w = 0.8$.
4.  **Assignment:** Set $s_7 = 0.8$.

**Phase 4: Agent Reaction**
* **State:** The Agent adds $0.8$ to the State Vector $S_t$.
* **Inference:** "This is high-value Finance traffic."
* **Behavior:** The Agent will strictly enforce authentication timeouts ($s_2$) and block any anomalous file uploads, even if the user behavior ($s_4$) looks mostly normal.




## Feature 4B: Service Reach Score ($s_{18}$)

### 1. Definition
The **Service Reach Score ($s_{18}$)** is a deterministic state feature that quantifies the **Minimum Network Connectivity** required for the session to function. It represents the "Functional Size" of the service on a normalized scale from 0 to 1.

In the **Causal RL framework**, this feature solves the visibility problem for **Objective 1A (Attack Surface)**. It provides the Agent with a **Baseline Target**:
* **$s_{18} \to 1.0$ (High Connectivity):** The session legitimately requires access to many Network Functions and external gateways (e.g., Internet Browsing).
* **$s_{18} \to 0.0$ (Low Connectivity):** The session legitimately requires access to very few, specific resources (e.g., a Local Sensor).

By observing this value, the Agent learns to scale the security permissions (Microsegment Action) to match the exact needs of the service, ensuring that "Critical" does not mean "Open to Everything."



### 2. Mathematical Formula
We model $s_{18}$ using a **Direct Lookup Function** ($\Phi$) that assigns a pre-defined connectivity score to the session's service type.

$$s_{18} = \Phi(S_{sst}, D_{nn})$$

Where the output is a scalar $s_{18} \in [0, 1]$.

### 3. Parameter Definitions & Calculation Logic

#### A. The Input Variables
These are standard attributes retrieved from the **SMF (Session Management Function)**:
* **$S_{sst}$ (Slice/Service Type):** The category of the slice (e.g., 1=Broadband, 2=Critical).
* **$D_{nn}$ (Data Network Name):** The name of the target network (e.g., "internet", "factory-local").

#### B. The Complexity Mapping Logic ($\Phi$)
The function $\Phi$ assigns a score based on the **Number of Network Functions** the traffic must legitimately pass through.

* **High Complexity ($s_{18} = 0.9$):**
    * **Scenario:** Mobile Broadband (Youtube, Web).
    * **Reason:** Traffic must pass through the Authentication Server, Policy Server, Billing System, Content Filters, and Internet Gateway. It needs a wide path.
* **Medium Complexity ($s_{18} = 0.5$):**
    * **Scenario:** Voice over LTE (IMS).
    * **Reason:** Traffic stays within the operator's Voice Core. It needs a medium path (isolated from the internet, but complex internally).
* **Low Complexity ($s_{18} = 0.1$):**
    * **Scenario:** Industrial Robot (URLLC).
    * **Reason:** Traffic goes directly from the Antenna to a Local Controller. It needs a very narrow, short path.

### 4. Implementation Logic (3GPP Architecture)
* **Architecture:** The mapping rules are defined in the **PCF (Policy Control Function)**.
* **Storage:** The rules are stored in the **UDR (Unified Data Repository)**.
* **Retrieval:** When a session starts, the RL Agent reads this value from the session context via the **Network Exposure Function (NEF)** or internal analytics service.

### 5. Detailed Implementation Action Plan

**Phase 1: Rule Configuration ($t < 0$)**
1.  **Admin Action:** The Network Administrator creates a `complexity_map.json` file.
    ```json
    [
      {"service": "Internet", "sst": 1, "dnn": "internet", "score": 0.9},
      {"service": "Voice",    "sst": 1, "dnn": "ims",      "score": 0.5},
      {"service": "Robot",    "sst": 2, "dnn": "robotics", "score": 0.1}
    ]
    ```
2.  **System Action:** These rules are loaded into the Policy Server (PCF).

**Phase 2: Session Start ($t = 0$)**
1.  **Event:** A Robot requests a connection to the "robotics" network.
2.  **Lookup:** The System matches the request to the "Robot" rule.
3.  **State Update:** The Agent receives **$s_{18} = 0.1$**.

**Phase 3: The Causal Learning (The Result)**
* **Input:** The Agent sees $s_{18} = 0.1$ (Requires Minimal Access).
* **Decision:** The Agent compares its options:
    * *Option A (Wide Open):* Grants access to 50 functions.
    * *Option B (Strictly Locked):* Grants access to 3 functions.
* **Outcome:**
    * If it chooses **Option A**, the Reward is **Negative** (because 50 is much larger than the needed 0.1).
    * If it chooses **Option B**, the Reward is **Positive** (because the access provided matches the access needed).
* **Conclusion:** The Agent learns that "Low Complexity" services must receive "Strict" security policies.

# Class 5: Contextual Features (The Situation)

## Feature 5A: Temporal Deviation Score ($s_9$)

### 1. Definition
**Temporal Deviation Score** is a probabilistic state metric that quantifies **Behavioral Deviation**. It relies on the **NWDAF (Network Data Analytics Function)** machine learning model to calculate the probability that the current session initiation time is anomalous for the specific user.

This feature delegates the historical context to the Core Network, ensuring the Zero Trust Agent remains stateless while leveraging the "Circadian Rhythm" learned by the network.
* **$s_9 \to 1.0$ (Anomaly):** The NWDAF reports a high probability of abnormal behavior (e.g., a "9-to-5" employee connecting at 03:00 AM).
* **$s_9 \to 0.0$ (Normal):** The behavior aligns with the learned historical baseline.



### 2. Mathematical Formula
We model $s_9$ by directly mapping the **Anomaly Probability** returned by the NWDAF API.

$$s_9 = \text{response.probabilityAnomaly}$$

Where:
* **$\text{probabilityAnomaly}$:** A normalized float $\in [0, 1]$ returned in the `abnormalBehavInfos` object, representing the likelihood that the observed event is unexpected based on the user's historical `TrafficPatternInfo`.

### 3. Parameter Definitions & Standards

* **Standard:** **3GPP TS 23.288** (NWDAF Architecture) & **TS 29.520** (Nnwdaf API).
* **Analytics ID:** `UE_ABNORMAL_BEHAVIOUR` (ID: 7).
    * *Purpose:* Detects deviations in UE traffic patterns, mobility, or communication behavior.
* **Target Entity:** **SUPI** (Subscriber Permanent Identifier).
* **Time Granularity:** The request targets specific time slots (e.g., hourly buckets) to detect wake-up anomalies.

### 4. Implementation Logic (Stateless Delegation)

To maintain the efficiency of the RL Agent, we do not store user history locally. We delegate "Context Awareness" to the NWDAF via a stateless API call.

* **Architecture:** The RL Agent acts as an **AF (Application Function)** consuming analytics.
* **Service:** `Nnwdaf_AnalyticsInfo_Request`.
* **Flow:**
    1.  **Trigger:** User requests access ($t=0$).
    2.  **Query:** Agent asks NWDAF "Is this specific time slot normal for this user?"
    3.  **Response:** NWDAF checks its trained model and returns a probability score.
    4.  **Decision:** Agent assigns the score to $s_9$.

### 5. Detailed Implementation Action Plan

**Phase 1: Model Training (Background)**
* **Actor:** NWDAF.
* **Input:** Subscribes to `Namf_EventExposure` for `UE_REACHABILITY` and `PDU_SESSION_STATUS`.
* **Process:** The NWDAF builds a histogram of activity per time-slot for the SUPI. It learns the "Circadian Rhythm" (e.g., User active 09:00-18:00).

**Phase 2: Stateless Inference (The API Call)**
* **Scenario:** The Agent intercepts a session request at **03:00 AM**.
* **Action:** The Agent sends a GET request to the NWDAF to validate the current time window.

* **Corrected Request Payload:**
    ```json
    GET /nnwdaf-analyticsinfo/v2/analytics
    {
      "analyticsId": "UE_ABNORMAL_BEHAVIOUR",
      "targetId": { 
          "supi": "imsi-208100000001234" 
      },
      "startTs": "2024-06-01T03:00:00Z",  
      "endTs": "2024-06-01T04:00:00Z"
    }
    ```

**Phase 3: The Response (Context Retrieval)**
* **Logic:** The NWDAF compares the time slot (03:00-04:00) against the user's history. It finds zero historical activity.
* **Corrected Response Payload:**
    ```json
    {
      "analyticsInfos": [{
        "abnormalBehavInfos": [{
           "ratio": 0,                 // Historical frequency is 0
           "confidence": 90,           // 90% confidence in the model
           "probabilityAnomaly": 0.95  // <--- THE KEY METRIC
        }]
      }]
    }
    ```

**Phase 4: State Injection**
* **Extraction:** The Agent parses `probabilityAnomaly: 0.95`.
* **Assignment:** $s_9 = 0.95$.
* **Impact:** The Agent registers a near-certain behavioral anomaly. Even if credentials are valid, the **Time Context** is rejected. The Agent forces a **Block** or **Step-Up Authentication** decision.

# Class 6: Threat Intelligence & Exposure Features (The External Knowledge)

## Feature 6A: Session IoC Score ($s_{12}$)

### 1. Definition
The **Session IoC Score** is a **normalized**, **weighted saturation metric** that quantifies the probability of compromise based on forensic evidence found in the session traffic (IoC stands for Indicator of Compromise).

The normalization is intrinsic to the formula: as evidence accumulates, the score naturally asymptotically approaches 1.0 (Certainty), but never exceeds it.

1.  **Network Layer Matches ($C_1$):** Connections to malicious IP addresses (Source: UPF Layer 3 Filter).
2.  **Application Layer Matches ($C_2$):** Connections to malicious Domains/URLs (Source: UPF Layer 7 Filter).

* **$s_{12} \to 1.0$ (Critical):** Confirmed compromise.
* **$s_{12} = 0.0$ (Clean):** No matches.

### 2. Mathematical Formula
We use the **Inverse Exponential Decay** (Saturation) function. This naturally maps any positive integer count into the range $[0, 1]$.

$$s_{12}(t) = 1 - e^{ -\left( \alpha_1 \cdot C_1(t) + \alpha_2 \cdot C_2(t) \right) }$$

* The weights **$\alpha$** now control the saturation speed directly.
    * If $\alpha_1 = 1.0$, a single packet results in $1 - e^{-1} \approx 0.63$ (High Risk immediately).
    * If $\alpha_2 = 0.01$, a single packet results in $1 - e^{-0.01} \approx 0.01$ (Negligible Risk).

### 3. Parameter Definitions

| Parameter | Name | Definition | 3GPP Source |
| :--- | :--- | :--- | :--- |
| **$C_1(t)$** | **Network IoC Count** | The integer count of packets matching **List A (IPs)** in the current observation window. | **UPF** (Usage Reporting Rule) |
| **$C_2(t)$** | **App IoC Count** | The integer count of HTTP/DNS requests matching **List B (Domains)** in the current observation window. | **UPF** (Deep Packet Inspection) |
| **$\alpha_1$** | **Network Impact Rate** | The "Lethality" of an IP match. <br>• Recommended: **1.0** (Critical). <br>• Logic: 1 packet to a C2 IP is enough to assume breach ($1-e^{-1} = 63\%$). 3 packets $\approx 95\%$. | **PCF** (Policy Rule) |
| **$\alpha_2$** | **App Impact Rate** | The "Lethality" of a Domain match. <br>• Recommended: **0.1** (Suspicious). <br>• Logic: It takes ~10 bad DNS requests to reach the same risk level as 1 C2 packet. | **PCF** (Policy Rule) |

### 4. Detailed Implementation Methodology
We utilize the **Network Exposure Function (NEF)** to inject the intelligence and the **UPF** to count the matches.



#### Phase 1: Intelligence Provisioning (NEF)
* **Step A:** The **Application Function (AF)** (Security Center) maintains two distinct threat lists based on the IoC type:
    * **List A (IP Addresses):** Contains confirmed malicious IPs (e.g., C2, Botnets).
    * **List B (Domain Names):** Contains malicious Domains (e.g., Phishing, Adware).
* **Step B:** The AF pushes these lists to the **NEF** via the `Nnef_PFDManagement_Create` service, mapping them to two distinct Application IDs:
    * List A $\to$ `APP_ID_MALICIOUS_IP`
    * List B $\to$ `APP_ID_MALICIOUS_DOMAIN`
* **Step C:** The NEF stores these Packet Flow Descriptions (PFDs) in the **UDR (Unified Data Repository)**.

#### Phase 2: Detection Enforcement (UPF)
* **Step A:** During PDU Session Establishment, the **SMF** retrieves the PFDs for both Application IDs from the UDR.
* **Step B:** The SMF installs **Packet Detection Rules (PDRs)** in the UPF, linking them to separate counters:
    * **PDR_1:** Matches traffic against `APP_ID_MALICIOUS_IP` $\to$ Increments Counter **$C_1$**.
    * **PDR_2:** Matches traffic against `APP_ID_MALICIOUS_DOMAIN` $\to$ Increments Counter **$C_2$**.

#### Phase 3: State Calculation (The Agent)
* **Step A:** The Agent retrieves the current values of $C_1$ and $C_2$ from the SMF Usage Report.
    * *Example:* The user visited a phishing site ($C_2=1$) and the site resolved to a C2 IP ($C_1=1$).
* **Step B:** The Agent applies the impact rates ($\alpha_1=1.0, \alpha_2=0.1$).
    * Exponent $X = (1 \times 1.0) + (1 \times 0.1) = 1.1$.
* **Step C:** Calculate $s_{12}$ using the formula.
    * $s_{12} = 1 - e^{-1.1} \approx 0.668$.
* **Result:** The State Vector receives **0.668**.


# Class 7: Policy Stability Features (The Controller)

These features provide the RL Agent with **Temporal Context** (Memory). Without them, the Agent is "forgetful" and might oscillate rapidly between decisions (Flapping), which destroys User Experience and causes control instability.

## Feature 7A: Stability Timer ($s_{15}$)

### 1. Definition
**Stability Timer** is a deterministic, continuous measure of **Temporal Hysteresis**. It tracks the magnitude of time elapsed since the *last* change in policy state for the current session.

* **Level:** **Session Level**.
* **Function:** It acts as the **Patience Signal**.
    * Unlike a fixed timer, this value grows continuously.
    * The RL Agent uses this to learn: *"If I just blocked this user 10ms ago ($s_{15} \approx 1$), I shouldn't try to unblock them yet. If I blocked them 1 hour ago ($s_{15} \approx 6$), maybe I should check again."*



### 2. Mathematical Formula
We model $s_{15}$ using a **Logarithmic Time Scale**. This compresses the huge dynamic range of time (from milliseconds to hours) into a small, neural-network-friendly scalar, without requiring any arbitrary thresholds ($T_{max}$).

$$s_{15} = \log_{10} \left( 1 + (t_{now} - t_{last\_change}) \right)$$

* **Unit:** The time difference $(t_{now} - t_{last\_change})$ is measured in **milliseconds**.
* **$\log_{10}$:** Ensures that 1 second and 1 hour are comparable inputs, preventing gradient explosion.

### 3. Parameter Definitions
* **$t_{now}$:** The current system timestamp (Unix Epoch in ms).
* **$t_{last\_change}$:** The timestamp when the Agent last output a *different* action than the previous timestep.
* **$1$:** A smoothing constant to ensure that if $\Delta t = 0$, the log result is $\log(1)=0$ (not undefined).

### 4. Implementation Logic
* **Storage:** The Session State Map stores `last_action_ts` (Long Integer).
* **Update Trigger:**
    * If $Action_{t} \neq Action_{t-1}$: Update `last_action_ts = t_{now}`.
    * Else: Keep `last_action_ts`.

### 5. Detailed Calculation Workflow
Since there is no $T_{max}$, the value scales naturally with time.

**Scenario A: Immediate Flapping (Bad Behavior)**
* **Time Delta:** 10 ms (Agent tries to change mind instantly).
* **Calculation:** $\log_{10}(1 + 10) = \log_{10}(11) \approx \mathbf{1.04}$.
* **Meaning:** Value is low. The Neural Network learns that changing actions when $s_{15} \approx 1.0$ leads to penalties (Negative Reward for instability).

**Scenario B: Short Wait (Re-Auth)**
* **Time Delta:** 2,000 ms (2 seconds).
* **Calculation:** $\log_{10}(1 + 2000) \approx \log_{10}(2000) \approx \mathbf{3.30}$.

**Scenario C: Long Term (Session Expiry)**
* **Time Delta:** 3,600,000 ms (1 Hour).
* **Calculation:** $\log_{10}(3,600,001) \approx \mathbf{6.56}$.
* **Meaning:** Value is distinctively high. The Network learns that after $s_{15} > 6.0$, it is safe to reset the policy.

---

## Feature 7B: Previous Action State ($s_{16}$)

### 1. Definition
**Previous Action State** is a categorical, stateful feature that explicitly informs the Agent of its own **History**. It satisfies the Markov Property ($S_{t+1}$ depends on $S_t$ and $A_t$) by feeding the previous output $A_{t-1}$ back as an input.

* **Level:** **Session Level**.
* **Function:** It acts as the **Inertia Bias (State Persistence)**.
    * It provides the Agent with "Momentum." If the previous state was `Mitigate`, the Agent is statistically biased to output `Mitigate` again unless the other input features (Risk Scores) change significantly.
    * This prevents the Agent from needing to "re-discover" the correct action every millisecond.



### 2. Mathematical Formula (One-Hot Encoding)
Since the actions are categorical (discrete types), we cannot use a single scalar. We use **One-Hot Encoding** to represent the previous action as a binary vector. This vector is appended to the State Space.

$$s_{16} = \vec{v} \in \{0, 1\}^{|A|}$$

Where $|A|$ is the size of the Action Space.

### 3. Parameter Definitions (The Action Space)
Assuming a standard Zero Trust Action Space with 4 possible interventions:

| Index | Action Name | Definition | One-Hot Vector ($s_{16}$) |
| :--- | :--- | :--- | :--- |
| **0** | **Allow** | Pass traffic normally. | `[1, 0, 0, 0]` |
| **1** | **Block** | Drop traffic and terminate session. | `[0, 1, 0, 0]` |
| **2** | **Authenticate** | Trigger Re-Auth (MFA/SIM Challenge). | `[0, 0, 1, 0]` |
| **3** | **Mitigate** | Apply QoS throttling or Deep Inspection. | `[0, 0, 0, 1]` |

### 4. Implementation Logic
* **Architecture:** The RL Environment Wrapper buffers the output of step $t-1$.
* **Initialization:** At the start of a session ($t=0$), the default is **Allow** (`[1, 0, 0, 0]`) or **Block** (`[0, 1, 0, 0]`) depending on the default stance.

### 5. Detailed Calculation Workflow
**Scenario:**
1.  **Time $t=1$:** Agent calculates risk and decides to **Authenticate (Index 2)**.
    * Output Action $A_1 = 2$.
    * System executes MFA trigger.
2.  **Time $t=2$:** The user sends the MFA token. The Agent needs to re-evaluate.
    * **Input Construction:** The Agent builds the State Vector $S_2$.
    * **Feature $s_{16}$ Generation:** The system looks at $A_1$ (which was 2).
    * It generates the vector `[0, 0, 1, 0]`.
    * This vector is flattened and appended to the inputs.
3.  **Inference:** The Neural Network sees `[..., 0, 0, 1, 0]`.
    * *Interpretation:* "I previously asked for MFA. Did the user pass?"
    * It combines this with the new Authentication Status ($s_{12}$). If $s_{12} = \text{Success}$, the Agent learns that the sequence `Auth -> Allow` is correct.