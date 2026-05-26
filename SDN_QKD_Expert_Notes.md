# SDN × QKD — Subject Matter Expert Reference
### Software-Defined Quantum Key Distribution Networks: State of the Art, Architecture, and India-Scale Deployment

> **Scope:** Complete technical reference covering SDN theory → implementation, QKD physics → protocol stack, their integration, your QKDN SaaS v2 architecture analysis, Mininet/C-band local testing, and an India-scale deployment model.  
> **Target depth:** Research/industry expert level. Suitable as pre-interview prep, project documentation backbone, and literature review supplement.

---

## Table of Contents

1. [Part I — Software Defined Networking](#part-i--software-defined-networking)
   - 1.1 Foundational Theory
   - 1.2 OpenFlow Deep Dive
   - 1.3 SDN Controllers (Ryu, ONOS, ODL)
   - 1.4 Data Plane Abstractions
   - 1.5 SDN Security & Fault Tolerance
   - 1.6 Mininet Lab Environment
2. [Part II — Quantum Key Distribution](#part-ii--quantum-key-distribution)
   - 2.1 Quantum Information Foundations
   - 2.2 BB84 Protocol — Full Technical Walkthrough
   - 2.3 Security Proofs & QBER Analysis
   - 2.4 Trusted-Node vs Trusted-Node-Free Architectures
   - 2.5 Advanced QKD Protocols
   - 2.6 Hardware Layer: Sources, Detectors, Fiber
   - 2.7 ETSI QKD Standards (014/015/018)
3. [Part III — SDN-QKD Integration](#part-iii--sdn-qkd-integration)
   - 3.1 Why SDN for QKD Networks
   - 3.2 Control-Plane Design for QKDN
   - 3.3 Key Management Service Architecture
   - 3.4 Multi-Tenant QKD as a Service
   - 3.5 Observability Stack
4. [Part IV — Your QKDN SaaS v2 Architecture Analysis](#part-iv--your-qkdn-saas-v2-architecture-analysis)
   - 4.1 Component-by-Component Review
   - 4.2 Gaps, Improvements, and Research Opportunities
   - 4.3 Trusted-Node-Free Claim — Is It Valid?
5. [Part V — C-Band & Mininet Local Testing](#part-v--c-band--mininet-local-testing)
   - 5.1 C-Band Fiber Optics Primer
   - 5.2 Mininet Topology for QKDN Emulation
   - 5.3 WDM Coexistence (Quantum + Classical on Same Fiber)
   - 5.4 Test Matrix and Benchmarking Framework
6. [Part VI — India-Scale QKDN Deployment](#part-vi--india-scale-qkdn-deployment)
   - 6.1 India Fiber Infrastructure Landscape
   - 6.2 Phased Rollout Strategy
   - 6.3 Regulatory and Spectrum Considerations
   - 6.4 Relevant Indian Programs (NM-QTA, NQMP, TAHQEECAT)
7. [Part VII — Post-Quantum Cryptography Hybrid Layer](#part-vii--post-quantum-cryptography-hybrid-layer)
8. [Part VIII — Research Frontiers](#part-viii--research-frontiers)
9. [Appendix A — Key Equations Reference Sheet](#appendix-a--key-equations-reference-sheet)
10. [Appendix B — Complete Resource Library](#appendix-b--complete-resource-library)

---

# Part I — Software Defined Networking

## 1.1 Foundational Theory

### The Core Paradigm Shift

Traditional networking conflates three logically distinct planes inside each device:

| Plane | Function | Location (Traditional) | Location (SDN) |
|-------|----------|------------------------|----------------|
| **Data Plane** | Forward packets per forwarding table | Switch ASIC | Switch ASIC (unchanged) |
| **Control Plane** | Compute forwarding tables (routing protocols) | CPU on each device | Centralized controller |
| **Management Plane** | Configuration, monitoring, telemetry | Per-device CLI/SNMP | Controller + NMS |

SDN extracts the control plane into a logically centralized controller. The southbound interface (typically OpenFlow) programs the data plane remotely. This gives:

- **Global visibility**: Controller has full topology view — no distributed convergence.
- **Programmability**: Network behavior changed in software, not by touching hardware.
- **Abstraction**: Applications see a network API, not raw forwarding tables.

### The Three-Layer SDN Stack

```
┌──────────────────────────────────────────────────────────────┐
│  APPLICATION LAYER                                           │
│  Network apps: load balancing, firewall, QKD orchestration   │
│  communicate via NORTHBOUND API (REST, gRPC, Intent API)     │
├──────────────────────────────────────────────────────────────┤
│  CONTROL LAYER                                               │
│  SDN Controller (Ryu / ONOS / ODL)                          │
│  Topology discovery, path computation, policy enforcement    │
│  communicates via SOUTHBOUND API (OpenFlow, NETCONF, P4)     │
├──────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE LAYER (Data Plane)                           │
│  OpenFlow switches, OVS, physical forwarding ASICs          │
│  Execute flow rules; send unmatched packets to controller    │
└──────────────────────────────────────────────────────────────┘
```

### Why SDN Matters for QKD Specifically

QKD imposes unique network constraints:
1. **Key rate ≪ data rate** — quantum channels are bandwidth-limited (kbps range). Traffic must be routed to conserve precious key material.
2. **QBER sensitivity** — path quality (loss, PMD, noise photons) directly degrades quantum bit error rate. SDN can reroute dynamically.
3. **Link failure recovery** — trusted-node-free networks need fast path re-establishment when a fiber segment degrades.
4. **Multi-tenancy** — QKD-as-a-Service requires tenant isolation at the key management level, which maps onto SDN's virtual network slicing.

---

## 1.2 OpenFlow Deep Dive

### Flow Table Architecture (OF 1.3)

Each OF1.3 switch maintains a **pipeline** of flow tables (Table 0–N). A packet traverses the pipeline until it hits a **goto-table** instruction or terminates with an action.

```
Packet In
   │
   ▼
┌─────────────────────────────────────────────┐
│ Table 0: Ingress classifier                 │
│  Match: in_port=1, eth_type=0x0800          │
│  Action: goto Table 1                       │
└─────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────┐
│ Table 1: IP forwarding                      │
│  Match: ipv4_dst=10.0.1.2/24               │
│  Action: output port 3                      │
└─────────────────────────────────────────────┘
```

**Match Fields (OF 1.3 OXM encoding):**

| Field | Bits | Notes |
|-------|------|-------|
| `in_port` | 32 | Physical or logical port |
| `eth_src` / `eth_dst` | 48 each | MAC address |
| `eth_type` | 16 | 0x0800=IPv4, 0x86DD=IPv6, 0x8100=VLAN |
| `ip_proto` | 8 | 6=TCP, 17=UDP, etc. |
| `ipv4_src` / `ipv4_dst` | 32 | With optional mask |
| `tcp_dst` / `udp_dst` | 16 | Transport layer port |
| `metadata` | 64 | Pipeline-internal carry field |

**Instruction Types** (executed when rule matches):

| Instruction | Effect |
|-------------|--------|
| `Apply-Actions` | Execute action set immediately |
| `Write-Actions` | Accumulate into packet's action set |
| `Goto-Table N` | Continue pipeline at table N |
| `Meter` | Apply a meter band (rate limiting) |
| `Write-Metadata` | Set metadata field for next table |
| `Clear-Actions` | Empty accumulated action set |

**Action Types:**

- `OUTPUT(port)` — forward to port; port=CONTROLLER sends Packet-In to controller
- `SET_FIELD(field, value)` — rewrite header fields (VLAN push/pop, IP rewrite)
- `GROUP(group_id)` — forward to a group table (used for multicast, ECMP, failover)
- `DROP` — implicit when no match and no table-miss entry

### Group Tables — Key for QKD Failover

Group tables allow advanced forwarding behaviors:

| Type | Semantics | QKD Use Case |
|------|-----------|--------------|
| `ALL` | Send to all buckets (multicast) | Replicate key refresh triggers |
| `SELECT` | Load balance across buckets | Multi-path QKD key delivery |
| `INDIRECT` | Single bucket (indirect forwarding) | Simple next-hop abstraction |
| `FAST_FAILOVER` | Use first live bucket (watches liveness) | Automatic fiber-cut recovery |

**FAST_FAILOVER for QKD links:**
```python
# Ryu: create a fast-failover group
ofp = datapath.ofproto
ofp_parser = datapath.ofproto_parser

buckets = [
    ofp_parser.OFPBucket(
        watch_port=PRIMARY_PORT,
        watch_group=ofp.OFPG_ANY,
        actions=[ofp_parser.OFPActionOutput(PRIMARY_PORT)]
    ),
    ofp_parser.OFPBucket(
        watch_port=BACKUP_PORT,
        watch_group=ofp.OFPG_ANY,
        actions=[ofp_parser.OFPActionOutput(BACKUP_PORT)]
    )
]
req = ofp_parser.OFPGroupMod(
    datapath, ofp.OFPGC_ADD, ofp.OFPGT_FF,
    group_id=1, buckets=buckets
)
datapath.send_msg(req)
```

### OpenFlow Messages

**Controller → Switch:**

| Message | Purpose |
|---------|---------|
| `OFPT_FLOW_MOD` | Add/modify/delete flow entries |
| `OFPT_GROUP_MOD` | Manage group table entries |
| `OFPT_PACKET_OUT` | Inject packet into switch pipeline |
| `OFPT_BARRIER_REQUEST` | Synchronization barrier |
| `OFPT_STATS_REQUEST` | Poll flow/port/table statistics |
| `OFPT_SET_CONFIG` | Configure miss_send_len, flags |

**Switch → Controller:**

| Message | Purpose |
|---------|---------|
| `OFPT_PACKET_IN` | Unmatched packet; reason: NO_MATCH or ACTION |
| `OFPT_FLOW_REMOVED` | Flow entry expired (hard/idle timeout) |
| `OFPT_PORT_STATUS` | Link up/down event |
| `OFPT_ERROR` | Malformed request, table full, etc. |
| `OFPT_STATS_REPLY` | Response to stats request |

---

## 1.3 SDN Controllers: Ryu, ONOS, OpenDaylight

### Ryu (your current controller)

**Architecture:**
```
ryu-manager app1.py app2.py [app3.py ...]
         │
         ├── EventBase (event loop, green threads via eventlet)
         ├── OpenFlow Protocol Parser (ofproto_v1_3_parser)
         ├── Controller Hub (manages switch connections)
         └── App Manager (loads, wires apps via observe_event/send_event)
```

**Core event handlers:**

```python
from ryu.base import app_manager
from ryu.controller import ofp_event
from ryu.controller.handler import CONFIG_DISPATCHER, MAIN_DISPATCHER, set_ev_cls
from ryu.ofproto import ofproto_v1_3

class QKDNController(app_manager.RyuApp):
    OFP_VERSIONS = [ofproto_v1_3.OFP_VERSION]

    @set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
    def switch_features_handler(self, ev):
        """Called once when switch connects — install table-miss rule."""
        datapath = ev.msg.datapath
        ofproto  = datapath.ofproto
        parser   = datapath.ofproto_parser
        # Table-miss: send to controller
        match  = parser.OFPMatch()
        actions= [parser.OFPActionOutput(ofproto.OFPP_CONTROLLER,
                                         ofproto.OFPCML_NO_BUFFER)]
        self.add_flow(datapath, 0, match, actions)

    @set_ev_cls(ofp_event.EventOFPPacketIn, MAIN_DISPATCHER)
    def packet_in_handler(self, ev):
        """React to unknown packet."""
        ...

    @set_ev_cls(ofp_event.EventOFPPortStatus, MAIN_DISPATCHER)
    def port_status_handler(self, ev):
        """Fiber link event — trigger QKD path recomputation."""
        if ev.msg.reason == ev.msg.datapath.ofproto.OFPPR_DELETE:
            self.handle_link_down(ev.msg.datapath.id, ev.msg.desc.port_no)
```

**Ryu REST API** — your `sdn/controller.py :7000`:
```python
from ryu.app.wsgi import ControllerBase, WSGIApplication, route

class QKDNTopologyAPI(ControllerBase):
    @route('topology', '/topology/links', methods=['GET'])
    def get_links(self, req, **kwargs):
        # returns JSON of all known links
        ...

    @route('qkdn', '/qkdn/route/{src}/{dst}', methods=['GET'])
    def compute_path(self, req, src, dst, **kwargs):
        # Dijkstra/k-shortest paths weighted by QBER and key_rate
        ...
```

### ONOS (Open Network Operating System) — Production Scale

ONOS is the right controller for India-scale deployment. Key differences vs Ryu:

| Feature | Ryu | ONOS |
|---------|-----|------|
| Language | Python | Java |
| HA | Single process (no HA) | Clustered (RAFT consensus) |
| Scale | Lab / small networks | Carrier-scale (Tier-1 telcos) |
| Southbound | OF only | OF + NETCONF + gRPC + P4 |
| Intent API | Manual flow programming | High-level Intent Framework |
| GUI | None built-in | Full topology GUI |
| Real deployments | Research | AT&T, China Unicom, SK Telecom |

**ONOS Intent Framework for QKD:**
```java
// Express "I want quantum-secured connectivity between node A and node B"
// ONOS computes path, reserves resources, installs flows
PointToPointIntent intent = PointToPointIntent.builder()
    .appId(appId)
    .key(Key.of("qkd-alice-bob", appId))
    .filteredIngressPoint(new FilteredConnectPoint(
        new ConnectPoint(DeviceId.deviceId("of:alice"), PortNumber.portNumber(1))))
    .filteredEgressPoint(new FilteredConnectPoint(
        new ConnectPoint(DeviceId.deviceId("of:bob"), PortNumber.portNumber(1))))
    .constraints(ImmutableList.of(
        new BandwidthConstraint(Bandwidth.bps(1000000)),  // 1 Mbps for classical
        new AnnotationConstraint("qber", 0.08)            // max 8% QBER
    ))
    .build();
intentService.submit(intent);
```

### OpenDaylight (ODL)

ODL is ONOS's main competitor in carrier space. Uses OSGi bundles (Karaf container). The **MD-SAL** (Model-Driven SAL) is its core: everything is modeled in YANG, REST and gRPC auto-generated. Heavier than ONOS, but dominant in NFV integration (integrates with OpenStack Neutron).

---

## 1.4 Data Plane Abstractions

### Open vSwitch (OVS) — Your Mininet Backend

OVS is a production-grade software switch used in every cloud datacenter. Architecture:

```
Userspace:
  ovs-vswitchd ──── ovsdb-server (configuration DB)
       │
       │ netlink
       │
Kernel:
  openvswitch.ko (fast path — in-kernel datapath)
       │
  NIC drivers / veth pairs
```

**OVS in Mininet context:**
```bash
# See all switches
sudo ovs-vsctl show

# Dump flows on switch s1
sudo ovs-ofctl dump-flows s1 -O OpenFlow13

# Add a flow manually
sudo ovs-ofctl add-flow s1 \
  "priority=100,ip,nw_dst=10.0.1.2 actions=output:3" -O OpenFlow13

# Monitor packet counts
sudo ovs-ofctl dump-ports s1 -O OpenFlow13
```

### P4 (Programming Protocol-Independent Packet Processors)

P4 is the evolution beyond OpenFlow — you define the packet parser and match-action tables in code compiled to the switch ASIC. Relevant for future QKDN hardware:

```p4
// Define a custom "QKD header" to carry key_ID in the data plane
header qkd_t {
    bit<128> key_id;
    bit<8>   qber_estimate;
}

// Table: route based on key_id
table qkd_routing {
    key = {
        hdr.qkd.key_id : ternary;
    }
    actions = {
        forward_to_kms;
        drop;
    }
    size = 1024;
}
```

---

## 1.5 SDN Security & Fault Tolerance

### SDN-Specific Attack Surface

| Attack | Description | Mitigation |
|--------|-------------|-----------|
| Controller compromise | Single controller = single point of failure | Clustered controller (ONOS RAFT) |
| Southbound spoofing | Fake switch connects to controller | TLS mutual auth on OF channel |
| Northbound API abuse | Malicious app installs flows | App authentication, policy engine |
| Flow rule flooding | Fill TCAM table → DoS | Rate-limit Packet-In; proactive flow installation |
| Topology poisoning | Inject false LLDP → wrong topology | Cryptographic LLDP (SECUREDGE) |

**For QKD specifically:** The SDN control channel must itself be secured. Irony: the system distributing quantum keys needs a secure classical channel to operate. Standard practice: bootstrap with PQC (Kyber) until first QKD keys are available, then switch to QKD-derived session keys for the control channel.

### Controller High Availability

**RAFT consensus (used by ONOS, etcd, CockroachDB):**

- 2F+1 nodes tolerate F failures (3-node cluster tolerates 1 failure, 5-node tolerates 2)
- Leader elected; all writes go through leader, replicated to majority before commit
- Leader failure → new election in ~150–300ms
- For QKDN: 3-node ONOS cluster minimum; 5-node for metro-scale

---

## 1.6 Mininet Lab Environment

### What Mininet Actually Does

Mininet creates a **virtual network** entirely in Linux using:
- **Network namespaces** — each host/switch gets isolated network stack
- **veth pairs** — virtual Ethernet cables connecting namespaces
- **OVS** — software switches with full OF pipeline
- **cgroups** — CPU/memory limits per node

```bash
# Install
sudo apt update && sudo apt install mininet openvswitch-switch

# Verify OVS
sudo systemctl start openvswitch-switch
sudo ovs-vsctl show

# Simple test
sudo mn --topo single,4 --controller remote,ip=127.0.0.1,port=6633
```

### QKDN Topology for Your Architecture

```python
#!/usr/bin/env python3
# qkdn_topo.py — star-mesh topology matching your QKDN SaaS v2
from mininet.net import Mininet
from mininet.node import RemoteController, OVSSwitch
from mininet.topo import Topo
from mininet.log import setLogLevel
from mininet.cli import CLI
from mininet.link import TCLink

class QKDNTopo(Topo):
    """
    Star-mesh: 5 KMS nodes (alice, bob, charlie, diana, eve)
    connected via a core switch (simulates QKD fiber star)
    Each KMS node also has a 'quantum' interface (separate subnet)
    for simulated photon exchange.
    """
    def build(self, n_nodes=5, bw_classical=1000, bw_quantum=0.1, delay='5ms'):
        nodes = ['alice', 'bob', 'charlie', 'diana', 'eve'][:n_nodes]
        
        # Core SDN switch (star center)
        core = self.addSwitch('s1', cls=OVSSwitch, protocols='OpenFlow13')
        
        # QKD engine node (central BB84 simulator)
        qkd_engine = self.addHost('qkdengine', ip='10.0.100.1/24')
        self.addLink(core, qkd_engine,
                     cls=TCLink, bw=bw_classical, delay=delay)
        
        # KMS nodes + quantum channel simulation
        for i, name in enumerate(nodes):
            ip_classical = f'10.0.0.{i+1}/24'
            ip_quantum   = f'10.1.0.{i+1}/24'
            
            host = self.addHost(name, ip=ip_classical)
            
            # Classical link (high BW — key material + API traffic)
            self.addLink(core, host,
                        cls=TCLink, bw=bw_classical, delay=delay)
            
            # Quantum channel (very low BW, lossy — simulates fiber QKD)
            # In reality: separate fiber carrying attenuated photon pulses
            # In sim: low-BW link with loss representing channel loss
            q_switch = self.addSwitch(f'qs{i+1}',
                                       cls=OVSSwitch, protocols='OpenFlow13')
            self.addLink(core, q_switch,
                        cls=TCLink, bw=bw_quantum, delay=delay, loss=5)
            self.addLink(q_switch, host,
                        cls=TCLink, bw=bw_quantum, delay=delay)
        
        # Mesh inter-KMS links (bypass star for direct node peering)
        for i in range(len(nodes)-1):
            h1 = self.getNodeByName(nodes[i])
            h2 = self.getNodeByName(nodes[i+1])
            # These represent direct fiber links between adjacent sites
            self.addLink(h1, h2, cls=TCLink, bw=10, delay='2ms')
        
        return

def run():
    setLogLevel('info')
    topo  = QKDNTopo()
    net   = Mininet(topo=topo,
                    controller=RemoteController('c0', ip='127.0.0.1', port=6633),
                    switch=OVSSwitch,
                    link=TCLink,
                    autoSetMacs=True)
    net.start()
    
    # Verify connectivity
    net.pingAll()
    
    # Start your QKDN services on the hosts
    alice = net.getNodeByName('alice')
    alice.cmd('cd /path/to/qkdn_saas && python3 kms/kms_sqlite.py &')
    
    CLI(net)
    net.stop()

if __name__ == '__main__':
    run()
```

### Mininet Link Parameters for QKD Simulation

| Parameter | Classical Channel | Quantum Channel Sim |
|-----------|-------------------|---------------------|
| `bw` | 1000 Mbps | 0.001–1 Mbps |
| `delay` | 1–50 ms | 5–100 ms (propagation) |
| `loss` | 0.01% | 1–20% (fiber loss model) |
| `jitter` | < 1ms | 5–10 ms |

**Loss model for fiber:**  
Real fiber: ~0.2 dB/km at 1550nm (C-band). Over 100km: 20 dB loss.  
Each 3 dB = 50% photon loss. Mininet's `loss` parameter is percentage packet drop.  
To simulate 50km C-band fiber: `loss=10` (10% packet drop ≈ 10dB ≈ 50km).

---

# Part II — Quantum Key Distribution

## 2.1 Quantum Information Foundations

### Qubits and Hilbert Space

A qubit lives in a 2-dimensional complex Hilbert space **H** = ℂ²:

```
|ψ⟩ = α|0⟩ + β|1⟩       where  α, β ∈ ℂ,  |α|² + |β|² = 1
```

The computational basis: `|0⟩ = [1,0]ᵀ`, `|1⟩ = [0,1]ᵀ`

**Bloch sphere representation:**
```
|ψ⟩ = cos(θ/2)|0⟩ + e^{iφ}sin(θ/2)|1⟩
```

Key quantum properties exploited by QKD:

1. **No-Cloning Theorem** (Wootters & Zurek, 1982):  
   *There exists no unitary operation U such that U|ψ⟩|0⟩ = |ψ⟩|ψ⟩ for arbitrary |ψ⟩.*  
   → An eavesdropper cannot copy quantum states without detection.

2. **Measurement disturbs state** (Heisenberg):  
   Measuring in the wrong basis collapses the state, introducing detectable errors.

3. **Quantum indistinguishability**:  
   Non-orthogonal quantum states cannot be perfectly distinguished.

### Polarization Encoding (BB84's Physical Layer)

BB84 uses **photon polarization** as the quantum degree of freedom:

| Basis | Symbol | States | Physical |
|-------|--------|--------|----------|
| Rectilinear `+` | `⊕` | `|→⟩` = 0, `|↑⟩` = 1 | 0°, 90° polarization |
| Diagonal `×` | `⊗` | `|↗⟩` = 0, `|↘⟩` = 1 | 45°, 135° polarization |

Relationship between bases:
```
|↗⟩ = (|→⟩ + |↑⟩)/√2     [Hadamard gate on |0⟩]
|↘⟩ = (|→⟩ - |↑⟩)/√2     [Hadamard gate on |1⟩]
```

**Key insight**: If Bob measures in the same basis Alice used → perfect correlation.  
If Bob measures in the wrong basis → completely random result (50% error).

---

## 2.2 BB84 Protocol — Full Technical Walkthrough

### Phase 1: Quantum Transmission

Alice randomly selects:
- Bit value: {0, 1}
- Encoding basis: {+, ×}

She transmits photons prepared in the corresponding state.

```
Alice's random bits:    1  0  1  1  0  0  1  0  1  1
Alice's random bases:   +  +  ×  +  ×  +  +  ×  ×  +
Alice's states:         ↑  →  ↘  ↑  ↗  →  ↑  ↗  ↘  ↑

Bob's random bases:     +  ×  ×  +  +  +  ×  ×  +  +
Bob's measurements:     1  ?  1  1  ?  0  ?  0  ?  1
                         (? = random — wrong basis)
```

### Phase 2: Sifting

Over classical channel (public, authenticated):
1. Bob announces which bases he used (NOT his measurement results).
2. Alice announces where their bases matched.
3. Both discard bits where bases differed.

Sifted key (≈ 50% of transmitted bits):
```
Positions:    1    3    4    6    8    10
Sifted bits:  1    1    1    0    0    1
```

**Sifting efficiency:** Standard BB84: 50% key rate.  
SARG04 and efficient BB84 variants can approach higher efficiency.

### Phase 3: Error Estimation

Alice and Bob publicly reveal a **random subset** (typically 10–20%) of sifted bits and compute the **Quantum Bit Error Rate (QBER)**:

```
QBER = (number of disagreements) / (number of test bits revealed)
```

**Security threshold**: If QBER > 11% (for single-photon BB84), eavesdropping makes security unachievable and protocol aborts.

The 11% threshold comes from the security proof: Eve's optimal intercept-resend attack introduces QBER = 25%. The threshold 11% (more precisely Q_max ≈ 11.0%) ensures mutual information between sifted key and Eve is bounded.

### Phase 4: Information Reconciliation (Error Correction)

The sifted key has errors (both from eavesdropping AND channel noise). Alice and Bob must obtain identical keys without revealing too much information.

**Cascade Protocol** (Brassard & Salvail, 1994):
- Divide key into blocks; for each block compute parity
- Binary search for error positions
- Iterates multiple passes with block size doubling
- Reveals O(1.16h(e)) bits where h(e) is binary entropy

**LDPC-based reconciliation** (modern approach):
- Alice sends parity-check syndrome to Bob
- Bob runs belief propagation decoding
- Advantage: near-Shannon-limit efficiency, high throughput
- Used in all modern QKD hardware

Information leaked during reconciliation: ≈ 1.16 × h(QBER) bits per sifted bit, where h(p) = -p log₂(p) - (1-p) log₂(1-p) is binary entropy.

### Phase 5: Privacy Amplification

After reconciliation, Eve may know some bits. Privacy amplification extracts a shorter, truly secret key using **universal hash functions**:

```
Final key = Hash_{r}(reconciled key)
```

Where r is a randomly chosen hash function from a two-universal family, and the final key length is:

```
ℓ = n_sifted × [1 - h(QBER) - f × h(QBER)] - leak_reconcile - log₂(1/ε)
```

Where:
- `f` = reconciliation efficiency (≥ 1; Shannon limit = 1)
- `ε` = security parameter (typically 10⁻¹⁰)
- `leak_reconcile` = bits revealed in error correction phase

**Your implementation**: `engine.py` uses SHA-256 on sifted bits as a simplified privacy amplification step. Production should use HKDF (RFC 5869) or a proper extractor.

### Phase 6: Authentication

The classical channel must be authenticated (else man-in-the-middle). Standard approach:

1. **Initial authentication**: Pre-shared authentication key (small, one-time)
2. **Ongoing**: Use a fraction of the QKD-generated key to authenticate future QKD sessions → **quantum key growing**

Authentication: Wegman-Carter MACs using pairwise independent hash functions, which are information-theoretically secure (unconditional).

---

## 2.3 Security Proofs & QBER Analysis

### Composable Security (Modern Framework)

Modern QKD security is proven in the **Universal Composability (UC)** framework (Canetti, 2001), extended for QKD by Renner (2005). A protocol is ε-secure if:

```
½ × ||ρ_KE - U_K ⊗ ρ_E||₁ ≤ ε
```

Where `ρ_KE` is the joint state of the final key K and Eve's quantum system E, and `U_K` is the uniform distribution. This means the key is ε-close to uniform and independent of Eve.

### Security Proof Strategies

| Approach | Key Idea | Security Level |
|----------|----------|---------------|
| Lo-Chau (1999) | Entanglement distillation reduction | Unconditional (information-theoretic) |
| Shor-Preskill (2000) | CSS codes reduction to E91 | Unconditional, practical |
| Renner (2005) | Smooth min-entropy framework | Composably secure, finite-key |
| Twin-field security | Phase-randomized coherent states | Extends reach to ~500 km |

### QBER Sources and Budget

```
Total QBER = QBER_channel + QBER_detector + QBER_alignment + QBER_Eve

QBER_channel:   0.5–3%    (dark counts, Raman noise from WDM coexistence)
QBER_detector:  0.1–0.5%  (detector dark count rate × gate width)
QBER_alignment: 0.1–1%    (polarization drift, basis misalignment)
QBER_Eve:       up to 25% (intercept-resend attack)
```

Typical deployed systems operate at QBER ≈ 2–5%, giving comfortable margin below 11% threshold.

---

## 2.4 Trusted-Node vs Trusted-Node-Free Architectures

### Trusted-Node Relay Networks

```
Alice ──[QKD]──► Node A ──[QKD]──► Node B ──[QKD]──► Bob
```

- Each hop generates a fresh QKD key
- Node A knows both Alice's key and Node B's key
- End-to-end key: K_final = K_Alice_A ⊕ K_A_B ⊕ K_B_Bob
- **Security assumption**: All intermediate nodes are trusted
- **Used in**: China's 2000km Beijing-Shanghai backbone, Tokyo QKD network

**Risk**: If any intermediate node is compromised, end-to-end security is broken. Classical security only.

### Trusted-Node-Free Approaches

These aim for end-to-end quantum security without trusting relays:

| Method | Mechanism | Current TRL |
|--------|-----------|------------|
| **Quantum Repeaters** | Entanglement swapping + purification | TRL 2–4 (lab only) |
| **Twin-Field QKD** | Interference-based; relay measures but doesn't learn key | TRL 6–7; ~500km demo |
| **MDI-QKD** | Measurement-device-independent; relay does Bell measurement | TRL 5–6; ~200km |
| **Satellite relay** | Free-space QKD; satellite not trusted for key | TRL 7 (Micius) |

### Your Architecture's Claim

Your README states "Trusted-node-free: Each end-user app talks directly to its nearest KMS." This is a **software architecture choice** (no intermediate KMS in key forwarding path), NOT a quantum-physical trusted-node-free guarantee.

In your current simulation, all KMS nodes run on the same machine/network — the fiber links are simulated. When deployed on real fiber, adjacent KMS nodes DO form a trusted-node topology unless you implement MDI-QKD or quantum repeaters.

**This is not a flaw** — it's the industry-standard approach for near-term deployment. The Beijing-Shanghai backbone uses trusted nodes. The key is: label it accurately as "trusted-node relay network with short hop distances" or implement MDI-QKD between KMS pairs.

---

## 2.5 Advanced QKD Protocols

### E91 (Ekert, 1991) — Entanglement-Based

Uses EPR pairs (Bell states). Alice and Bob each receive one qubit of an entangled pair from a source.

```
Source produces: |Φ⁺⟩ = (|00⟩ + |11⟩)/√2
Alice measures her qubit: gets 0 or 1
Bob measures his qubit: perfectly correlated (if same basis)
```

Security based on Bell inequality violation (CHSH test). If Eve intercepts, entanglement is disturbed → Bell inequality satisfied → detected.

**CHSH inequality**: S = |E(a,b) - E(a,b') + E(a',b) + E(a',b')| ≤ 2 (classical)  
Quantum max: S_quantum = 2√2 ≈ 2.828

### MDI-QKD (Lo, Curty & Qi, 2012)

Both Alice and Bob send pulses to an untrusted relay (Charlie) who performs a Bell state measurement (BSM) and announces results. The relay learns nothing about the key bits.

```
Alice ──photons──► RELAY (Charlie, untrusted) ◄──photons── Bob
                         │
                   Bell measurement
                   announces BSM result
                   (does NOT reveal key)
```

**Advantage**: Immune to detector side-channel attacks (dominant attack vector in real QKD hardware).  
**Disadvantage**: Lower key rate (O(η²) instead of O(η) for prepare-measure).

### Twin-Field QKD (TF-QKD) (Lucamarini et al., 2018)

Key insight: key rate scales as O(√η) instead of O(η) for standard QKD, matching quantum repeater rates.

```
Alice ──────────────────────────────────────────── Bob
         ↓ weak coherent pulses (WCP)    ↓
         ────────────► [Charlie] ◄────────────
                     interference + measurement
```

Alice and Bob encode key bits in the phase of WCP. Charlie measures single photons via interference. Achieves practical QKD at ~500 km (demonstrated by Toshiba, 2021).

**This is the protocol to watch for India's long-haul inter-city QKD.**

### CV-QKD (Continuous-Variable QKD)

Instead of discrete polarization, uses **quadrature amplitudes** of coherent laser pulses (homodyne/heterodyne detection).

```
Alice prepares: (x_A, p_A) ~ N(0, V_A) [Gaussian modulation]
Transmits over fiber
Bob measures: one or both quadratures
```

**Advantages**:
- Uses standard telecom components (no single-photon detectors)
- Can run on standard DWDM C-band infrastructure
- Commercially available: ID Quantique Clavis300, Toshiba QKD

**Disadvantages**:
- Lower distance (≤100km without repeaters in practice)
- More complex security proof (Gaussian optimality conjecture, now proven)
- Sensitive to excess noise from WDM channel crosstalk

**For C-band home/office LAN testing: CV-QKD is the most practical approach.**

### DPS-QKD (Inoue et al., 2002)

Differential Phase Shift QKD uses train of coherent pulses with random phase shifts (0 or π). Bob measures interference between adjacent pulses. Simple hardware, but security proof more complex.

---

## 2.6 Hardware Layer: Sources, Detectors, and Fiber

### Single-Photon Sources

| Type | Principle | g²(0) | Key Rate | Status |
|------|-----------|-------|----------|--------|
| Attenuated laser (WCP) | ND filter on CW laser, μ < 0.1 photons/pulse | > 0 | High | Commercial |
| SPDC (spontaneous parametric down-conversion) | Nonlinear crystal (BBO, KTP) | ~0.01 | Medium | Lab/semi-commercial |
| Quantum dots | Semiconductor emitter | ~0.01 | High potential | Research |
| NV centers | Color center in diamond | ~0.001 | Low | Research |

**WCP (Weak Coherent Pulses)** — standard in deployed systems:
- Mean photon number μ ≈ 0.1–0.5 per pulse
- Poisson distribution: P(n) = e^{-μ} × μⁿ / n!
- Multi-photon pulses (n≥2) enable PNS (photon number splitting) attack
- Mitigated by **decoy states**: mix signal (μ_s), weak decoy (μ_w), vacuum decoy

### Single-Photon Detectors

| Type | Efficiency | Dark Count | Jitter | Operating Temp | Use Case |
|------|------------|------------|--------|----------------|----------|
| InGaAs/InP APD | 10–25% | 10⁻⁵ /gate | ~100ps | -50°C (TE cooled) | Telecom, deployed |
| Si APD | 65–75% | 10⁻⁶ | ~350ps | Room temp | 400–900nm only |
| SNSPD | 85–98% | 10⁻³ /s (free-run) | ~30ps | 2.5K (cryo) | Research, ultrafast |
| SSPD (SNSPD) | 90%+ | Very low | <10ps | 4K | State of art |

**SNSPD (Superconducting Nanowire SPD)**: state of the art. Used in Toshiba's 605km TF-QKD record. Requires liquid helium cooling — not practical for field deployment yet, but 4K cryostats are emerging.

### Fiber Properties for QKD

| Parameter | Value | Impact on QKD |
|-----------|-------|--------------|
| Attenuation (1550nm C-band) | ~0.2 dB/km | Limits distance; 100km = 20dB = 99% photon loss |
| Attenuation (1310nm O-band) | ~0.35 dB/km | Worse for long-haul |
| Chromatic Dispersion (C-band) | 17 ps/nm·km | Pulse broadening → timing jitter |
| PMD (Polarization Mode Dispersion) | 0.01–0.1 ps/√km | Polarization rotation → BB84 basis errors |
| Raman Scattering (from classical WDM) | Forward/backward | Noise photons in quantum window → QBER increase |
| Brillouin Scattering | -20dB relative | Coherent noise source for CV-QKD |

**PMD mitigation for BB84**:
- Active polarization control (motorized wave plates) at receiver
- Reference pulse tracking (polarization-encoded reference pulse)
- Phase encoding instead of polarization (insensitive to PMD)

---

## 2.7 ETSI QKD Standards

### ETSI GS QKD 014 (Your API Standard)

The REST API standard for interfacing QKD Key Management Systems with applications.

**Key Interface** (KI) defined endpoints:
```
GET  /api/v1/keys/{slave_SAE_ID}/enc_keys
     ?number=N&size=256
     → { "keys": [{"key_ID": "uuid", "key": "base64_or_hex"}] }

POST /api/v1/keys/{master_SAE_ID}/dec_keys
     body: {"key_IDs": [{"key_ID": "uuid"}]}
     → { "keys": [{"key_ID": "uuid", "key": "base64_or_hex"}] }

GET  /api/v1/keys/{slave_SAE_ID}/status
     → { "source_KME_ID": ..., "target_KME_ID": ...,
         "master_SAE_ID": ..., "slave_SAE_ID": ...,
         "key_size": 256, "stored_key_count": 98,
         "max_key_count": 100, "max_key_size": 1024,
         "max_SAE_ID_count": 0 }
```

**Terminology:**
- **SAE (Secure Application Entity)**: Application consuming keys
- **KME (Key Management Entity)**: Your KMS service
- `enc_keys` endpoint used by master (key requester/encryptor)
- `dec_keys` endpoint used by slave (key getter/decryptor)
- Key identified by UUID, same key retrieved by both sides

### ETSI GS QKD 015 (Control Interface)

Defines how the SDN controller/orchestrator communicates with QKD devices to:
- Request establishment of quantum links
- Query link status (key rate, QBER, distance)
- Configure QKD module parameters

```json
POST /qkd/v1/links
{
  "source": "kme-delhi",
  "destination": "kme-mumbai",
  "requested_key_rate": 1000,
  "qos": {"priority": 5, "max_qber": 0.08}
}
```

### ETSI GS QKD 018 (Key Relay)

Defines the "trusted node" relay protocol — how KMS nodes forward keys across multi-hop paths. Your current codebase doesn't implement this (you have direct point-to-point links), but it will be needed for India-scale.

### ETSI GS QKD 004 / 005 (Security Guidelines)

- 004: Security requirements for QKD modules
- 005: Security proofs for specific protocols
- Key takeaway: QKD hardware must be certified; key material must never leave a tamper-evident module in plaintext

---

# Part III — SDN-QKD Integration

## 3.1 Why SDN for QKD Networks

QKD networks without SDN have fundamental operational problems:

**Problem 1: Static routing wastes keys.**  
QKD key rates vary with fiber quality. A link with degraded QBER generates fewer keys. Static routes may overload low-rate links. SDN enables dynamic re-routing to balance key consumption.

**Problem 2: Multi-path key management is complex.**  
If a packet stream uses multiple paths (ECMP), keys from multiple QKD links must be coordinated. SDN's centralized view enables coherent multi-path QKD key scheduling.

**Problem 3: Service provisioning time.**  
Manually configuring QKD links between new nodes takes hours. SDN + ETSI QKD 015 enables automated link provisioning in seconds.

**Problem 4: Fault recovery.**  
Fiber cut → QKD link down → key starvation for all services on that link. SDN detects link failure via OF PortStatus, triggers path re-routing, and re-establishes QKD on the new path — automatic, sub-second.

## 3.2 Control-Plane Design for QKDN

### The Dual-Plane Architecture

```
┌─────────────────────────────────────────────────────┐
│  QUANTUM CONTROL PLANE                              │
│  SDN Controller (Ryu/ONOS)                         │
│  ├── Topology manager (knows all QKD links + QBER) │
│  ├── Key rate monitor (polls KMS /status endpoints) │
│  ├── Path computation (Dijkstra weighted by QBER)  │
│  └── ETSI QKD 015 interface                        │
├─────────────────────────────────────────────────────┤
│  CLASSICAL CONTROL PLANE (standard SDN)             │
│  ├── L2 learning switch                             │
│  ├── L3 routing                                     │
│  └── QoS / metering                                │
└─────────────────────────────────────────────────────┘
```

### Path Computation for QKD

Standard Dijkstra uses hop-count or bandwidth as weight. For QKD, the weight function should be:

```
w(link) = (1 / key_rate_remaining) × QBER_penalty(qber) × distance_penalty(km)

where:
  key_rate_remaining = current_key_pool / max_key_pool
  QBER_penalty(q)    = 1 / (1 - h(q)/h(q_max))   [higher QBER = higher cost]
  distance_penalty   = exp(0.2 × km / 100)         [exponential with distance]
```

**Implementation in your Ryu controller:**
```python
import networkx as nx

class QKDNPathComputer:
    def __init__(self):
        self.graph = nx.Graph()
    
    def update_link_weight(self, src, dst, key_rate, qber, distance_km):
        """Call this every time KMS /status is polled."""
        import math
        # Binary entropy
        h = lambda p: -p*math.log2(p) - (1-p)*math.log2(1-p) if 0<p<1 else 0
        
        key_rate_weight = 1.0 / (key_rate + 1e-9)  # avoid div by zero
        qber_weight = 1.0 / (1.0 - h(qber) / h(0.11) + 1e-6)
        dist_weight = math.exp(0.02 * distance_km)
        
        weight = key_rate_weight * qber_weight * dist_weight
        self.graph[src][dst]['weight'] = weight
        self.graph[src][dst]['qber']   = qber
        self.graph[src][dst]['key_rate'] = key_rate
    
    def get_qkd_path(self, src, dst, k=3):
        """Return k shortest QKD-weighted paths."""
        return list(nx.shortest_simple_paths(self.graph, src, dst, weight='weight'))[:k]
```

## 3.3 Key Management Service Architecture

### Your KMS Design (`kms_service.py`)

Your KMS implements ETSI QKD 014 on top of:
- **PostgreSQL**: persistent key storage (survives restarts)
- **Redis**: ephemeral key pool cache (fast in-memory access)
- **QKD Engine**: key generation via BB84 simulator on `:8100`

**Key lifecycle:**
```
QKD Engine generates key pair → stored in PostgreSQL → 
cached in Redis pool → consumed via /enc_keys → 
matching /dec_keys retrieves same key → deleted after use
```

**Pool management (your `admin/refill` endpoint):**
```python
async def refill_pool(self, peer_node: str, target_count: int = 100):
    """
    Request new keys from QKD engine.
    In hardware mode: this triggers physical BB84 exchange.
    In simulator mode: calls engine.py /generate_key.
    """
    current_count = await self.get_pool_size(peer_node)
    needed = target_count - current_count
    if needed <= 0:
        return
    
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"http://qkd_engine:8100/generate_keys",
            json={"count": needed, "key_size": 256,
                  "alice": self.node_name, "bob": peer_node}
        )
    keys = resp.json()["keys"]
    await self.store_keys(peer_node, keys)  # → PostgreSQL + Redis
```

### Key Synchronization Between KMS Nodes

Critical issue: Both Alice-KMS and Bob-KMS must have the **same key** for the same `key_ID`. How does this work?

**Option A (your current approach — centralized engine):**
```
QKD Engine generates (key_A, key_B) where key_A == key_B
Distributes key_A to alice-KMS and key_B to bob-KMS
Trust model: QKD Engine must be trusted
```

**Option B (proper QKD — decentralized):**
```
Alice sends photons → Bob measures
Sifting over classical channel
Error correction produces identical keys at both ends
No central party ever has the key
```

For simulation, Option A is fine. For production security claims, Option B is required.

---

## 3.4 Multi-Tenant QKD as a Service

### Tenant Isolation Requirements

| Layer | Mechanism | Your Implementation |
|-------|-----------|---------------------|
| Network | VLAN / VxLAN per tenant | Not yet implemented |
| API | JWT with tenant claim | ✓ `POST /auth/token?tenant_id=demo` |
| Key pool | Per-tenant key namespace in DB | Partially — single global pool |
| Audit | Per-tenant audit log | ✓ `/api/v1/audit` |
| Rate limiting | Per-tenant API rate limit | ✓ nginx rate-limit config |

**Enhancing tenant isolation in your PostgreSQL schema:**
```sql
-- Add tenant isolation to key pool
ALTER TABLE key_pool ADD COLUMN tenant_id VARCHAR(64) NOT NULL DEFAULT 'default';
CREATE INDEX idx_key_pool_tenant ON key_pool(tenant_id, peer_node, consumed);

-- Policy: KMS only returns keys for requesting tenant
CREATE POLICY tenant_isolation ON key_pool
  USING (tenant_id = current_setting('app.current_tenant'));
```

### QoS for Different Tenants

```python
# Key allocation policy: premium tenants get priority access
class KeyAllocationPolicy:
    TIER_LIMITS = {
        'premium': {'keys_per_minute': 1000, 'min_key_rate': 100},
        'standard': {'keys_per_minute': 100, 'min_key_rate': 10},
        'free': {'keys_per_minute': 10, 'min_key_rate': 1},
    }
    
    async def allocate(self, tenant_id: str, peer: str, n_keys: int):
        tier = await self.get_tier(tenant_id)
        limit = self.TIER_LIMITS[tier]
        # Check rate limit, pool level, priority queue
        ...
```

---

## 3.5 Observability Stack

### Your Prometheus + Grafana Setup

**Metrics your KMS should expose (`/metrics`):**

```python
from prometheus_client import Counter, Gauge, Histogram, generate_latest

# Key consumption rate
keys_consumed_total = Counter('qkdn_keys_consumed_total',
    'Total keys consumed', ['node', 'peer', 'tenant'])

# Key pool depth
key_pool_depth = Gauge('qkdn_key_pool_depth',
    'Number of keys in pool', ['node', 'peer'])

# QBER gauge
qber_current = Gauge('qkdn_qber_current',
    'Current quantum bit error rate', ['link'])

# Key generation latency
key_gen_latency = Histogram('qkdn_key_gen_seconds',
    'Key generation latency', ['node'],
    buckets=[0.001, 0.01, 0.1, 0.5, 1.0, 5.0])

# API request latency (ETSI 014)
api_latency = Histogram('qkdn_api_latency_seconds',
    'API endpoint latency', ['endpoint', 'method'],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5])
```

**Critical Grafana dashboards to build:**
1. **Key Rate Dashboard**: keys/sec per link, pool depth heatmap, refill events
2. **QBER Timeline**: per-link QBER trend, threshold alerts at 8% (warning) and 11% (critical)
3. **SDN Topology**: live topology with link states and key rates as edge labels
4. **Multi-Tenant Usage**: per-tenant key consumption, quota utilization
5. **Security Events**: auth failures, QBER threshold crossings, key exhaustion events

---

# Part IV — Your QKDN SaaS v2 Architecture Analysis

## 4.1 Component-by-Component Review

### Gateway (nginx, `:443/:80/:8080`)

**Current**: Reverse proxy with rate limiting and self-signed TLS cert.

**Strengths**: 
- mTLS capability commented in nginx.conf (good practice — enable for production)
- Rate limiting prevents key pool exhaustion attacks

**Gaps**:
- Self-signed cert → should use Let's Encrypt or internal CA
- No API versioning beyond `/api/v1/` (plan `/api/v2/` for ETSI QKD 015 additions)
- Missing: request signing verification (HMAC on payload for non-repudiation)

### QKD Engine (`:8100`)

**Current**: BB84 simulator with hardware abstraction stubs.

**Strengths**:
- Clean HW/SW interface (sim vs serial vs REST hardware modes)
- Correct QBER threshold at 0.11

**Gaps**:
- Privacy amplification uses SHA-256 directly on sifted bits. **This is not secure in general** — SHA-256 is a one-way function, not a randomness extractor in the information-theoretic sense. Should use HKDF-SHA-256 or explicit Toeplitz matrix hashing.
- No decoy state simulation (important for WCP-based BB84 security)
- No finite-key correction: the key length formula doesn't account for statistical fluctuations in small key block sizes. For n < 10^6 bits, finite-key effects reduce secure key length significantly.
- Error correction simulated as "just XOR errors randomly" — should implement Cascade or LDPC to accurately model information leakage.

### KMS Service (`:5001–:5005`)

**Current**: ETSI QKD 014 REST API, PostgreSQL + Redis storage.

**Strengths**: 
- Correct ETSI 014 endpoint naming and response format
- Redis pool for fast key delivery
- PostgreSQL for persistence

**Gaps**:
- Keys stored as plaintext hex in PostgreSQL → should be encrypted at rest (AES-256-GCM with HSM-derived key)
- No key expiry policy: stale keys in pool are a security risk. ETSI recommends max key lifetime (e.g., 24h)
- The `dec_keys` endpoint should atomically delete the key after retrieval (prevent double-use). Verify this with `SELECT ... FOR UPDATE` + `DELETE` in a transaction.
- Missing: key confirmation flow (Bob confirms key received before Alice's copy deleted)

### SDN Controller (`:7000/:6633`)

**Current**: Ryu OF1.3 with REST topology API.

**Strengths**: 
- Correct choice of OF1.3 for its pipeline and group table support
- Dual port (6633 for OF, 7000 for REST northbound)

**Gaps**:
- No QBER-weighted path computation (routes are currently shortest-hop, not QKD-quality-aware)
- No link failure handling that triggers KMS renegotiation
- No LLDP authentication (topology poisoning risk in multi-tenant context)
- Single controller instance → SPOF. For any production use: move to ONOS cluster.

### Dashboard (`:9000`)

**Current**: WebSocket-based live ops view.

**Strengths**: Real-time key pool visualization is genuinely useful for QKD operations.

**Gaps**:
- Authentication on dashboard WebSocket? Anyone who can reach :9000 sees operational data.
- No alerting integration (PagerDuty, Alertmanager webhook)

---

## 4.2 Gaps, Improvements & Research Opportunities

### Immediate Technical Improvements (V2 → V2.1)

1. **HKDF for privacy amplification** (security correctness):
```python
import hashlib, hmac
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives import hashes

def privacy_amplify(sifted_key_bytes: bytes, output_len_bytes: int,
                     salt: bytes = None, info: bytes = b"qkdn_v2") -> bytes:
    hkdf = HKDF(algorithm=hashes.SHA256(), length=output_len_bytes,
                salt=salt, info=info)
    return hkdf.derive(sifted_key_bytes)
```

2. **Atomic key consumption** (double-use prevention):
```python
async def consume_key(self, key_id: str, peer: str) -> Optional[str]:
    async with self.db.transaction():
        row = await self.db.fetchrow(
            "SELECT key_hex FROM key_pool WHERE key_id=$1 AND peer=$2 "
            "AND consumed=FALSE FOR UPDATE",
            key_id, peer
        )
        if not row:
            return None
        await self.db.execute(
            "UPDATE key_pool SET consumed=TRUE, consumed_at=NOW() "
            "WHERE key_id=$1", key_id
        )
        return row['key_hex']
```

3. **QBER-weighted routing in Ryu controller** (see Section 3.2 code).

4. **Key pool encryption at rest** using Fernet (AES-128-CBC + HMAC) or AES-256-GCM.

### Research Opportunities

1. **Adaptive key generation rate**: Use ML (LSTM or simple PID controller) to predict key consumption rates per tenant and proactively trigger QKD engine refills, minimizing key starvation events.

2. **SDN-assisted QBER monitoring**: Use SDN telemetry (INT — In-band Network Telemetry) to correlate classical channel quality with quantum channel QBER. Sudden classical link degradation predicts quantum QBER increase.

3. **Multi-path QKD**: Instead of single path, XOR keys from multiple independent QKD links: K_final = K_path1 ⊕ K_path2. Security increases (attacker must compromise all paths simultaneously). SDN coordinates multi-path selection.

4. **Zero-trust integration**: Combine QKD key delivery with a SPIFFE/SPIRE-based workload identity system. Each microservice gets a QKD-derived secret, renewed continuously.

---

# Part V — C-Band & Mininet Local Testing

## 5.1 C-Band Fiber Optics Primer

### ITU-T Wavelength Bands

| Band | Wavelength | Primary Use | QKD Relevance |
|------|-----------|-------------|---------------|
| O-band | 1260–1360 nm | 100G CWDM | Lower loss than S, but higher than C |
| E-band | 1360–1460 nm | Rarely used (OH absorption) | Poor for QKD |
| S-band | 1460–1530 nm | Some CWDM | Moderate |
| **C-band** | **1530–1565 nm** | **DWDM, 5G fronthaul, everything** | **Best for QKD: lowest loss** |
| L-band | 1565–1625 nm | Extended DWDM | Slightly higher loss, usable |
| U-band | 1625–1675 nm | Monitoring | Poor |

**Why C-band for QKD:**
- Minimum fiber attenuation: ~0.185 dB/km at 1550nm
- Full ITU-T DWDM grid available: 100 GHz spacing (0.8nm) = 40 channels, 50GHz = 80 channels
- Mature components: EDFAs, AWGs, WSSes

### C-Band LAN Cable for Local Testing

When testing over **standard SMF-28 patch cables** in a lab/LAN:
- Length typically 1–100m → negligible loss (~0.02–0.2dB total)
- PMD negligible at these lengths
- **The limiting factor becomes connector reflections, detector noise, and electronic jitter**

**Practical lab setup for C-band QKD testing:**
```
Alice PC ──[SMF-28 patch]──► Variable Attenuator ──[SMF-28]──► Bob PC
                                   ↑
                    Set to simulate 50km loss (10dB)
                    or 100km loss (20dB)
```

The variable optical attenuator (VOA) is critical: it lets you test different "distance" scenarios in a lab without actually having long fiber.

**C-band component cost guide (Lajpat Rai / online imports):**

| Component | Purpose | Approx Cost (INR) |
|-----------|---------|-------------------|
| SMF-28 patch cord (LC-LC, 1m) | Lab connections | 500–1000 |
| Variable optical attenuator (VOA) | Simulate fiber loss | 8,000–15,000 |
| InGaAs photodetector + TIA | Classical channel Rx | 15,000–40,000 |
| 1550nm DFB laser module | Light source | 5,000–12,000 |
| FC/APC connectors | Low reflection | 200–400 each |
| Optical power meter | Calibration | 8,000–20,000 |

For quantum-level (single-photon) work: InGaAs SPAD modules cost ₹2–5 lakh (ID Quantique ID230, Excelitas SPCM). For simulation, you don't need these — purely software emulation.

---

## 5.2 Mininet Topology for Your Architecture

### Complete QKDN Test Setup

```bash
# Terminal 1: Start Ryu controller
ryu-manager sdn/controller.py --observe-links \
  --wsapi-host=0.0.0.0 --wsapi-port=8080

# Terminal 2: Start Mininet topology
sudo python3 qkdn_topo.py

# Terminal 3: On mininet CLI, start services on each node
mininet> alice python3 kms/kms_sqlite.py --port=5001 --node=alice --peers=bob,charlie &
mininet> bob   python3 kms/kms_sqlite.py --port=5002 --node=bob   --peers=alice,charlie &

# Terminal 4: Run integration tests
mininet> alice curl http://localhost:5001/api/v1/keys/bob/status
mininet> alice curl "http://localhost:5001/api/v1/keys/bob/enc_keys?number=5"
```

### Emulating Fiber Loss in Mininet

```python
# In QKDNTopo.build(), add loss parameter to simulate different distances
# 50km C-band fiber: ~10dB → ~10% of photons arrive → loss=90 in Mininet
# For classical traffic simulation: loss=1 (1% packet loss) is realistic

self.addLink(alice_switch, bob_switch,
    cls=TCLink,
    bw=0.001,    # 1 kbps → simulates very low photon rate
    delay='250us', # 50km at speed of light in fiber (c/1.5 ≈ 200,000 km/s)
    loss=10,     # 10% = simulating ~50km fiber loss  
    jitter='10us'
)
```

### Delay Calculations for Fiber Links

Speed of light in SMF: v = c/n ≈ 200,000 km/s (n ≈ 1.5)

| Distance | One-way delay | Round-trip |
|----------|--------------|------------|
| 10km | 50 µs | 100 µs |
| 50km | 250 µs | 500 µs |
| 100km | 500 µs | 1 ms |
| 500km (Delhi-Mumbai) | 2.5 ms | 5 ms |
| 2000km (Delhi-Chennai) | 10 ms | 20 ms |

---

## 5.3 WDM Coexistence: Quantum + Classical on Same Fiber

One of the most practical challenges in real QKD deployment: you want to share fiber with existing classical DWDM traffic (saves huge cost), but classical channels generate **Raman noise** and **four-wave mixing** that leak photons into the quantum channel window.

### Raman Noise Analysis

Stimulated Raman Scattering creates broadband noise photons across all wavelengths when high-power classical channels propagate:

```
Raman noise power ≈ P_classical × L × α_Raman(Δλ)

where Δλ = separation between classical and quantum channels
      α_Raman peaks at ~100nm Stokes shift
```

**Mitigation strategies:**
1. **Spectral separation**: Put quantum channel in O-band (1310nm) and classical in C-band (1550nm). 240nm separation dramatically reduces Raman coupling. Tradeoff: higher O-band loss.

2. **Narrow quantum channel bandpass filter**: Use 0.3nm FWHM filter at receiver. Rejects most Raman noise while passing quantum signal.

3. **Time-domain gating**: Gate detector only during quantum pulse window (synchronous with Alice's laser clock). Reject noise outside timing window.

4. **Anti-Stokes channel**: Place quantum channel at wavelength shorter than classical channels (anti-Stokes Raman is much weaker than Stokes).

5. **Separate fiber strands**: For high-security applications, dedicate dark fiber to quantum channel. Cost: significant, but eliminates Raman noise entirely.

**For your Mininet test with C-band LAN cables**: coexistence is not an issue (you're simulating). When moving to real fiber, plan for Option 2 (narrow filter) + Option 3 (gating) as minimum viable coexistence.

---

## 5.4 Test Matrix and Benchmarking Framework

### Functional Test Matrix

| Test | Expected Result | Acceptance Criteria |
|------|----------------|---------------------|
| Alice↔Bob key agreement | `enc_keys` key == `dec_keys` key | 100% match |
| Key uniqueness | No two requests return same key | 0% collision |
| Key exhaustion | Pool → 0 → refill triggers | Refill within 30s |
| QBER threshold | Inject 15% errors → abort | Protocol aborts cleanly |
| Multi-tenant isolation | Tenant A cannot access Tenant B's keys | 0% cross-tenant leakage |
| Link failure recovery | Kill one KMS → reroute via alternate path | < 5s failover |
| Concurrent users | 100 simultaneous key requests | All served without collision |

### Performance Benchmarking

```python
# bench_qkdn.py — benchmark key delivery latency and throughput
import asyncio, httpx, time, statistics

async def bench_enc_keys(base_url: str, peer: str, 
                          n_requests: int = 1000, concurrency: int = 10):
    latencies = []
    
    async def single_request(client):
        t0 = time.perf_counter()
        r = await client.get(f"{base_url}/api/v1/keys/{peer}/enc_keys?number=1")
        t1 = time.perf_counter()
        assert r.status_code == 200
        latencies.append((t1 - t0) * 1000)  # ms
    
    async with httpx.AsyncClient() as client:
        tasks = [single_request(client) for _ in range(n_requests)]
        await asyncio.gather(*tasks)
    
    print(f"P50:  {statistics.median(latencies):.2f}ms")
    print(f"P95:  {sorted(latencies)[int(0.95*n_requests)]:.2f}ms")
    print(f"P99:  {sorted(latencies)[int(0.99*n_requests)]:.2f}ms")
    print(f"Throughput: {n_requests / sum(latencies) * 1000:.1f} req/s")
```

---

# Part VI — India-Scale QKDN Deployment

## 6.1 India Fiber Infrastructure Landscape

### Existing National Fiber Backbones

| Network | Owner | Coverage | Length |
|---------|-------|----------|--------|
| **BharatNet** (OFC network) | DoT/BSNL | 2.5 lakh gram panchayats | >5 lakh km |
| **NOFN (National Optical Fiber Network)** | BSNL | District HQs → Block HQs | 1.2 lakh km |
| **BSNL interstate network** | BSNL | All state capitals | 7.5 lakh km total |
| **Private (Reliance, Airtel, TATA)** | Bharti, JioFiber, TATA | Tier-1 + Tier-2 cities | ~10 lakh km combined |
| **Railways OFC** | RailTel | Along railway lines | 60,000+ km |
| **NKN (National Knowledge Network)** | NIC/MeitY | Universities, research institutes | 35,000 km |

**For QKD pilot deployment**: NKN is ideal — connects IITs, IISc, DRDO labs, government data centers. Already has dark fiber strands available for research use. Distances between major nodes:
- Delhi (IITD) ↔ Kanpur (IITK): ~500km
- Delhi ↔ Mumbai: ~1400km (multiple fiber routes)
- Delhi ↔ Bangalore: ~2000km

### Dark Fiber Availability

Many fiber routes have unused (dark) fiber strands. These are ideal for QKD:
- No Raman noise from classical channels
- Full control of wavelength plan
- Can deploy DWDM QKD + dedicated classical sync channel

**Current dark fiber lease rates (approximate):**
- Metro dark fiber (per km, per month): ₹500–2000
- Intercity dark fiber: ₹200–800/km/month (varies by route)
- Dedicated fiber from BSNL for government/research: subsidized

---

## 6.2 Phased Rollout Strategy

### Phase 0: Local Lab (Current — Your Setup)
- **Infrastructure**: Single server, Mininet + Docker
- **Nodes**: 5 simulated KMS nodes
- **Goal**: Protocol correctness, API compliance, SDN integration testing
- **Duration**: Current stage
- **Hardware cost**: ₹0 (pure software simulation)

### Phase 1: Campus/Enterprise Testbed (3–6 months)
- **Infrastructure**: 2–3 physical servers connected by dark fiber loops or C-band patch cables
- **Nodes**: 3 real KMS nodes (Alice, Bob, Charlie) on separate servers
- **Distance**: 1–5km (building-to-building at IIT Delhi campus)
- **QKD hardware**: Either CV-QKD over existing fiber, or continued simulation with real fiber channel loss
- **SDN**: Ryu controller managing real OVS switches
- **Goal**: End-to-end real-fiber testing, coexistence, WDM integration

**TAHQEECAT relevance**: This is exactly what the TAHQEECAT project targets — fiber QKD testbed at IITD, C-band, SDN integration.

### Phase 2: City-Scale Pilot (1–2 years)
- **Infrastructure**: Dark fiber ring in Delhi NCR (IITD ↔ DRDO ↔ NIC HQ ↔ AIIMS ↔ back)
- **Nodes**: 6–8 QKD nodes
- **Distance**: 20–50km between nodes
- **QKD hardware**: Toshiba QKD appliances, or IDQ Cerberis XG, or indigenous (C-DOT)
- **SDN**: ONOS cluster (3 nodes) for HA
- **Key applications**: Secure inter-agency communication, healthcare data (AIIMS), financial transactions
- **Government support**: Likely under NQMP (National Quantum Mission)

### Phase 3: Inter-City Metro Backbone (3–5 years)
- **Infrastructure**: BSNL/NKN fiber on Delhi–Mumbai, Delhi–Hyderabad, Bangalore–Chennai routes
- **Nodes**: Trusted-node relays at 50–80km intervals (standard for today's technology)
- **Distance**: 1400–2000km total routes
- **QKD**: TF-QKD for 300–500km spans (reduces trusted-node density)
- **SDN**: Multi-domain ONOS with inter-domain coordination
- **Target users**: BFSI (banking, insurance), Defense, Government

### Phase 4: National QKD Network (5–10 years)
- **Scope**: All state capitals interconnected, integration with satellite QKD (ISRO QKD mission)
- **Architecture**: Hybrid terrestrial + satellite, QKD-secured 5G backhaul, post-quantum hybrid fallback
- **Standards**: Full ETSI QKD 014/015/018 compliance, quantum-safe PKI

---

## 6.3 Regulatory and Spectrum Considerations

### Current Regulatory Landscape (India)

| Body | Jurisdiction | QKD Relevance |
|------|-------------|---------------|
| **DoT (Dept of Telecom)** | Spectrum, licensed telecom | QKD fiber links need telecom license if commercial |
| **SEBI / RBI** | Financial sector security | May mandate QKD for certain transactions eventually |
| **CERT-In** | Cybersecurity standards | Will certify QKD systems for government use |
| **NIC (National Informatics Centre)** | Government IT infrastructure | Key early adopter of QKDN |
| **DRDO / ISRO** | Defense / space | Already running QKD experiments |

### Licensing Considerations

- Commercial QKD service deployment requires ISP license or VNOIPC (Virtual Network Operator) license from DoT
- For research/academic QKD: covered under IIT/DRDO institutional licenses
- Spectrum: QKD operates on optical fiber, not wireless spectrum → no spectrum license needed for fiber QKD

### Export Control

QKD hardware (especially high-performance SPDs) may be subject to dual-use export controls (Wassenaar Arrangement). Indian customs may require clearance for importing advanced QKD hardware from US/EU vendors.

---

## 6.4 Relevant Indian Programs

### National Quantum Mission (NQM / NQMP)

Budget: ₹6000 crore over 8 years (2023–2031).

Four thematic hubs (T-Hubs):
1. **Quantum Computing** (IITB/TIFR consortium)
2. **Quantum Communication** (C-DOT, IITD, IISc)
3. **Quantum Sensing & Metrology** (NPL, CSIR)
4. **Quantum Materials** (JNCASR, IITs)

**QKD-specific goals under NQM:**
- Indigenous QKD transmitter/receiver modules (C-DOT leading)
- Intercity QKD demonstrations (Delhi-Agra fiber pilot)
- Satellite-based QKD link (ISRO collaboration)
- Quantum-secure communication for government networks

### TAHQEECAT (IIT Delhi, Prof. Bhaskar Kanseri's Lab)

The project you applied for. Focus areas:
- Fiber-based QKD experiments in C-band
- SDN integration for QKD network orchestration
- Phase-encoded vs polarization-encoded BB84 comparison
- Long-distance protocols including MDI-QKD and TF-QKD simulation

### C-DOT (Centre for Development of Telematics)

Government telecom R&D lab under DoT. Running an indigenous QKD development program:
- Developing QKD transceiver modules for BB84 over C-band fiber
- Integration with BharatNet infrastructure
- Testing at BSNL Delhi-Agra route (reportedly ~200km)

### DRDO Quantum Program

DRDO's Laser Science & Technology Centre (LASTEC) has active QKD experiments for defense communication. Classified details unavailable, but known to work on:
- Free-space QKD between buildings
- Integration with Army SATCOM for PQC+QKD hybrid

---

# Part VII — Post-Quantum Cryptography Hybrid Layer

## 7.1 Why Hybrid (QKD ⊕ PQC) Instead of Either Alone

| Scenario | QKD Only | PQC Only | Hybrid |
|----------|----------|----------|--------|
| Physical attack on fiber | Vulnerable (QBER detects but can't prevent) | Safe | Safe |
| Long-distance (>500km without repeaters) | Impractical today | Fine | Fine (PQC covers the gap) |
| Quantum computer attack | Secure (information-theoretic) | Vulnerable (to CRQCs) | Secure |
| Side-channel (hardware bugs) | Potentially vulnerable | Potentially vulnerable | Defense in depth |
| Availability (fiber cut) | Complete failure | Works | Works (falls back to PQC) |

**The hybrid XOR construction (your roadmap item):**
```
K_final = K_QKD ⊕ K_PQC

where K_PQC = KEM(Kyber-1024) shared secret
```

If K_QKD is information-theoretically secure, K_final ≥ min(H(K_QKD), H(K_PQC)) security.  
If K_PQC is computationally secure against quantum computers, K_final is quantum-safe even if QKD fails.

## 7.2 CRYSTALS-Kyber (FIPS 203) Implementation

Kyber is a lattice-based KEM (Key Encapsulation Mechanism) selected by NIST:

```python
# pip install kyber-py (pure Python reference)
from kyber import Kyber1024

# Key generation (Bob generates keypair)
pk, sk = Kyber1024.keygen()

# Alice encapsulates (sends ciphertext to Bob)
key_pqc, ct = Kyber1024.enc(pk)

# Bob decapsulates
key_pqc_recovered = Kyber1024.dec(sk, ct)

assert key_pqc == key_pqc_recovered

# Hybrid with QKD key
import hashlib
K_final = bytes(a ^ b for a, b in zip(key_qkd, key_pqc))
# Or: K_final = HKDF(K_QKD || K_PQC)  [better: uses both as input material]
```

## 7.3 PQC + QKD for Your SDN Control Channel

```python
# Bootstrap sequence for new QKDN link
class HybridKeyExchange:
    async def bootstrap_link(self, peer_kms_url: str):
        """
        Step 1: Use Kyber to establish initial secure channel
        Step 2: Start QKD key exchange
        Step 3: Once QKD pool available, switch to QKD-derived control channel key
        Step 4: Renew QKD control channel key periodically (e.g., every 10 min)
        """
        # Step 1: Kyber key exchange
        pk_remote = await self.get_peer_public_key(peer_kms_url)  # via mTLS
        key_pqc, ct = Kyber1024.enc(pk_remote)
        await self.send_ciphertext(peer_kms_url, ct)
        
        # Step 2: QKD key exchange (runs asynchronously, fills pool)
        asyncio.create_task(self.run_qkd_protocol(peer_kms_url))
        
        # Step 3: Upgrade when QKD pool ready
        while not await self.qkd_pool_ready(peer_kms_url, min_keys=10):
            await asyncio.sleep(1.0)
        
        key_qkd = await self.get_qkd_key(peer_kms_url)
        key_hybrid = HKDF_combine(key_pqc, key_qkd)
        
        return key_hybrid
```

---

# Part VIII — Research Frontiers

## 8.1 Quantum Repeaters

The fundamental enabler for long-range trusted-node-free QKD. Three generations:

| Generation | Mechanism | Required Hardware | Status |
|------------|-----------|-------------------|--------|
| 1st gen | Entanglement swapping + purification (photonic) | SPDC sources, BSMs, classical feed-forward | Lab demonstration, ~100km |
| 2nd gen | Error correction on stored qubits | Quantum memories + QEC codes | Early research |
| 3rd gen | All-optical QEC (no quantum memory) | SNSPD arrays, cluster states | Theory phase |

**Quantum memory candidates** (for 1st gen repeaters):

| Platform | Storage time | Efficiency | Bandwidth | Wavelength |
|----------|-------------|------------|-----------|-----------|
| DLCZ (cold atoms) | 100ms–1s | 50–90% | GHz | 795nm (Rb), 780nm |
| AFC (Pr:YSO) | 1h (record) | 50% | GHz | 606nm |
| NV centers (diamond) | ms | 70% | MHz | 637nm |
| Atomic ensembles (warm) | µs | 50% | GHz | Rb/Cs transitions |

**Convert to telecom C-band**: Requires quantum frequency conversion (QFC) via nonlinear optics (difference frequency generation). Active research area.

## 8.2 Continuous-Variable QKD Over WDM Networks

CV-QKD is the most compatible with existing telecom infrastructure because:
- Uses coherent detection (standard in 100G/400G transceiver modules)
- Gaussian modulation → compatible with QAM signal processing chains
- No cryogenic hardware required

**Key research papers:**
- Laudenbach et al., "Continuous-Variable QKD with Gaussian Modulation," Advanced Quantum Technologies, 2018
- Pirandola et al., "Advances in Quantum Cryptography," Advances in Optics and Photonics, 2020
- Jouguet et al., "Experimental demonstration of long-distance CV-QKD," Nature Photonics, 2013

**Open problem**: CV-QKD over WDM with classical channel coexistence. The noise from adjacent DWDM channels (even 50 GHz spacing) introduces excess noise ε that degrades the secure key rate. Active research in noise mitigation filters, DSP-based noise estimation, and adaptive modulation variance.

## 8.3 Satellite QKD

Micius satellite (China, 2016–2021) demonstrated:
- 1200km ground-to-ground QKD via satellite relay (trusted-node at satellite)
- 7600km intercontinental QKD (Beijing–Vienna)
- Ground-to-satellite entanglement distribution at 1200km

**For India:** ISRO's QSAT program is developing satellite QKD payload. The challenge:
- Atmospheric turbulence → beam wander → photon loss
- Satellite is a trusted node (key passes through it) unless entanglement distribution is used
- Daylight operation requires narrow spectral filtering (suppress solar background)

**Free-space QKD channel model:**
```
Transmittance η = η_atmosphere × η_pointing × η_aperture

η_atmosphere ≈ exp(-α × sec(θ) × H_eff)   [Beer-Lambert]
η_pointing   ≈ exp(-2σ²_point/θ²_div)      [Gaussian beam, pointing error σ_point]
η_aperture   = (D_rx/2 / θ_div × L)²       [D_rx = receiver aperture, L = distance]
```

## 8.4 SDN for Quantum Internet

The **Quantum Internet Alliance (QIA)** has a roadmap for a future quantum internet:

| Stage | Capability | SDN Role |
|-------|-----------|----------|
| Stage 1 | Trusted-node QKD | Current — route optimization |
| Stage 2 | Prepare-and-measure (MDI-QKD) | Untrusted relay orchestration |
| Stage 3 | Entanglement distribution | Entanglement routing, purification scheduling |
| Stage 4 | Quantum memory networks | Memory reservation, entanglement buffering |
| Stage 5 | Fault-tolerant quantum computing | Full quantum network stack |

**SDN for entanglement routing** is an active research area. The key difference from classical routing:
- Entanglement is probabilistic: a link may or may not successfully distribute an entangled pair
- Entanglement is non-cloneable: can't buffer or copy
- Must coordinate entanglement swapping across multiple hops with precise timing

Papers: 
- Caleffi (2017), "Optimal Routing for Quantum Networks," IEEE Access
- Chakraborty et al. (2020), "Distributed Routing in a Quantum Internet," Physical Review Research

---

# Appendix A — Key Equations Reference Sheet

## QKD Key Rate

**Asymptotic secure key rate (BB84, decoy state):**
```
R = Q_1 [1 - h(e_1)] - Q_μ f × h(E_μ)

where:
  Q_1   = gain of single-photon component
  e_1   = error rate of single-photon component
  Q_μ   = overall gain (detection rate / pulse rate)
  E_μ   = overall QBER
  f     = error correction efficiency (≥ 1)
  h(x)  = binary entropy = -x log₂(x) - (1-x) log₂(1-x)
```

**Finite-key correction (Tomamichel et al., 2012):**
```
ℓ = n [1 - h(e_1 + γ)] - m f × h(E_μ) - log₂(2/ε_sec) - 2 log₂(1/ε_cor)

where:
  n = number of signal pulses detected
  m = number of key bits used for parameter estimation
  γ = statistical fluctuation term (~ sqrt(log(1/ε)/n))
  ε_sec, ε_cor = security and correctness parameters
```

## Fiber Loss and Distance

```
Loss (dB) = α × L       [α = 0.2 dB/km at 1550nm]

Transmittance η = 10^(-Loss/10) = 10^(-α×L/10)

Detection probability = η_fiber × η_detector × η_coupling

e.g., 100km:
  η_fiber = 10^(-20/10) = 0.01 (1%)
  With η_detector = 0.20 (20%): detection rate = 0.2%
```

## QBER and Security

```
QBER_intercept_resend = (1/4) × (1 - cos²(2θ))    [general attack]
QBER_intercept_resend = 1/4 = 25%                   [optimal intercept-resend]

Security condition: QBER < 11.0% (for BB84 with single photons)
                    QBER < 15.0% (for E91 / six-state protocol)
```

## Key Pool Sizing

```
Required pool = max_key_rate × max_key_consumption × safety_factor

max_key_rate        = expected peak application demand (keys/s)
max_key_consumption = burst duration (seconds)
safety_factor       = 2–5 × (to handle QKD link downtime)

e.g., 10 keys/s peak, 60s burst, factor 3:
  Required pool = 10 × 60 × 3 = 1800 keys of 256-bit each
               = 1800 × 32 bytes = 57,600 bytes ≈ 56 KB (tiny)
```

---

# Appendix B — Complete Resource Library

## Foundational Papers (Must-Read)

### QKD

| Paper | Authors | Year | Why Read |
|-------|---------|------|----------|
| "Quantum Cryptography: Public Key Distribution and Coin Tossing" | Bennett, Brassard | 1984 | Original BB84 |
| "Quantum Cryptography Based on Bell's Theorem" | Ekert | 1991 | E91 protocol |
| "Unconditional Security of Quantum Key Distribution Over Arbitrarily Long Distances" | Lo, Chau | 1999 | First composable security proof |
| "Simple Proof of Security of the BB84 QKD Protocol" | Shor, Preskill | 2000 | Cleaner security proof |
| "Security of QKD with imperfect devices" | GLLP (Gottesman, Lo, Lutkenhaus, Preskill) | 2004 | Realistic device security |
| "Advances in Quantum Cryptography" | Pirandola et al. | 2020 | Comprehensive review (300+ pages) |
| "Twin-field QKD beyond the fundamental rate-distance limit" | Lucamarini et al. | 2018 | TF-QKD origin paper |
| "Measurement-Device-Independent QKD" | Lo, Curty, Qi | 2012 | MDI-QKD |

### SDN

| Paper | Authors | Year | Why Read |
|-------|---------|------|----------|
| "OpenFlow: Enabling Innovation in Campus Networks" | McKeown et al. | 2008 | Original OpenFlow paper |
| "A Survey of Software Defined Networking" | Kreutz et al. | 2015 | Comprehensive SDN survey |
| "ONOS: Towards an Open, Distributed SDN OS" | Berde et al. | 2014 | ONOS design |
| "B4: Experience with a Globally-Deployed SDN WAN" | Jain et al. (Google) | 2013 | SDN at Google scale |

### SDN + QKD Integration

| Paper | Authors | Year | Why Read |
|-------|---------|------|----------|
| "Software Defined Networking for Quantum Key Distribution Networks" | Aguado et al. | 2017 | First SDN-QKD integration |
| "Madrid-Bilbao QKD Link" | Aguado, Lopez et al. | 2019 | Real SDN-QKD deployment |
| "Optical SDN for QKD Network Control Plane" | Cao et al. | 2019 | China Quantum Backbone SDN |
| "ETSI QKD 014 Standard" | ETSI | 2019 | Must-read for your API compliance |
| "Quantum Internet: A Vision for the Road Ahead" | Wehner, Elkouss, Hanson | 2018 | Science magazine; roadmap |

## Online Resources

### Documentation
- **ETSI QKD Standards**: https://www.etsi.org/technologies/quantum-key-distribution
- **OpenFlow 1.3 Spec**: https://opennetworking.org/wp-content/uploads/2014/10/openflow-switch-v1.3.5.pdf
- **Ryu Documentation**: https://ryu.readthedocs.io
- **ONOS Documentation**: https://wiki.onosproject.org
- **Mininet Documentation**: http://mininet.org/walkthrough/
- **OVS Documentation**: https://docs.openvswitch.org

### Courses & Lectures
- **MIT 8.422 — Atomic and Optical Physics II**: Covers QKD foundations (lecture notes on OCW)
- **TU Delft QuTech QKD MOOC**: Comprehensive QKD course (edX)
- **Stanford CS244 — Advanced Topics in Networking**: SDN (lecture slides online)
- **Coursera "The Quantum Internet and Quantum Computers"**: TU Delft, free audit

### Simulators & Tools

| Tool | Purpose | Language | Link |
|------|---------|---------|------|
| **QuNetSim** | Quantum network simulation | Python | https://github.com/tqsd/QuNetSim |
| **NetSquid** | Discrete-event quantum net sim | Python | https://netsquid.org (registration required) |
| **SeQUeNCe** | Quantum networking simulation | Python | https://github.com/sequence-toolkit/sequence |
| **SimulaQron** | Classical simulation of quantum nodes | Python | http://www.simulaqron.org |
| **Qiskit** | General quantum computing | Python | https://qiskit.org |
| **Mininet** | Classical network emulation | Python/C | http://mininet.org |
| **ONOS** | SDN controller | Java | https://onosproject.org |
| **GNS3 + OVS** | Network emulation (alternative to Mininet) | Various | https://gns3.com |

### Indian Research Groups (for Networking/Collaboration)

| Group | Institution | Focus | Contact Point |
|-------|-------------|-------|---------------|
| Prof. Bhaskar Kanseri | IIT Delhi (Photonics & Quantum Optics) | QKD, entanglement sources | TAHQEECAT project |
| Quantum Information & Computation group | IISc Bangalore | QEC, quantum algorithms | |
| C-DOT QKD Team | C-DOT Delhi | Indigenous QKD hardware | DoT-funded |
| Prof. R.P. Singh group | PRL Ahmedabad | Free-space QKD, entanglement | |
| DRDO LASTEC | Delhi | Defense QKD applications | Classified |
| Prof. Anindita Banerjee | JNU | Quantum crypto protocols | |

## Books

| Title | Authors | Focus |
|-------|---------|-------|
| *Quantum Computation and Quantum Information* | Nielsen & Chuang | Foundational textbook; Ch 12 on QKD |
| *Quantum Cryptography and Secret-Key Distillation* | Gisin & Thew | Dedicated QKD textbook |
| *Introduction to Quantum Key Distribution* | Scarani et al. | Review article length book (free: arXiv:0802.4155) |
| *Software Defined Networks: A Comprehensive Approach* | Goransson, Black, Culver | SDN engineering reference |
| *OpenFlow in Practice* | ONF | Practical OF programming |
| *Security Engineering* (3rd ed.) | Ross Anderson | Ch 21: Quantum Cryptography in context |

## arXiv Reading List

These are free, preprint versions of key papers:

```
quant-ph/0512152  — Renner PhD thesis (finite-key security)
arXiv:1906.04402  — Twin-field QKD review
arXiv:1710.09802  — CV-QKD review (Laudenbach)
arXiv:1907.02771  — MDI-QKD practical implementations
arXiv:1805.04553  — Quantum Internet roadmap (Wehner)
arXiv:1910.09495  — Quantum network SDN (comprehensive)
arXiv:0802.4155   — Scarani et al. QKD review (highly readable)
```

---

## Quick Reference: Your Architecture Checklist

For TAHQEECAT interview / architecture review:

- [x] BB84 simulation with QBER monitoring
- [x] ETSI QKD 014 REST API compliance
- [x] SDN controller (Ryu OF1.3)
- [x] Multi-node KMS topology (5 nodes)
- [x] PostgreSQL + Redis key storage
- [x] Prometheus + Grafana observability
- [x] Docker Compose deployment
- [x] JWT authentication + multi-tenancy scaffold
- [ ] HKDF-based privacy amplification (replace SHA-256)
- [ ] Decoy state simulation
- [ ] Finite-key length calculation
- [ ] QBER-weighted SDN path computation
- [ ] ETSI QKD 015 (control interface) implementation
- [ ] ETSI QKD 018 (trusted-node relay) for multi-hop
- [ ] PQC hybrid layer (Kyber-1024 + QKD XOR)
- [ ] Controller HA (ONOS cluster for production)
- [ ] Authenticated LLDP for topology integrity
- [ ] Key encryption at rest

---

*Document version: 1.0 | Generated for QKDN SaaS v2 + TAHQEECAT preparation*  
*Scope: SDN + QKD subject matter expertise from fundamentals to India-scale deployment*
