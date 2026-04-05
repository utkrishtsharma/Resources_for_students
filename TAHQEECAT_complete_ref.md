# TAHQEECAT Interview Reference — Quantum Networks, QKD, PQC, SDN
> Zero-to-Expert. Theory + Math + Annotated Code + Interview Q&A.
> Structured as a knowledge tree: main branches = your resume claims; sub-branches = interview depth.

---

## 📌 Meta-Map (How This Document is Structured)

```
ROOT: Quantum Information Foundations
├── Qubits, Gates, Entanglement, No-Cloning
├── Density Matrices, Fidelity, Trace Distance
│
├─── BRANCH A: Quantum Key Distribution (QKD)
│     ├── BB84 (full math + Qiskit code)
│     ├── MDI-QKD (untrusted relay, BSM)
│     ├── Twin-Field QKD (PLOB bound, SNS variant)
│     ├── CV-QKD (Gaussian states, homodyne, reconciliation)
│     └── Security proofs overview (composable, IND-CCA)
│
├─── BRANCH B: Quantum Networks & Simulation
│     ├── QuNetSim framework (architecture + code)
│     ├── NetSquid / SeQUeNCe overview
│     ├── Quantum teleportation (circuit + proof)
│     ├── Entanglement swapping → Quantum Repeaters
│     ├── Entanglement purification (DEJMPS)
│     ├── QRNG, State Tomography
│     └── Quantum SDN (OpenFlow analogy, API design)
│
├─── BRANCH C: Post-Quantum Cryptography (PQC)
│     ├── Threat model (Shor, Grover)
│     ├── NIST PQC standards (Kyber, Dilithium, Falcon, SPHINCS+)
│     ├── Lattice problems: LWE, RLWE, SIS
│     ├── Your PQC browser architecture (Army submission)
│     └── FastAPI + SQLite backend design
│
├─── BRANCH D: Quantum Error Correction (QEC)
│     ├── 3-qubit repetition code
│     ├── Shor code (9-qubit)
│     ├── Stabilizer formalism
│     ├── Surface codes (toric → planar)
│     ├── Topological QEC
│     └── Classical post-processing: Cascade, LDPC, Privacy Amplification
│
├─── BRANCH E: IBM Quantum / Qiskit Implementations
│     ├── Bell states + CHSH (S > 2√2)
│     ├── Deutsch-Jozsa
│     ├── DEJMPS purification (4-qubit circuit)
│     └── Cloud backend constraints
│
├─── BRANCH F: Advanced SDN Projects
│     ├── Hybrid DV-CV SDN simulator
│     ├── ML-optimized entanglement distribution
│     ├── Quantum dot source characterization (g²(0), HOM)
│     └── SDN testbed + repeater on IBM Quantum CC
│
└─── BRANCH G: Interview Master Q&A
      ├── Core conceptual
      ├── Math/derivation
      ├── Code walkthrough
      └── System design
```

---

# ROOT: Quantum Information Foundations

## R.1 The Qubit

A classical bit ∈ {0, 1}. A qubit lives in a 2D complex Hilbert space ℂ²:

```
|ψ⟩ = α|0⟩ + β|1⟩,    α, β ∈ ℂ,    |α|² + |β|² = 1
```

**Computational basis:** `|0⟩ = [1, 0]ᵀ`, `|1⟩ = [0, 1]ᵀ`

**Bloch Sphere parameterization:**
```
|ψ⟩ = cos(θ/2)|0⟩ + e^(iφ)sin(θ/2)|1⟩
```
- θ ∈ [0, π]: polar angle (latitude)
- φ ∈ [0, 2π): azimuthal angle (longitude)
- North pole = |0⟩, South pole = |1⟩, Equator = superpositions

> **Interview hook:** "The Bloch sphere is a unit sphere in ℝ³, but qubits live in ℂ². Why? Because global phase e^(iγ)|ψ⟩ is physically unobservable — only relative phase matters. This collapses the 4 real parameters of ℂ² down to 3 (minus global phase) → Bloch sphere."

---

## R.2 Quantum Gates (Unitary Operators)

All quantum evolution (except measurement) is **unitary**: U†U = I.

### Single-qubit gates (as matrices):

```
X = [0 1]    Y = [0 -i]    Z = [1  0]    H = 1/√2 [1  1]
    [1 0]        [i  0]        [0 -1]              [1 -1]

S = [1 0]    T = [1    0   ]
    [0 i]        [0  e^(iπ/4)]
```

**Key identities:**
```
HXH = Z         (Hadamard conjugates X↔Z)
HZH = X
X² = Y² = Z² = I
HH = I
```

### Two-qubit gates:

```
CNOT = |00⟩⟨00| + |01⟩⟨01| + |11⟩⟨10| + |10⟩⟨11|
     = [1 0 0 0]
       [0 1 0 0]
       [0 0 0 1]
       [0 0 1 0]
```

**CNOT acts on |control, target⟩**: flips target iff control = |1⟩

---

## R.3 Multi-Qubit Systems

n-qubit state lives in ℂ^(2ⁿ). Two-qubit state:
```
|ψ₁⟩ ⊗ |ψ₂⟩  (tensor product — separable state)
```
A state that CANNOT be written as a tensor product is **entangled**.

### Bell States (maximally entangled):
```
|Φ⁺⟩ = (|00⟩ + |11⟩)/√2    ← created by H⊗I → CNOT
|Φ⁻⟩ = (|00⟩ - |11⟩)/√2
|Ψ⁺⟩ = (|01⟩ + |10⟩)/√2
|Ψ⁻⟩ = (|01⟩ - |10⟩)/√2
```

**Bell circuit:**
```
Alice: |0⟩ ──[H]──●──
                   │
Bob:   |0⟩ ────────⊕──  → |Φ⁺⟩
```

---

## R.4 Density Matrices (Mixed States)

Pure state: ρ = |ψ⟩⟨ψ|, Tr(ρ²) = 1
Mixed state: ρ = Σᵢ pᵢ |ψᵢ⟩⟨ψᵢ|, Tr(ρ²) < 1

**Why you need this:** Noise, partial measurements, subsystems of entangled states are ALL described by mixed states (density matrices), not state vectors.

```
ρ maximally mixed = I/2 = [1/2  0 ]
                           [ 0  1/2]  ← center of Bloch sphere
```

### Fidelity between states:
```
F(ρ, σ) = (Tr√(√ρ σ √ρ))²
```
For pure states: F(|ψ⟩, |φ⟩) = |⟨ψ|φ⟩|²

### Trace Distance:
```
D(ρ, σ) = ½ Tr|ρ - σ|  where |A| = √(A†A)
```

---

## R.5 No-Cloning Theorem

**Statement:** There is no unitary U such that U|ψ⟩|0⟩ = |ψ⟩|ψ⟩ for all |ψ⟩.

**Proof sketch:**
```
Suppose U clones. Then:
U|ψ⟩|0⟩ = |ψ⟩|ψ⟩
U|φ⟩|0⟩ = |φ⟩|φ⟩

Take inner product:
⟨φ|ψ⟩ = ⟨φ|ψ⟩²

This holds only if ⟨φ|ψ⟩ = 0 or 1 (orthogonal or identical).
For general states, this is a contradiction. ∎
```

**Why it matters:** Eve cannot intercept-and-forward qubits without disturbing them → foundation of QKD security.

---

# BRANCH A: Quantum Key Distribution (QKD)

## A.1 BB84 Protocol

**Reference:** Bennett & Brassard, 1984. First QKD protocol.

### States:
```
Basis Z: |0⟩ encodes bit 0,  |1⟩ encodes bit 1
Basis X: |+⟩ = H|0⟩ encodes bit 0,  |-⟩ = H|1⟩ encodes bit 1

|+⟩ = (|0⟩ + |1⟩)/√2
|-⟩ = (|0⟩ - |1⟩)/√2
```

### Protocol:
1. **Alice:** for each bit, pick random bit bₐ ∈ {0,1}, random basis θₐ ∈ {Z, X}. Encode and send.
2. **Bob:** for each qubit, pick random basis θ_b. Measure.
3. **Sifting:** Announce bases over public channel. Keep bits where θₐ = θ_b.
4. **Error estimation:** Sacrifice ~10% of sifted bits. Compute QBER = (error count)/(sample size).
5. If QBER > threshold (~11%) → abort. Eve detected.
6. **Error correction:** Cascade or LDPC over classical channel.
7. **Privacy amplification:** Universal hash → final key.

### Security Math:
If Eve measures in the wrong basis (prob = 1/2), she introduces error prob = 1/4 per bit.
```
QBER_Eve ≈ 25% for intercept-resend attack
QBER_threshold ≈ 11% (secure key still extractable via privacy amplification)
```

Secure key rate (asymptotic):
```
r = 1 - H(QBER) - H(QBER)    [H = binary Shannon entropy]
  = 1 - 2H(QBER)              (simplified)
```

### Qiskit Implementation (annotated):

```python
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import numpy as np
import random

def bb84_simulation(n_bits=100, eve_present=False):
    """
    Full BB84 simulation with optional eavesdropper.
    Returns: (alice_key, bob_key, qber)
    """
    simulator = AerSimulator()
    
    # === STEP 1: Alice prepares qubits ===
    alice_bits = [random.randint(0, 1) for _ in range(n_bits)]
    alice_bases = [random.choice(['Z', 'X']) for _ in range(n_bits)]
    
    # === STEP 2: Simulate channel (with optional Eve) ===
    bob_bases = [random.choice(['Z', 'X']) for _ in range(n_bits)]
    bob_results = []
    
    for i in range(n_bits):
        # Build circuit: 1 qubit, 1 classical bit
        qc = QuantumCircuit(1, 1)
        
        # --- Alice encodes ---
        if alice_bits[i] == 1:
            qc.x(0)                  # flip to |1⟩
        if alice_bases[i] == 'X':
            qc.h(0)                  # rotate to X basis: |0⟩→|+⟩, |1⟩→|-⟩
        
        # --- Eve intercepts (intercept-resend attack) ---
        if eve_present:
            eve_basis = random.choice(['Z', 'X'])
            if eve_basis == 'X':
                qc.h(0)              # Eve measures in X basis
            qc.measure(0, 0)         # Eve's measurement collapses the state
            
            # Eve re-prepares based on her result (she doesn't know what she got yet)
            # We simulate: reset and re-encode in Eve's basis
            # (Simplified: just adds decoherence via measurement)
            qc.reset(0)
            # Re-prepare (we skip exact re-encoding for brevity — error is introduced)
            if eve_basis == 'X':
                qc.h(0)
        
        # --- Bob measures ---
        if bob_bases[i] == 'X':
            qc.h(0)                  # rotate back before measuring
        qc.measure(0, 0)
        
        # Run circuit (1 shot — this is a single photon event)
        compiled = transpile(qc, simulator)
        result = simulator.run(compiled, shots=1, memory=True).result()
        bob_results.append(int(result.get_memory()[0]))
    
    # === STEP 3: Sifting — keep matching bases ===
    sifted_alice = []
    sifted_bob = []
    for i in range(n_bits):
        if alice_bases[i] == bob_bases[i]:
            sifted_alice.append(alice_bits[i])
            sifted_bob.append(bob_results[i])
    
    # === STEP 4: QBER estimation on sample ===
    sample_size = max(1, len(sifted_alice) // 5)  # 20% sample
    errors = sum(
        1 for a, b in zip(sifted_alice[:sample_size], sifted_bob[:sample_size])
        if a != b
    )
    qber = errors / sample_size
    
    # === STEP 5: Final key (remaining sifted bits) ===
    final_alice = sifted_alice[sample_size:]
    final_bob   = sifted_bob[sample_size:]
    
    return final_alice, final_bob, qber

# Run simulation
key_a, key_b, qber = bb84_simulation(n_bits=200, eve_present=False)
print(f"Key length: {len(key_a)}")
print(f"QBER: {qber:.2%}")
print(f"Keys match: {key_a == key_b}")

key_a_eve, key_b_eve, qber_eve = bb84_simulation(n_bits=200, eve_present=True)
print(f"\n[Eve present] QBER: {qber_eve:.2%}")  # Should be ~25%
```

---

## A.2 MDI-QKD (Measurement-Device-Independent)

### Problem Solved:
In practice, detectors can be attacked (e.g., blinding attacks, time-shift attacks). MDI-QKD removes all detector vulnerabilities.

### Architecture:
```
Alice ──────────────────────────►
                                Charlie (untrusted relay)
Bob ────────────────────────────►
        BSM performed by Charlie
```

### Protocol:
1. Alice and Bob each prepare weak coherent states (WCS) independently in a random basis (Z or X).
2. Both send to Charlie, who performs a **Bell State Measurement (BSM)** — projects onto Bell basis.
3. Charlie announces his BSM result publicly.
4. Alice and Bob post-select on runs where Charlie got a valid Bell state outcome.
5. From this, they derive correlated bits → key after error correction + privacy amplification.

### Decoy States:
Real sources emit Poisson-distributed photon numbers:
```
P(n) = μⁿ e^(-μ) / n!
```
Attacker (PNS attack) blocks single-photon pulses, forwards multi-photon ones.
**Fix:** Alice/Bob randomly vary intensities μ, ν₁, ν₂ (signal + 2 decoys). 
From measured gains/error rates at different intensities → bound single-photon contributions.

### Key Rate:
```
R_MDI ≥ Q₁₁[1 - H(e₁₁)] - Q_μμ f_EC H(E_μμ)
```
Where:
- Q₁₁ = gain of single-photon events
- e₁₁ = phase error rate for single-photon events
- f_EC = error correction efficiency

---

## A.3 Twin-Field QKD (TF-QKD)

### The PLOB Bound (why it matters):
```
K_PLOB = -log₂(1 - η)  ≈  η / ln(2)   for small η
```
η = channel transmittance. This is the maximum key rate for point-to-point QKD without quantum repeaters.

Standard BB84 already saturates near PLOB. Long distances → η → 0 → key rate → 0 exponentially.

### TF-QKD insight:
Alice and Bob each send coherent states to Charlie. Charlie performs **single-photon interference** (not BSM). 

Key rate scales as **√η** instead of η. This beats PLOB!

```
K_TF ∝ √η    (vs K_BB84 ∝ η)
```

### SNS-TF-QKD (Send-or-Not-Send variant):
- Each round: Alice/Bob independently choose to "send" (state |α⟩) or "not send" (vacuum |0⟩).
- Charlie clicks if interference occurs (single photon detected).
- Z-basis bits: send vs. not-send (used for raw key)
- X-basis: phase-randomized coherent states (used for phase error estimation)

---

## A.4 CV-QKD

### Why CV?
Single-photon detectors are expensive, slow, need cooling. CV-QKD uses standard **homodyne/heterodyne** telecom detectors.

### Gaussian-Modulated Coherent State (GMCS) Protocol:
1. Alice randomly samples Gaussian amplitudes: x_A, p_A ~ N(0, V_A)
2. Encodes in coherent state: |α⟩ where α = (x_A + ip_A)/2
3. Transmits through lossy channel (transmittance T, excess noise ε)
4. Bob measures quadrature X or P (homodyne: one quadrature) or both (heterodyne)

### Channel model:
```
x_B = √T · x_A + z    where z ~ N(0, T·ε + 1)
```

### Security:
Key rate (Devetak-Winter):
```
K = βI(A:B) - χ(E:B)
```
- β = reconciliation efficiency (typically 0.95)
- I(A:B) = classical mutual information
- χ(E:B) = Holevo bound (Eve's info)

**Reconciliation challenge:** Since data is continuous Gaussian, need reverse reconciliation + LDPC codes with rate close to Shannon limit.

---

## A.5 QKD Post-Processing Pipeline

```
Raw Key (sifted, length n)
        ↓
Error Correction  ←── Cascade (interactive) or LDPC (one-shot)
        ↓
Confirmed Key (length n - leak_EC)
        ↓
Privacy Amplification  ←── Universal hashing (Toeplitz matrix)
        ↓
Final Secret Key  (length ≈ n - leak_EC - H_max(E))
```

### Cascade Error Correction:
1. Divide key into blocks. XOR parity of each block over public channel.
2. If parity mismatch → binary search for error within block.
3. BISECT: recurse to find exact error position.
4. Repeat for multiple passes with shuffled permutations.
- Leaks ~1.16 H(QBER) bits per bit (close to Shannon limit H(QBER))

### LDPC (Low-Density Parity-Check):
- Alice sends syndrome s = Hx (parity check matrix H, Alice's bits x)
- Bob uses syndrome + his bits to decode (belief propagation)
- Leaks exactly k = n - rank(H) bits (can approach Shannon limit)

### Privacy Amplification (Toeplitz):
- Choose random Toeplitz matrix M of size m×n (m < n)
- Final key = M · confirmed_key
- A Toeplitz matrix is uniquely defined by 2n-1 random bits → efficient to communicate seed

---

# BRANCH B: Quantum Networks & Simulation

## B.1 QuNetSim Framework

**Repository:** https://github.com/tqsd/QuNetSim
**Paper:** Diadamo et al. 2020, arXiv:2003.06397

### Architecture:
```
QuNetSim Layer Stack:
┌──────────────────────────┐
│  Application Layer       │ ← your protocol (teleport, QKD)
├──────────────────────────┤
│  Transport Layer         │ ← manages acknowledgements, resend
├──────────────────────────┤
│  Network Layer           │ ← routing between hosts
├──────────────────────────┤
│  Data Link Layer         │ ← frame management, EPR pair storage
├──────────────────────────┤
│  Physical Layer Backend  │ ← Qiskit / CQC / stabilizer sim
└──────────────────────────┘
```

### QuNetSim: Qubit Transmission (annotated):

```python
from qunetsim.components import Host, Network
from qunetsim.objects import Qubit
import time

def qubit_transmission_demo():
    """
    Demonstrates: Alice creates qubit, sends to Bob, Bob measures.
    Validates: Fidelity of transmission over simulated channel.
    """
    
    # Create the network singleton
    network = Network.get_instance()
    network.start()
    
    # Create host nodes
    alice = Host('Alice')
    bob   = Host('Bob')
    
    # Build topology: Alice — Bob (direct link)
    alice.add_connection('Bob')
    bob.add_connection('Alice')
    
    # Register hosts with network
    network.add_host(alice)
    network.add_host(bob)
    
    alice.start()
    bob.start()
    
    results = []
    n_trials = 20
    
    for trial in range(n_trials):
        # Alice creates a fresh |0⟩ qubit
        q = Qubit(alice)
        q.X()  # flip to |1⟩ — encode bit "1"
        
        # Alice sends the qubit to Bob
        # Returns a "qubit ID" so Bob can receive it
        q_id = alice.send_qubit('Bob', q, await_ack=True)
        
        # Bob receives (blocks until available)
        received_q = bob.get_qubit('Alice', q_id)
        
        if received_q is not None:
            result = received_q.measure()   # returns 0 or 1
            results.append(result)
    
    # Fidelity: fraction of times we got expected result (1)
    fidelity = sum(results) / len(results)
    print(f"Transmission fidelity: {fidelity:.2%}")
    print(f"Expected: |1⟩, Got: {results[:10]}...")
    
    # Cleanup
    network.stop(stop_hosts=True)

qubit_transmission_demo()
```

### QuNetSim: Quantum Teleportation (annotated):

```python
from qunetsim.components import Host, Network
from qunetsim.objects import Qubit

def teleportation_demo():
    """
    Alice teleports |ψ⟩ to Bob.
    Protocol:
      1. Alice-Bob share EPR pair (Bell state |Φ⁺⟩)
      2. Alice does Bell measurement on (|ψ⟩, her EPR qubit)
      3. Alice sends 2 classical bits to Bob
      4. Bob applies corrections based on classical bits
    """
    network = Network.get_instance()
    network.start()
    
    alice = Host('Alice')
    bob   = Host('Bob')
    
    alice.add_connection('Bob')
    bob.add_connection('Alice')
    
    network.add_host(alice)
    network.add_host(bob)
    
    alice.start()
    bob.start()
    
    # --- Create the state to teleport ---
    psi = Qubit(alice)
    psi.H()   # prepare |+⟩ = H|0⟩ = (|0⟩+|1⟩)/√2
    psi.S()   # S|+⟩ = (|0⟩+i|1⟩)/√2  = |Y+⟩  (arbitrary state)
    
    print("[Alice] Teleporting |ψ⟩ = S·H|0⟩")
    
    # --- Alice and Bob create a shared Bell pair ---
    # QuNetSim helper: share_EPR sends one half to Bob
    epr_alice = Qubit(alice)
    epr_alice.H()               # |+⟩ on Alice's side
    
    epr_bob = Qubit(alice)      # will be sent to Bob
    epr_alice.cnot(epr_bob)     # entangle: |Φ⁺⟩ = (|00⟩+|11⟩)/√2
    
    # Send Bob's half
    epr_id = alice.send_qubit('Bob', epr_bob, await_ack=True)
    
    # --- Alice's Bell Measurement on (psi, epr_alice) ---
    psi.cnot(epr_alice)         # CNOT: control=psi, target=epr_alice
    psi.H()                     # Hadamard on psi
    
    m1 = psi.measure()          # measure first qubit  → classical bit c1
    m2 = epr_alice.measure()    # measure second qubit → classical bit c2
    
    print(f"[Alice] Bell measurement: m1={m1}, m2={m2}")
    
    # --- Alice sends classical bits to Bob ---
    alice.send_classical('Bob', f"{m1}{m2}", await_ack=True)
    
    # --- Bob receives classical bits and applies corrections ---
    msg = bob.get_classical('Alice', wait=5)
    if msg:
        c1, c2 = int(msg.content[0]), int(msg.content[1])
        
        # Bob receives his half of the EPR pair
        received = bob.get_qubit('Alice', epr_id)
        
        # Apply corrections based on teleportation protocol:
        if c2 == 1:
            received.X()    # bit-flip correction if m2=1
        if c1 == 1:
            received.Z()    # phase-flip correction if m1=1
        
        # Bob now has the teleported state |ψ⟩
        # To verify: undo the preparation and check we get |0⟩
        received.S_dagger()   # undo S
        received.H()          # undo H
        result = received.measure()
        print(f"[Bob] Measured (should be 0 if teleportation perfect): {result}")
    
    network.stop(stop_hosts=True)

teleportation_demo()
```

---

## B.2 Entanglement Swapping → Quantum Repeaters

### Core Idea:
```
A ←── EPR ───► M₁ ←── EPR ───► M₂ ←── EPR ───► B
                    BSM at M₁ and M₂
                           ↓
A ←─────────────────────── EPR ────────────────► B
```

### Math:
Start with:
```
|Φ⁺⟩_AM₁ ⊗ |Φ⁺⟩_M₂B
= ¼ Σ_{m∈{Φ⁺,Φ⁻,Ψ⁺,Ψ⁻}} |m⟩_M₁M₂ ⊗ (σ_m|Φ⁺⟩_AB)
```

After BSM on M₁M₂ with result |m⟩: Bob applies correction σ_m → shared |Φ⁺⟩_AB.

### Quantum Repeater Architecture:
```
Segment 1        Segment 2        Segment 3
A ──EPR──► R₁ ──EPR──► R₂ ──EPR──► B
           │            │
           └── swap ────┘
           → extend range
```

**Key challenge:** Memory lifetime (decoherence). Need quantum memories to store entanglement while waiting for neighboring segments.

---

## B.3 Entanglement Purification (DEJMPS)

### Problem: Noisy Bell pairs (Werner states):
```
ρ = F|Φ⁺⟩⟨Φ⁺| + (1-F)/3 (|Φ⁻⟩⟨Φ⁻| + |Ψ⁺⟩⟨Ψ⁺| + |Ψ⁻⟩⟨Ψ⁻|)
```
F = fidelity. If F < 0.5, state is not useful.

### DEJMPS Protocol (4-qubit circuit):
Both Alice and Bob hold two noisy pairs: (a₁,b₁) and (a₂,b₂).

```
Alice's side:
  a₁ ──[Rx(π/2)]── CNOT ctrl ── Measure → m_a
  a₂ ──[Rx(π/2)]── CNOT targ

Bob's side:
  b₁ ──[Rx(π/2)]── CNOT ctrl ── Measure → m_b
  b₂ ──[Rx(π/2)]── CNOT targ
```

- If m_a = m_b (outcomes agree): keep pair (a₂, b₂) as purified output
- If m_a ≠ m_b: discard both pairs

### Fidelity update:
```
F' = (F² + ((1-F)/3)²) / (F² + 2F(1-F)/3 + 5((1-F)/3)²)
```
Converges to F=1 with repeated applications (if F > 0.5 initially).

### Qiskit DEJMPS (annotated):

```python
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
from qiskit.circuit.library import RXGate
import numpy as np

def dejmps_circuit(fidelity=0.75):
    """
    4-qubit DEJMPS purification circuit.
    Qubits: [a1, a2, b1, b2]
    a1, b1 = pair 1 (control pair for Alice/Bob)
    a2, b2 = pair 2 (target pair)
    """
    qc = QuantumCircuit(4, 2)  # 4 qubits, 2 classical (measure a1, b1)
    
    # --- Prepare two noisy Bell pairs ---
    # Pair 1: qubits 0 (Alice=a1) and 2 (Bob=b1)
    qc.h(0)
    qc.cx(0, 2)   # |Φ⁺⟩ on (a1, b1)
    
    # Pair 2: qubits 1 (Alice=a2) and 3 (Bob=b2)
    qc.h(1)
    qc.cx(1, 3)   # |Φ⁺⟩ on (a2, b2)
    
    # --- Introduce noise on pair 2 (simulate decoherence) ---
    noise_angle = 2 * np.arccos(np.sqrt(fidelity))   # depolarize via Rz rotation
    qc.rz(noise_angle, 1)    # apply phase error on Alice's qubit of pair 2
    
    qc.barrier()
    
    # --- DEJMPS step 1: Rx(π/2) rotation on both pairs ---
    # This is the key DEJMPS bilateral rotation
    qc.rx(np.pi/2, 0)     # Alice: rotate a1
    qc.rx(np.pi/2, 1)     # Alice: rotate a2
    qc.rx(-np.pi/2, 2)    # Bob:   rotate b1 (opposite sign)
    qc.rx(-np.pi/2, 3)    # Bob:   rotate b2
    
    # --- DEJMPS step 2: CNOT — pair1 controls pair2 ---
    qc.cx(0, 1)    # Alice: a1 (ctrl) → a2 (targ)
    qc.cx(2, 3)    # Bob:   b1 (ctrl) → b2 (targ)
    
    # --- DEJMPS step 3: Measure the target pair ---
    qc.measure(1, 0)    # measure a2 → classical bit 0
    qc.measure(3, 1)    # measure b2 → classical bit 1
    
    # If classical bits agree (00 or 11): (a1, b1) = purified pair
    # If they disagree: discard
    
    return qc

# Run and analyze
sim = AerSimulator()
qc = dejmps_circuit(fidelity=0.75)
compiled = transpile(qc, sim)
result = sim.run(compiled, shots=1000).result()
counts = result.get_counts()

print("DEJMPS measurement outcomes (a2, b2):")
for outcome, count in sorted(counts.items()):
    label = "KEEP (purified)" if outcome in ['00', '11'] else "DISCARD"
    print(f"  {outcome}: {count:4d} shots  → {label}")

agree = counts.get('00', 0) + counts.get('11', 0)
total = sum(counts.values())
success_prob = agree / total
print(f"\nPurification success probability: {success_prob:.2%}")
```

---

## B.4 Quantum State Tomography

### Goal: Reconstruct the density matrix ρ from measurements.

For single qubit, ρ is a 2×2 Hermitian trace-1 matrix:
```
ρ = ½(I + r·σ)   where r = (⟨X⟩, ⟨Y⟩, ⟨Z⟩) is the Bloch vector
```

Measure in X, Y, Z bases:
```
⟨X⟩ = Tr(Xρ) = P(+1)_X - P(-1)_X
⟨Y⟩ = Tr(Yρ) = P(+1)_Y - P(-1)_Y
⟨Z⟩ = Tr(Zρ) = P(+1)_Z - P(-1)_Z
```

For n qubits: need 4ⁿ - 1 expectation values. Exponential in n → state tomography is expensive!

---

## B.5 Quantum SDN (Software-Defined Networking)

### Classical SDN recap:
```
┌─────────────────────────────────┐
│  SDN Controller (OpenDaylight)  │  ← centralized control plane
│  - global network view          │
│  - flow table computation       │
└─────────────┬───────────────────┘
              │ OpenFlow / REST API
    ┌─────────┴──────────┐
    │  Data Plane        │  ← forwarding only
    │  switches/routers  │
    └────────────────────┘
```

### Quantum SDN adaptation:
```
┌────────────────────────────────────────┐
│  Q-SDN Controller                      │
│  - entanglement routing                │
│  - protocol selection (DV vs CV)       │
│  - purification scheduling             │
│  - BSM timing coordination             │
└─────────────────┬──────────────────────┘
                  │ REST API + classical channel
    ┌─────────────┴──────────────┐
    │ Quantum Data Plane         │
    │ - quantum memories         │
    │ - BSM stations (repeaters) │
    │ - QKD transceivers         │
    └────────────────────────────┘
```

### API design (your work — explain clearly):

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional
import sqlite3, time, hashlib

app = FastAPI(title="Q-SDN Controller API")

# --- Data Models ---
class EntanglementRequest(BaseModel):
    source_node: str        # "Alice"
    dest_node: str          # "Bob"
    protocol: str           # "BB84" | "MDI-QKD" | "CV-QKD"
    min_fidelity: float     # 0.0 - 1.0
    key_length_bits: int    # desired final key length

class Session(BaseModel):
    session_id: str
    nodes: list[str]
    protocol: str
    timestamp: float
    status: str             # "active" | "completed" | "aborted"

# --- DB setup ---
def get_db():
    conn = sqlite3.connect("qsdn.sqlite")
    conn.row_factory = sqlite3.Row
    return conn

# --- Routes ---

@app.post("/sessions/create")
def create_session(req: EntanglementRequest):
    """
    Creates a QKD session between source and dest nodes.
    Returns session ID and protocol parameters.
    """
    session_id = hashlib.sha256(
        f"{req.source_node}{req.dest_node}{time.time()}".encode()
    ).hexdigest()[:16]
    
    conn = get_db()
    conn.execute(
        "INSERT INTO sessions VALUES (?, ?, ?, ?, ?)",
        (session_id, f"{req.source_node},{req.dest_node}", 
         req.protocol, time.time(), "active")
    )
    conn.commit()
    
    return {
        "session_id": session_id,
        "selected_protocol": req.protocol,
        "estimated_rate_bps": compute_key_rate(req.protocol, req.min_fidelity)
    }

@app.get("/sessions/{session_id}")
def get_session(session_id: str):
    conn = get_db()
    row = conn.execute(
        "SELECT * FROM sessions WHERE session_id=?", (session_id,)
    ).fetchone()
    return dict(row) if row else {"error": "not found"}

@app.delete("/sessions/{session_id}")
def terminate_session(session_id: str):
    conn = get_db()
    conn.execute(
        "UPDATE sessions SET status='terminated' WHERE session_id=?",
        (session_id,)
    )
    conn.commit()
    return {"status": "terminated"}

def compute_key_rate(protocol: str, fidelity: float) -> float:
    """Rough key rate estimate (bps) based on protocol + fidelity."""
    # In practice: query current channel measurements from nodes
    base_rates = {"BB84": 1e6, "MDI-QKD": 1e5, "CV-QKD": 5e6, "TF-QKD": 2e5}
    return base_rates.get(protocol, 1e5) * fidelity
```

---

# BRANCH C: Post-Quantum Cryptography

## C.1 The Quantum Threat to Classical Crypto

| Algorithm | Classical Security | After Shor | After Grover |
|-----------|-------------------|------------|--------------|
| RSA-2048  | 2¹⁰⁴⁸             | Broken     | N/A          |
| ECC-256   | 2¹²⁸              | Broken     | N/A          |
| AES-256   | 2²⁵⁶              | N/A        | 2¹²⁸ (safe if key doubled) |
| SHA-256   | 2²⁵⁶              | N/A        | 2¹²⁸ (collision security halved) |

**Shor's Algorithm complexity:** O((log N)³) → polynomial. Destroys RSA, DH, ECC.
**Grover's Algorithm complexity:** O(√N) → quadratic speedup. Weakens symmetric crypto.

---

## C.2 Lattice-Based Cryptography (Core of NIST Standards)

### Hard Problems:

**Learning With Errors (LWE):**
```
Given: matrix A ∈ ℤqⁿˣᵐ, vector b = As + e  (s = secret, e = small error)
Find:  s
```
Believed hard even for quantum computers. Reduction from worst-case SVP.

**Ring-LWE (RLWE):**
Same but in polynomial ring R = ℤ_q[x]/(xⁿ + 1). More efficient — O(n log n) vs O(n²).

### CRYSTALS-Kyber (KEM):

```
Key Generation:
  A ← R_q^{k×k}  (random matrix)
  s, e ← χ^k    (small error)
  t = As + e
  pk = (A, t),   sk = s

Encapsulation:
  r, e₁, e₂ ← χ^k
  u = Aᵀr + e₁
  v = tᵀr + e₂ + ⌊q/2⌋m    (m = message/key seed)
  ciphertext = (u, v)

Decapsulation:
  m' = ⌊ v - sᵀu ⌉_{round}
```

Security parameter: Kyber-512 (128-bit classical + quantum), Kyber-768 (192-bit), Kyber-1024 (256-bit).

### CRYSTALS-Dilithium (Signature):
Based on Fiat-Shamir with aborts. Module-LWE + Module-SIS hardness.

### SPHINCS+ (Hash-based):
- No lattice math — pure hash functions
- Stateless: no need to track used keys
- Slower signing than Dilithium, but "bet-hedging" security
- Used where long-term trust in lattice hardness is uncertain

---

## C.3 Your PQC Browser Architecture (Army Submission)

### System Architecture:

```
┌─────────────────────────────────────────────────────────┐
│                    PyQt5 UI Layer                        │
│  Security dashboard, threat map, connection status       │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│           Layer 3: PQC Security Layer                    │
│  - Kyber-768 key encapsulation                          │
│  - Dilithium-3 message signing                          │
│  - AES-256-GCM symmetric encryption                     │
│  - Timestamped packet headers (replay protection)       │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│           Layer 2: P2P Routing Network                   │
│  - LPI (Low Probability of Intercept) inspired by DRDO  │
│  - Frequency hopping / spread spectrum concept          │
│  - Onion-routing style relay                            │
│  - Offline graph-based routing (no central DNS)         │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│           Layer 1: Secure Core Rendering                 │
│  - Hardened browser engine (Chromium stripped)          │
│  - No JavaScript JIT (reduce attack surface)            │
│  - Sandboxed renderer process                           │
│  - Certificate pinning                                  │
└──────────────────────────────────────────────────────────┘
         ↓ FastAPI (Unix socket — no network exposure)
┌──────────────────────────────────────────────────────────┐
│           SQLite Backend                                  │
│  Tables: sessions, certificates, history, threat_log    │
└──────────────────────────────────────────────────────────┘
```

### Why Unix sockets (not TCP)?
- Unix domain sockets never leave the kernel
- No port exposure → no remote exploit surface
- ~10x faster than localhost TCP (no IP stack overhead)

### Timestamped Packets (Post-Intercept Sync):
```python
import time, hashlib, hmac

def create_packet(payload: bytes, pqc_key: bytes) -> dict:
    timestamp = int(time.time() * 1000)  # milliseconds
    nonce = os.urandom(16)
    
    # HMAC over timestamp + payload (replay protection)
    mac = hmac.new(pqc_key, 
                   timestamp.to_bytes(8, 'big') + nonce + payload, 
                   hashlib.sha3_256).digest()
    return {
        "timestamp": timestamp,
        "nonce": nonce.hex(),
        "payload": payload.hex(),
        "mac": mac.hex()
    }

def verify_packet(packet: dict, pqc_key: bytes, max_skew_ms=5000) -> bool:
    now = int(time.time() * 1000)
    ts = packet["timestamp"]
    
    # Replay protection: reject if too old
    if abs(now - ts) > max_skew_ms:
        return False
    
    # Verify MAC
    payload = bytes.fromhex(packet["payload"])
    nonce   = bytes.fromhex(packet["nonce"])
    expected_mac = hmac.new(
        pqc_key, ts.to_bytes(8, 'big') + nonce + payload, hashlib.sha3_256
    ).digest()
    
    return hmac.compare_digest(expected_mac, bytes.fromhex(packet["mac"]))
```

---

# BRANCH D: Quantum Error Correction

## D.1 Why QEC is Fundamentally Different

Classical: duplicate bit → majority vote → done.

Quantum problems:
1. **No cloning** → can't copy qubit
2. **Continuous errors** → not just bit flips, but arbitrary rotations
3. **Measurement destroys state** → can't check qubit without disturbing it

Solutions:
1. **Redundancy in code space** (not cloning — encoding)
2. **Discretization via syndrome measurement** (arbitrary → Pauli errors)
3. **Non-destructive syndrome measurement** (ancilla qubits)

---

## D.2 3-Qubit Repetition Code

### Encoding:
```
|0⟩_L = |000⟩
|1⟩_L = |111⟩
|ψ⟩ = α|0⟩ + β|1⟩  →  α|000⟩ + β|111⟩
```

Encoding circuit:
```
|ψ⟩ = α|0⟩+β|1⟩ ─────●─────●──── q₀ (original)
|0⟩              ──CNOT────────── q₁ (copy)
|0⟩              ──────────CNOT── q₂ (copy)
```

### Error Detection:
Measure stabilizers: Z₁Z₂ and Z₂Z₃ (parities, without measuring individual qubits)

| Z₁Z₂ | Z₂Z₃ | Error |
|-------|-------|-------|
|  +1   |  +1   | None  |
|  -1   |  +1   | q₁ flipped |
|  +1   |  -1   | q₂ flipped |
|  -1   |  -1   | q₀ flipped |

### Correction: Apply X to identified qubit. ✓

**Limitation:** Only corrects bit-flip (X) errors. Phase flip (Z) requires a different code or the Shor code.

---

## D.3 Shor Code (9 qubits)

Corrects both X and Z errors by concatenating:
```
|0⟩_L = (|+⟩|+⟩|+⟩)(|+⟩|+⟩|+⟩)(|+⟩|+⟩|+⟩) ← phase-flip outer code
|1⟩_L = (|-⟩|-⟩|-⟩)(|-⟩|-⟩|-⟩)(|-⟩|-⟩|-⟩)

Each |±⟩ → |±±±⟩ in bit-flip inner code
```

→ First QEC code to correct arbitrary single-qubit errors.

---

## D.4 Stabilizer Formalism

A stabilizer code is defined by a group S of Pauli operators (the stabilizers) that fix the code space:
```
g|ψ⟩ = |ψ⟩  for all g ∈ S
```

Syndrome measurement: measure each stabilizer generator.
- Result +1 → no error of that type
- Result -1 → error anticommutes with this stabilizer

**Key advantage:** Error detection without learning the encoded state.

---

## D.5 Surface Codes

Qubits arranged on 2D grid. Stabilizers are:
- **Z-stabilizers (plaquettes):** ZZZZ on each square
- **X-stabilizers (vertices):** XXXX on each cross

Error threshold: ~1% per gate. Most hardware-friendly known code.

```
· — · — · — ·
|   |   |   |
· — · — · — ·
|   |   |   |
· — · — · — ·
```
Data qubits at · (corners and edges), ancilla qubits at centers of plaquettes/crosses.

Logical error rate: L ~ (p/p_th)^{d/2} where d = code distance, p = physical error rate.

---

## D.6 Privacy Amplification (Toeplitz)

After error correction, Eve still has partial information. 

**Goal:** Extract m bits where Eve has negligible info, even if she knows n-m bits.

**Universal hashing:** Choose random hash function h: {0,1}ⁿ → {0,1}^m.
Final key = h(confirmed_key).

**Toeplitz construction:**
```
A Toeplitz matrix T is defined by its first row and first column:
T_{ij} = t_{i-j}   (depends only on i-j)
```
An n×m Toeplitz matrix needs only n+m-1 random bits to specify.
T is picked uniformly at random → sends as seed over public channel.
Final key = T × confirmed_key (GF(2) arithmetic)

```python
import numpy as np

def toeplitz_privacy_amplification(confirmed_key: list[int], 
                                   output_length: int) -> list[int]:
    """
    Privacy amplification using random Toeplitz matrix (GF2).
    confirmed_key: list of 0/1 bits (length n)
    output_length: m (< n)
    """
    n = len(confirmed_key)
    m = output_length
    
    # Generate random Toeplitz matrix: needs n+m-1 bits
    seed = np.random.randint(0, 2, n + m - 1)
    
    # Build Toeplitz matrix T (m×n)
    T = np.zeros((m, n), dtype=int)
    for i in range(m):
        for j in range(n):
            T[i, j] = seed[j - i + m - 1]
    
    # Final key = T × confirmed_key (mod 2)
    key_vec = np.array(confirmed_key)
    final_key = (T @ key_vec) % 2
    
    return final_key.tolist()

# Test
raw = [1, 0, 1, 1, 0, 0, 1, 0, 1, 1, 0, 1, 0, 1, 0, 1]  # 16 bits
final = toeplitz_privacy_amplification(raw, output_length=8)
print(f"Input:  {raw}  ({len(raw)} bits)")
print(f"Output: {final}  ({len(final)} bits)")
```

---

# BRANCH E: IBM Quantum / Qiskit

## E.1 Bell States + CHSH (S > 2)

### CHSH inequality:
Measure two qubits in angles a, a' (Alice) and b, b' (Bob):
```
S = E(a,b) - E(a,b') + E(a',b) + E(a',b')
```
Classical: |S| ≤ 2 (Bell inequality)
Quantum maximum: |S| = 2√2 ≈ 2.828 (Tsirelson bound)
Optimal angles: a=0°, a'=90°, b=45°, b'=135°

```python
from qiskit import QuantumCircuit, transpile
from qiskit_aer import AerSimulator
import numpy as np

def chsh_correlation(theta_a: float, theta_b: float, shots=8192) -> float:
    """
    Measure ⟨AB⟩ correlation for CHSH with measurement angles theta_a, theta_b.
    Returns expectation value E(a,b) ∈ [-1, +1].
    """
    qc = QuantumCircuit(2, 2)
    
    # Prepare |Φ⁺⟩
    qc.h(0)
    qc.cx(0, 1)
    
    # Rotate measurement bases
    qc.ry(2 * theta_a, 0)    # Alice measures at angle theta_a
    qc.ry(2 * theta_b, 1)    # Bob measures at angle theta_b
    qc.measure([0, 1], [0, 1])
    
    sim = AerSimulator()
    compiled = transpile(qc, sim)
    counts = sim.run(compiled, shots=shots).result().get_counts()
    
    # E(a,b) = P(00) + P(11) - P(01) - P(10)
    p00 = counts.get('00', 0) / shots
    p11 = counts.get('11', 0) / shots
    p01 = counts.get('01', 0) / shots
    p10 = counts.get('10', 0) / shots
    
    return p00 + p11 - p01 - p10

# CHSH measurement
a, a_ = 0, np.pi/4           # Alice angles (0° and 45°)
b, b_ = np.pi/8, -np.pi/8   # Bob angles (22.5° and -22.5°)

E_ab   = chsh_correlation(a, b)
E_ab_  = chsh_correlation(a, b_)
E_a_b  = chsh_correlation(a_, b)
E_a_b_ = chsh_correlation(a_, b_)

S = E_ab - E_ab_ + E_a_b + E_a_b_
print(f"CHSH S = {S:.4f}  (Classical max: 2.0000, Quantum max: {2*np.sqrt(2):.4f})")
print(f"Bell inequality violated: {abs(S) > 2}")
```

---

## E.2 Deutsch-Jozsa Algorithm

**Problem:** Given f:{0,1}ⁿ → {0,1}, promised constant or balanced. Determine which.
**Classical:** O(2^{n-1} + 1) queries worst case.
**Quantum:** O(1) query!

```python
def deutsch_jozsa(oracle_type='balanced', n=3):
    """
    n-bit Deutsch-Jozsa.
    oracle_type: 'constant' (f always 0) or 'balanced' (f(x)=x₀)
    """
    qc = QuantumCircuit(n + 1, n)   # n input qubits + 1 ancilla
    
    # Initialize ancilla to |−⟩ = H|1⟩
    qc.x(n)       # flip ancilla to |1⟩
    qc.h(n)       # Hadamard → |−⟩
    
    # Apply H to all input qubits → equal superposition
    qc.h(range(n))
    
    qc.barrier()
    
    # Oracle
    if oracle_type == 'constant':
        pass        # identity oracle (f=0 everywhere)
    elif oracle_type == 'balanced':
        # f(x) = x₀ (first bit of input)
        qc.cx(0, n)   # CNOT: control=q0, target=ancilla
    
    qc.barrier()
    
    # Apply H to input qubits again
    qc.h(range(n))
    
    # Measure input qubits
    qc.measure(range(n), range(n))
    
    return qc

for oracle in ['constant', 'balanced']:
    qc = deutsch_jozsa(oracle_type=oracle, n=3)
    sim = AerSimulator()
    result = sim.run(transpile(qc, sim), shots=100).result()
    counts = result.get_counts()
    # Constant → always measures '000...' ; Balanced → never '000...'
    verdict = 'constant' if '000' in counts else 'balanced'
    print(f"Oracle: {oracle:10s} | Measurement: {counts} | Verdict: {verdict}")
```

---

# BRANCH F: Advanced SDN Projects

## F.1 Hybrid DV-CV SDN Simulator

### Design Decision Matrix:

| Metric | DV-QKD (BB84/MDI) | CV-QKD (GMCS) |
|--------|-------------------|----------------|
| Photon source | Single photon | Coherent laser |
| Detector | SNSPD (expensive, -200°C) | Homodyne (room temp) |
| Key rate | Lower | Higher (short range) |
| Max distance | ~500 km (TF-QKD) | ~100 km practical |
| Integration | Hard | Telecom-compatible |

**SDN Controller decides** which protocol to assign to each link based on:
- Measured channel loss
- Available hardware at nodes
- Required key rate and security level
- Distance between nodes

### API for protocol selection:

```python
def select_protocol(link: dict) -> str:
    """
    Heuristic protocol selection for Q-SDN controller.
    link = {distance_km, loss_dB, has_snspd, has_quantum_memory}
    """
    d  = link['distance_km']
    L  = link['loss_dB']
    snspd = link['has_snspd']
    qm    = link['has_quantum_memory']
    
    if d > 300 and snspd:
        return 'TF-QKD'           # beats PLOB at long distance
    elif d > 100 and qm:
        return 'MDI-QKD'          # repeater node, secure detectors
    elif d < 80 and not snspd:
        return 'CV-QKD'           # cost-effective short range
    else:
        return 'BB84'             # default
```

---

## F.2 ML-Optimized Entanglement Distribution

### Problem:
In a multi-node quantum network with noisy links and limited quantum memories (decoherence time T₂), when and how do you attempt entanglement generation to maximize end-to-end entanglement rate?

### RL Formulation:
```
State:  s = (memory_states, link_qualities, queue_lengths, T₂ remaining)
Action: a = (try_entangle on link L₁, wait, purify pair P, swap)
Reward: r = +1 if end-to-end Bell pair delivered; -lifetime_penalty
```

### Simplified ML approach (supervised):

```python
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier

def build_training_data():
    """
    Generate (link_state → optimal_action) pairs via simulation.
    Features: [link_fidelity, memory_age, queue_depth, distance_km]
    Label: action (0=wait, 1=entangle, 2=purify, 3=swap)
    """
    # In practice: run NetSquid/SeQUeNCe simulation → collect (state, optimal_action)
    np.random.seed(42)
    n = 1000
    X = np.column_stack([
        np.random.uniform(0.7, 1.0, n),   # link_fidelity
        np.random.exponential(10, n),       # memory_age (ms)
        np.random.randint(0, 5, n),        # queue_depth
        np.random.uniform(10, 200, n)       # distance_km
    ])
    # Heuristic labels (replace with RL-derived optimal policy)
    y = np.where(X[:,0] > 0.9, 1,         # high fidelity → entangle
        np.where(X[:,0] < 0.75, 2,        # low fidelity → purify
        np.where(X[:,1] > 50, 3, 0)))      # old memory → swap or wait
    
    return X, y

X, y = build_training_data()
clf = GradientBoostingClassifier(n_estimators=100, max_depth=4)
clf.fit(X, y)

# Predict action for a new link state
link_state = np.array([[0.82, 15.0, 2, 80.0]])
action = clf.predict(link_state)[0]
action_names = {0: 'wait', 1: 'entangle', 2: 'purify', 3: 'swap'}
print(f"Recommended action: {action_names[action]}")
```

---

## F.3 Quantum Dot Source Characterization

### Key Metrics:

**1. g²(0) — Second-order correlation (single-photon purity):**
```
g²(τ) = ⟨a†(t)a†(t+τ)a(t+τ)a(t)⟩ / ⟨a†a⟩²
```
- g²(0) = 0: perfect single-photon emitter (no two-photon events)
- g²(0) = 1: coherent (classical) light
- g²(0) > 1: bunched (thermal) light
- Threshold: g²(0) < 0.5 → non-classical (single-photon source)

**Measurement:** Hanbury Brown-Twiss (HBT) setup
```
Photon source → 50:50 beamsplitter → Detector 1
                                   → Detector 2
                                   → Time-correlator
g²(τ) = coincidences at delay τ
```

**2. Indistinguishability — Hong-Ou-Mandel (HOM) dip:**
```
Two identical photons on a 50:50 BS → always exit together (no coincidences)
HOM visibility = (R_classical - R_quantum) / R_classical
```
Perfect indistinguishability → V_HOM = 1.

**3. Brightness:** photons collected per excitation pulse. Limited by:
- Collection efficiency (NA of lens)
- Quantum efficiency of QD
- Above-band vs resonant excitation

---

# BRANCH G: Interview Master Q&A

## G.1 Core Conceptual

**Q: Why can't we just use AES for key exchange?**
> AES is symmetric — both parties need the same key. The problem is securely sharing that key over an insecure channel. QKD solves key distribution using physics, not computational hardness.

**Q: What's the fundamental difference between QKD and PQC?**
> QKD: hardware-based, security from laws of physics (no-cloning, measurement disturbance). Requires quantum channels. PQC: software-based, security from computational hardness (lattice problems). Runs on classical hardware. They're complementary — PQC for software/internet, QKD for high-security physical links.

**Q: If Eve measures in the wrong basis only 50% of the time, why does she introduce only 25% errors?**
> Eve picks wrong basis 50% of the time. When she does, her re-prepared qubit has a 50% chance of being wrong in Bob's correct basis. So: P(error) = P(wrong basis) × P(wrong state given wrong basis) = 0.5 × 0.5 = 0.25.

**Q: Can MDI-QKD work if Charlie is completely malicious?**
> Yes — that's the whole point. Charlie can be controlled by Eve. He only announces classical BSM results. Alice and Bob can verify the statistics of their own data independently to detect any manipulation. Security proof covers malicious Charlie.

**Q: Why does TF-QKD beat the PLOB bound?**
> PLOB bound applies to entanglement transmission protocols. TF-QKD uses single-photon interference rather than entanglement. The key correlations are established through which-path interference at Charlie, and the key rate scales as √η (square root of transmittance) instead of η.

---

## G.2 Math/Derivation

**Q: Derive the secure key rate for BB84.**
```
Information-theoretic framework (Devetak-Winter):
K ≥ I(A:B) - χ(A:E)

For BB84 with QBER = Q:
I(A:B) = 1 - H(Q)         [Shannon mutual information]
χ(A:E) ≤ H(Q)             [Holevo bound for collective attacks]

K ≥ 1 - 2H(Q)
where H(Q) = -Q log₂Q - (1-Q) log₂(1-Q)

K > 0 when Q < 11% (exactly).
```

**Q: What is the Holevo bound and why does it limit Eve?**
```
For ensemble {pᵢ, ρᵢ}, Holevo bound χ limits accessible info:
I(X:M) ≤ χ({pᵢ, ρᵢ}) = S(Σ pᵢρᵢ) - Σ pᵢS(ρᵢ)
where S = von Neumann entropy = -Tr(ρ log ρ)

Eve's state is highly mixed → S is high → χ is bounded.
```

**Q: Show that CHSH maximum for quantum is 2√2.**
```
E(a,b) = -cos(a-b) for entangled state |Φ⁺⟩

S = E(a,b) - E(a,b') + E(a',b) + E(a',b')
  = -cos(a-b) + cos(a-b') - cos(a'-b) - cos(a'-b')

Maximize over all angles... optimal: a=0, a'=π/2, b=π/4, b'=-π/4
S = -cos(π/4) + cos(π/4) - cos(π/4) - cos(-π/4)
  = -4cos(π/4) ... wait, let me redo:
  = cos(π/4) + cos(π/4) + cos(π/4) + cos(π/4) = 4/√2 = 2√2 ✓
```

---

## G.3 System Design

**Q: Design a QKD-secured communication system for two Army posts 200 km apart.**

```
System Design Answer:

Physical Layer:
- Dark fiber (dedicated, no WDM interference)
- Protocol: TF-QKD (distance > 100 km, need to beat PLOB)
- Detectors: SNSPD (superconducting nanowire, >90% efficiency)
- Quantum memory at mid-node for entanglement swapping if needed

Network Layer:
- Quantum SDN controller at each site
- REST API for session management
- Classical authenticated channel (Wegman-Carter MACs) alongside quantum

Key Management:
- One-Time Pad (OTP) for most sensitive traffic (keys stored in HSM)
- PQC (Kyber + Dilithium) as fallback if QKD unavailable
- Key storage: FIPS 140-3 Level 3 HSM

Monitoring:
- Real-time QBER monitoring → alert if > 5% (anomaly)
- Automatic abort + key zeroization if QBER > 11%
- Tamper-evident fiber monitoring (OTDR)

Redundancy:
- Free-space QKD backup link (weather permitting)
- PQC-encrypted classical backup for emergency
```

**Q: How would you test your QuNetSim implementation for correctness?**
```
1. Mathematical correctness:
   - For teleportation: measure received qubit in same basis as input state
   - Verify fidelity F = |⟨ψ_in|ψ_out⟩|² ≈ 1.0 (ideal) 
   - With noise: verify F matches theoretical F = (1 + 3·η)/4 for depolarizing noise

2. Statistical correctness:
   - Run N=1000 teleportations of known states {|0⟩, |1⟩, |+⟩, |-⟩}
   - Measure F for each: should match ≥ 0.99 for ideal simulation
   
3. Protocol correctness:
   - For BB84: verify QBER ≈ 0% with no Eve, ≈ 25% with full intercept-resend
   - For entanglement swapping: verify resulting pair has F ≥ F_link² (fidelity composition)

4. Edge cases:
   - Photon loss (η=0): verify graceful failure (empty key)
   - Maximum noise: verify protocol abort at correct QBER threshold
```

---

## G.4 FPGA / Hardware Questions (from JD)

**Q: What FPGA skills are relevant to quantum networks?**

In QKD hardware:
- **Timing electronics:** FPGA implements Time-Correlated Single Photon Counting (TCSPC). Timestamp photon arrival times with sub-nanosecond resolution (typically 100ps bins).
- **Sifting hardware:** FPGA compares Alice/Bob basis choices in real time (GHz clock rate needed).
- **Post-processing acceleration:** LDPC decoding, privacy amplification (Toeplitz matrix-vector multiply) on FPGA >> CPU speed.
- **Random number generation:** FPGA samples entropy from shot noise, runs NIST randomness tests in hardware.

**Q: What is FPGA-based SDN?**
- P4 language: program packet processing pipeline on FPGA
- OpenFlow hardware offload: move flow table lookup to FPGA line rate
- In quantum SDN: FPGA handles classical control messages (BSM results, basis announcements) at ns latency — critical because quantum memories have limited coherence time

---

## G.5 Fiber Optics / Instrumentation (from JD)

**Key concepts:**

| Term | Meaning | Quantum relevance |
|------|---------|-------------------|
| Attenuation | Loss in fiber (dB/km) | 0.2 dB/km @ 1550nm → limits QKD range |
| Chromatic dispersion | Pulse spreading | Affects timing of single photons |
| PMD | Polarization mode dispersion | Scrambles qubit polarization → requires active compensation |
| WDM | Wavelength division multiplexing | QKD channel co-propagates with classical traffic at different λ |
| OTDR | Optical time-domain reflectometer | Detects physical tampering on QKD fiber |
| SNSPD | Superconducting nanowire single-photon detector | State-of-art: >90% efficiency, ~10 dark counts/s |
| SPAD | Single-photon avalanche diode | Cheaper, less efficient, higher dark counts |

---

# Quick Revision Summary

```
MUST KNOW COLD:
─────────────────────────────────────────────────────────
1. BB84: states, protocol, QBER threshold (11%), key rate formula
2. MDI-QKD: why (detector attacks), how (BSM at Charlie), decoy states
3. TF-QKD: PLOB bound, √η scaling, why it matters
4. CV-QKD: Gaussian modulation, homodyne, reconciliation
5. No-cloning theorem proof sketch
6. Bell states: all 4, how to make them (H + CNOT)
7. CHSH: S_max = 2√2, optimal angles
8. Teleportation: 4-step protocol, why 2 classical bits needed
9. Entanglement swapping: how it extends range
10. DEJMPS: what it solves (fidelity upgrade), when to keep/discard
11. 3-qubit code: syndrome table, correction
12. Surface codes: threshold ~1%, hardware advantage
13. Privacy amplification: Toeplitz, why it works
14. Kyber: lattice basis, KEM structure
15. Your PQC architecture: 3 layers, Unix sockets rationale
─────────────────────────────────────────────────────────

YOUR PITCH (30 seconds):
"I spent 2 years systematically building from quantum foundations up
to full protocol implementations — BB84 through TF-QKD in Qiskit,
network simulations in QuNetSim and NetSquid, and a real PQC system
submitted to the Army. I'm particularly interested in the hardware-
software interface: FPGA-accelerated QKD post-processing and SDN
control of quantum networks — exactly what TAHQEECAT needs."
```

---

*Document generated for TAHQEECAT interview preparation. All code tested conceptually; run in fresh Python environment with `qiskit`, `qiskit-aer`, `qunetsim`, `fastapi`, `numpy`, `sklearn` installed.*
